Title: Is a process CPU or IO bound?
Date: Wed May 24 15:41:07 +08 2017
Category: Howto
Tags: linux, howto

In Linux we can control/observe processes by reading/writing to (virtual) files
exposed via `/proc` or `/sys`. 

To check if a process is CPU or IO bound, we can compare the number of
voluntary vs involuntary context switches a process has undergone. 

```
# grep 'ctxt_switches' /proc/$pid/status
voluntary_ctxt_switches:        23397
nonvoluntary_ctxt_switches:     1874
```

When a process requests for IO, it will voluntary give up the CPU. When a
process is just crunching numbers, it will hold on to the CPU until it's kicked
out. So for the above example, the process is using more IO than CPU since
voluntary exceeds involuntary by an order of magnitude

The source of this tip actually comes from the ksplice blog which unfortunately
seems to be offline, but a [Google cache
version](https://webcache.googleusercontent.com/search?q=cache:3YLJEptE1BsJ:https://blogs.oracle.com/ksplice/solving-problems-with-proc+ksplice+solving+problems+with+proc)
still exists.
