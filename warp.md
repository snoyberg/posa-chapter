# Warp

Authors: Michael Snoyman and Kazu Yamamoto

Warp is a high-performance library of HTTP server side in Haskell,
a purely functional programming language.
Both Yesod, a web application framework, and `mighty`, an HTTP server,
are implemented over Warp.
According to our throughput benchmark,
`mighty` provides performance on par with nginx.
This article will explain
the architecture of Warp and how we improved its performance.
Warp can run on many platforms
including Linux, BSD variants, MacOS, and Windows.
To make our explanation simple, however, we will talk about Linux only
for the rest of this article.

## Network programming in Haskell

Some people may still misunderstand that 
functional programming languages are slow or impractical.
However, to our best knowledge, 
Haskell provides the best scheme for network programming.
This is because that GHC (Glasgow Haskell Compiler), 
the flagship compiler of Haskell, provides
lightweight and robust user thread (or sometime called green thread).
In this section, we will briefly explain history of
network programming in server side.

### Native threads

Traditional servers use a technique called thread programming.
In this architecture, each connection is handled
by a single process or native thread (or sometime called OS thread)

This architecture can be broken down by how to create processes or native threads.
When using a thread pool, multiple processes or native threads are created in advance.
An example of this is the prefork mode in Apache.
Otherwise, a process or native thread is spawn each time a connection is received. Fig XXX illustrates this.

![Native threads](1.png)

The advantage of this architecture is that clear code can be written
because the code is not decided into event handlers.
Also, because the kernel assigns processes or
native threads to available cores,
we can balance utilization of cores.
Its disadvantage is a large number of
context switches between kernel and processes or native threads occur.
So, performance gets poor.

### Event driven

Recently, it is said that event-driven programming is 
required to implement high performance servers.
In this architecture multiple connections are handled by a single process (Fix XXX).
Lighttpd is an example of web server using this architecture.

![Event driven](2.png)

Since there is no need to switch processes,
less context switches occur, and performance is improved.
This is its chief advantage.
However, it has two shortcomings.
The first is the fact that only one core
can be utilized because there is only a single process.
The second is that it requires asynchronous programming,
so code is fragmented into event handlers.
Asynchronous programming also prevents the conventional
use of exception handling (although there are no exceptions in C).


### 1 process per core 

Many have hit upon the idea of creating
N event-driven processes to utilize N cores (Fig XXX).
Port 80 must be shared for web servers.
Using the prefork technique (please don't confuse with Apache's prefork mode),
port sharing can be achieved by modifying code slightly.

![1 process per core](3.png)

One web server that uses this architecture is nginx.
Node.js used the event-driven architecture in the past but
it also implemented this scheme recently.

The advantage of this architecture is
that it utilizes all cores and improves performance. 
However, it does not resolve the issue of programs having poor clarity.

### User threads

GHC's user threads can be used to solve the code clarity.
They are implemented over an event-driven IO manager in GHC's runtime system.
Starndard libraries of Haskell use non-blocking system calls
so that they can cooperate with the IO manager.
GHC's user threads are lightweight: 
modern computers can run 100,000 user threads smoothly.
They are robust: even asynchronous exceptions are catch
(we explain this later in detail).

