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
    * sendto() vs writev()
    * sendfile()
    * file info cache -- open()/stat()/close()
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
