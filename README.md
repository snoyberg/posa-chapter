posa-chapter
============

Chapter for Performance of Open Source Applications

---

We'll focus mostly on Warp performance. Things to consider:

* Network programming architecture
    * Good old thread programming with multi processes/OS threads
    * Event driven
    * Event driven with preforked processes (nginx, nodejs)
    * Green threads on event driven with preforked processes (current GHC RTS)
    * Green threads on event driven with OS threads in one process (future)
* Minimize system calls.
    * Non-blocking
    * accept() vs accept4()
        * An accepted socket inherits the flags from its listening socket
          on BSD.
        * An accepted socket does not inherit the flags from its listening
          socket on Linux. Two fcntl()s are necessary to set O_NONBLOCK:
          reading the flags, ORed them with O_NONBLOCK, and set them.
          accept4() can return an accepted socket with O_NONBLOCK set.
    * sendto() vs writev()
    * sendfile()
    * file info cache -- open()/stat()/close()
    * gettimeofday() and time() are system calls on Linux 3.
      But they are not system calls anymore on Linux 3.
      BSD's gettimeofday() and time() are not system calls.
* Parsing. (#1)
    * Dedicated parser (not using parser combinator)
* Building. (#2)
    * Blaze builder (O(1) appending, O(N) copying)
* Logging
    * Handle
    * Date builder
         http-date:    359ns
         unix-time:  2,071ns
         time:      43,636ns
        * Use http-date to generate web date
        * Use unix-time on Unix to generate zoned date
            * This is cached
* Lock free
    * Using CAS instead of lock (MVar)

#1 and #2 should be symmetric.

---

Schedule

* 1-page outline by September 3rd (Labour Day)
* First draft by the end of October
* Second draft mid-January (after review by us)
* Third draft in March (after review by the review team)
* Publication in May
