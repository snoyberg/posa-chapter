# Warp

Authors: Michael Snoyman and Kazu Yamamoto

Warp is a high-performance library of HTTP server side in Haskell,
a purely functional programming language.
Both Yesod, a web application framework, and Mighttpd, an HTTP server,
are implemented over Warp.
According to our throughput benchmark,
Mighttpd provides performance on par with nginx.
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
by a single process or native thread.

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

As of this writing, Mighttpd uses the prefork technique to fork processes
to utilize cores and Warp does not have this functionality.
Haskell community is now developing parallel IO manager.
If it will be merged to GHC, Warp itself can use this architecture
without any modifications.

## Warp's architecture

Warp is an HTTP engine for WAI (Web Application Interface).
It runs WAI applications over HTTP.
As we described before both Yesod and Mighttpd are
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

The user thread repeats this procedure if necessary and terminates by itself when
the connection is closed by the peer.

## Performance of Warp

Before we explain how to improve the performance of Warp,
we would like to show the results of our benchmark.
We measured throughput of Mighttpd 2.8.2 (with Warp x.x.x) and nginx 1.2.4.
Our benchmark environment is as follows:

- One "12 cores" machine (Intel Xeon E5645, two sockets, 6 cores per 1 CPU, two QPI between two CPUs)
- Linux version 3.2.0 (Ubuntu 12.04 LTS), which is running directly on the machine (i.e. without a hypervisor)

We tested several benchmark tools in the past and
our favorite one was `httperf`.
Since it uses the `select()` system call and is just a single process program,
it reaches its performance limits when we try to measure HTTP servers on
multi-cores.
So, we switched to `weighttp`, which 
is based on the `epoll` system call family and can use
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

We carefully configured both Mighty and `nginx` as follows:

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
So, we need to use as fewe system calls as possible.
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
The `accept4()` system call is an extension version of `accept()` on Linux.
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

TBD

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
It uses the `send()` system call with the `MSG_MORE` flag to store a header
and the `sendfile()` system call to send both the stored header and a file.
This made the throughput at least 100 times faster.

## Clean-up with timers

### For connections

- Requirements
- System.Timeout.timeout (not scale because one timeout thread per thread)
- MVar (slow because homebrew spin lock is used)
- IORef
- Its algorithm

Need a fig

### For file descriptors

- Requirements
-- no leakage
-- lookup
-- multimap
-- fast pruning
- Red black tree

Need a fig

## Logging (xxx necessary?)

- Handle
- From the Mighty article in Monad.Reader

Need a fig

- date
- avoiding gettimeofday()
- caching the formatted date

## Future work

- lock free memory allocation

## Profiling and benchmarking

Each item should be included in other chapters.

- weighttp (done)
- GHC profiler
- strace (done)
- eventlog
- prof
- tcpdump (done)

## Conclusion
