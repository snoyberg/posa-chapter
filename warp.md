1. Enumerate what we should write (design this article)
2. Then flesh them out (implement this article)

# Warp

Authors: Michael Snoyman and Kazu Yamamoto

- What is Warp?
-- http://www.yesodweb.com/blog/2012/09/header-body

## Network programming in Haskell

- Cover the contents of these articles:
- http://www.iij.ad.jp/en/company/development/tech/mighttpd/
- http://www.yesodweb.com/blog/2012/09/improving-warp

## Warp's architecture

- http://www.yesodweb.com/blog/2012/09/header-body

We should say these here?

1. Issuing as few system calls as possible
2. Avoiding re-calculation

## HTTP request parser

- Parser generator vs handmade parser
- From "Warp: A Haskell Web Server"?
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

- http://www.yesodweb.com/blog/2012/09/header-body

## Clean-up with timers

### For connections

- Requirements
- System.Timeout.timeout
- MVar vs IORef
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

## Logging

- Handle
- From the Mighty article in Monad.Reader

Need a fig

- date
- avoiding gettimeofday()
- caching the formatted date

## Other tips

- accept4
- char8
- pessimistic read

## Profiling and benchmarking

- GHC profiler
- httperf
- strace
- threadscope

## Conclusion