Some languages and libraries provided user threads in the past,
but they are not commonly used now because they are not lightweight
or are not robust.
But in Haskell, most computation is non-destructive.
This means that almost all functions are thread-safe.
GHC uses data allocation as a safe point of context switch of user threads.
Because of functional programming style,
new data are frequently created and it is known that 
[such data allocation is regulaly enough for context switching](http://www.aosabook.org/en/ghc.html).

Use of lightweight threads makes
it possible to write code with good clarity
like traditional thread programming
while keeping high performance (Fig XXX).

![User threads](4.png)

As of this writing, `mighty` uses the prefork technique to fork processes
to utilize cores and Warp does not have this functionality.
Haskell community is now developing parallel IO manager.
A Haskell program with parallel IO manager is
executed as a single process and
multiple IO managers run as native threads to utilize cores.
And user threads are executed on one of cores.
If it will be merged to GHC, Warp itself can use this architecture
without any modifications.

## Warp's architecture

Warp is an HTTP engine for WAI (Web Application Interface).
It runs WAI applications over HTTP.
As we described before both Yesod and `mighty` are
examples of WAI applications as illustrated in Fix XXX.

![WAI](wai.png)

The type of WAI applications is as follows:

    type Application = Request -> ResourceT IO Response

In Haskell, argument types of function are separated by right arrows and
the most right one is the type of return value.
So, we can interpret the definition
as an application takes `Request` and returns `Response`.

After accepting a new HTTP connection, a dedicated user thread is spawn for the
connection.
It first receives an HTTP request from a client
and parses it to `Request`.
Then, Warp gives the `Request` to an application and
takes a `Response` from it.
Finally, Warp builds an HTTP response based on `Response`
and sends it back to the client.
This is illustrated in Fix XXX.

![Warp](warp.png)

The user thread repeats this procedure
if necessary and terminates by itself
when the connection is closed by the peer.
It is also killed by the dedicated user thread for timeout
if a significant amount of data is not received in a specified period.

## Performance of Warp

Before we explain how to improve the performance of Warp,
we would like to show the results of our benchmark.
We measured throughput of `mighty` 2.8.2 (with Warp x.x.x) and nginx 1.2.4.
Our benchmark environment is as follows:

- One "12 cores" machine (Intel Xeon E5645, two sockets, 6 cores per 1 CPU, two QPI between two CPUs)
- Linux version 3.2.0 (Ubuntu 12.04 LTS), which is running directly on the machine (i.e. without a hypervisor)

We tested several benchmark tools in the past and
our favorite one was `httperf`.
Since it uses `select()` and is just a single process program,
it reaches its performance limits when we try to measure HTTP servers on
multi-cores.
So, we switched to `weighttp`, which 
is based on the `epoll` family and can use
multiple native threads. 
We used `weighttp` as follows:

    weighttp -n 100000 -c 1000 -t 3 -k http://127.0.0.1:8000/

This means that 1,000 HTTP connections are established and
each connection sends 100 requests.
3 native threads are spawn to carry out these jobs..

For all requests, the same `index.html` file is returned.
We used `nginx`'s `index.html` whose size is 151 bytes.
As "127.0.0.1" suggests, We measured web servers locally.
We should have measured from a remote machine but
we don't have suitable environment at this moment.
(NOTE: I'm planning to do benchmark using two machines soon.)

Since Linux has many control parameters,
we need to configure the parameters carefully.
You can find a good introduction about
Linux parameter tuning in [ApacheBench & HTTPerf](http://gwan.com/en_apachebench_httperf.html).

We carefully configured both `mighty` and `nginx` as follows:

- using file descriptor cache
- no logging
- no rate limitation

Since our machine has 12 cores and
`weighttp` uses three native threads,
we measured web servers from one worker to
eight workers(to our experience,
three native thread is enough to measure 8 workers).
Here is the result:

![Performance of Warp and nginx](multi-workers.png)

X-axis is the number of workers and y-axis means throughput
whose unit is requests per second.

## Key ideas

There are three key ideas to implement high-performance server in Haskell:

1. Issuing as few system calls as possible
2. Specialization and avoiding re-calculation
3. Avoiding locks

### Issuing as few system calls as possible

If a system call is issued, 
CPU time is given to kernel and all user threads stop.
So, we need to use as few system calls as possible.
For a HTTP session to get a static file,
Warp calls `recv()`, `send()` and `sendfile()` only (Fig warp.png).
`open()`, `stat()`, `close()` and other system calls can be committed
thanks to cache mechanism described later.

We can use `strace` to see what system calls are actually used.
When we observed behavior of `nginx` with `strace`, 
we noticed that it uses `accept4()`, about which we don't know at that time.

Using Haskell's standard network library, 
a listening socket is created with the non-blocking flag set.
When a new connection is accepted from the listening socket,
it is necessary to set the corresponding socket as non-blocking, too.
The `network` package implements this by calling `fcntl()` twice:
one is to get the current flags and the other is to set
the flags with the non-blocking flag *ORed*.

On Linux, the non-block flag of a connected socket
is always unset even if its listening socket is non-blocking.
`accept4()` is an extension version of `accept()` on Linux.
It can set the non-blocking flag when accepting.
So, if we use `accept4()`, we can avoid two unnecessary `fcntl()`s.
Our patch to use `accept4()` on Linux has been already merged to
the network library.

### Specialization and avoiding re-calculation

GHC profiling
criterion
Char8
http-date

### Avoiding locks

Unnecessary locks are evil for programming.
Our code sometime uses unnecessary locks imperceptibly
because runtime systems or libraries uses locks deep inside.
To implement high-performance server,
we need to identify locks and
avoid locks if possible.
It is worth pointing out that
locks will become much more critical under
the parallel IO managers.
We will talk how to identify and avoid locks
in Section XXX and Section XXX.

## HTTP request parser

- Parser generator vs handmade parser
- No timeout care thanks to timeout manager
-- From "Warp: A Haskell Web Server"?
- Conduit

## HTTP response builder

### response header

- Blaze builder vs low level memory operations

### response body

- Three types
- Blaze builder
- Conduit
- sendfile

### sending header and body together


When we measured the performance of Warp,
we always did it with high concurrency.
That is, we always make multiple connections at the same time.
It gave us a good result.
However, when we set the number of concurrency to 1, 
we found Warp is really really slow.

Observing the results of `tcpdump`, 
we realized that this is because old Warp uses
the combination of writev() for header and sendfile() for body.
In this case, an HTTP header and body are sent in separate TCP packets (Fig xxx).

![Packet sequence of old Warp](tcpdump.png)

To send them in a single TCP packet (when possible),
new Warp switched from `writev()` to `send()`.
It uses `send()` with the `MSG_MORE` flag to store a header
and `sendfile()` to send both the stored header and a file.
This made the throughput at least 100 times faster.

## Clean-up with timers

This section explain how to implement connection timeout and
how to cache file descriptors.

### Timers for connections

To prevent slowloris attacks, a dedicated user thread kills a user thread,
which communicates with a client,
if the client does not send a significant amount of data for a specified period (30 seconds by default).

TBD: System.Timeout

The heart of Warp's timeout system is the following two points:

- Double `IORef`s
- Safe swap and merge algorithm

Suppose that status of connections is described as `Active` and `Inactive`.
To clean up inactive connections,
a dedicated Haskell thread, called the timeout manager, repeatedly inspects the status of each connection.
If status is `Active`, the timeout manager turns it to `Inactive`.
If `Inactive`, the timeout manager kills its associated Haskell thread.

Each status is refereed by an `IORef`.
To update status through this `IORef`,
atomicity is not necessary because status is just overwritten.
In addition to the timeout manager,
each Haskell thread repeatedly turns its status to `Active` through its own `IORef` if its connection actively continues.

To check all statuses easily,
the timeout manager uses a list of the `IORef` to status.
A Haskell thread spawned for a new connection
tries to 'cons' its new `IORef` for an `Active` status to the list.
So, the list is a critical section and we need atomicity to keep
the list consistent.

![A list of status](timeout.png)

A standard way to keep consistency in Haskell is `MVar`.
But `MVar` is slow
because each `MVar` is protected with a home-brewed spin lock.
So, he used another `IORef` to refer the list and `atomicModifyIORef`
to manipulate it.
`IORef` is a reference whose value can be destructively updated.
`atomicModifyIORef` is a function to atomically update `IORef`'s values.
It is fast since it is implemented via CAS (Compare-and-Swap),
which is much faster than spin locks.

The following is the outline of the safe swap and merge algorithm:

    do xs <- atomicModifyIORef ref (\ys -> ([], ys)) -- swap with an empty list, []
       xs' <- manipulates_status xs
       atomicModifyIORef ref (\ys -> (merge xs' ys, ()))

The timeout manager atomically swaps the list with an empty list.
Then it manipulates the list by turning status and/or removing
unnecessary status for killed Haskell threads.
During this process, new connections may be created and
their status are inserted with `atomicModifyIORef` by
corresponding Haskell threads.
Then, the timeout manager atomically merges
the pruned list and the new list.

### Timers for file descriptors

Let's consider the case where Warp sends the entire file by `sendfile()`.
Unfortunately, we need to call `stat()`
to know the size of the file
because `sendfile()` on Linux requires the caller
to specify how many bytes to be sent
(`sendfile()` on FreeBSD/MacOS has magic number '0'
which indicates the end of file).

If WAI applications know the file size,
Warp can avoid `stat()`.
It is easy for WAI applications to cache file information
such as size and modification time.
If cache timeout is fast enough (say 10 seconds),
the risk of cache inconsistency problem is not serious.
And because we can safely clean up the cache,
we don't have to worry about leakage.

Since `sendfile()` requires a file descriptor,
the naive sequence to send a file is
`open()`, `sendfile()` repeatedly if necessary, and `close()`.
In this section, we consider how to cache file descriptors
to avoid `open()` and `close()`.
Caching file descriptors should work as follows:
If a client requests that a file be sent, 
a file descriptor is opened by `open()`.
And if another client requests the same file shortly thereafter,
the file descriptor is reused.
At a later time, the file descriptor is closed by `close()`
if no user thread uses it.

A typical tactic for this case is reference counter.
But we was not sure that we could implement a robust mechanism
for a reference counter.
What happens if a user thread is killed for unexpected reasons?
If we fail to decrement its reference counter,
the file descriptor leaks.
We noticed that the scheme of connection timeout is safe
to reuse as a cache mechanism for file descriptors
because it does not use reference counters.
However, we cannot simply reuse Warp's timeout code for some reasons:

Each user thread has its own status. So, status is not shared.
But we would like to cache file descriptors to avoid `open()` and
`close()` by sharing.
So, we need to search a file descriptor for a requested file
from cached ones.
Since this look-up should be fast, we should not use a list.
Also,
because requests are received concurrently,
two or more file descriptors for the same file may be opened.
So, we need to store multiple file descriptors for a single file name.
This is technically called a *multimap*.

We implemented a multimap whose look-up is O(log N) and
pruning is O(N) with red-black trees
whose node contains a non-empty list.
Since a red-black trees is one of binary search trees,
look-up is O(log N) where N is the number of nodes.
Also, we can translate it into an order list in O(log N).
In our implementation, 
pruning nodes which contains a file descriptor to be closed is
also done during this procedure.
An algorithm is known, which converts a order list to a red-black tree in O(N).

## Future work

We have some items to improve Warp in the future but
we will explain two here.

### Memory allocation

When receiving and sending packets, buffers are allocated.
We think that these memory allocations may be the current bottleneck.
GHC runtime system uses `pthread_mutex_lock`
to obtain a large object (larger than 409 bytes in 64 bit machines).

We tried to measure how much memory allocation
for HTTP response header consume time.
We copied the `create` function of `ByteString` to Warp and
surrounded `mallocByteString` with `Debug.Trace.traceEventIO`. 
Then we complied `mighty` with it and took eventlog.
The result eventlog is illustrated as follows:

![eventlog](eventlog.png)

Brick red bars indicates the event created by `traceEventIO`. 
The area surrounded by two bars is the time consumed by `mallocByteString`.
It is about 1/10 of an HTTP session.
We are confident that the same thing happens when allocating receiving buffers.

### New thundering herd

Thundering herd is an old but new problem. 
Suppose that processes/native threads are pre-forked to share a listening socket.
They call `accept()` on the socket.
When a connection is created, old Linux and FreeBSD
wakes up all of them.
And only one can accept it and the others sleeps again.
Since this causes many context switches, 
we face performance problem.
This is called *thundering* *herd*.
Recent Linux and FreeBSD wakes up only one process/native thread.
So, this problem became a thing of the past.

Recent network servers tend to use the `epoll` family.
If worker processes share a listen socket and
they manipulate accept connections through the `epoll` family,
thundering herd appears again.
This is because
the semantics of the `epoll` family is to notify
all processes/native threads. 
`nginx` and `mighty` are victims of new thundering herd.

The parallel IO manager is free from new thundering herd.
In this architecture,
only one IO manager accepts new connections through the `epoll` family. 
And other IO managers handle established connections.

## Conclusion
