Title: Kill is a shell built-in
Date: Mon Mar  5 22:20:16 +08 2018
Category: gotcha
Tags: kill, shell, gotcha

`kill` is a common shell built in. `man kill` gives you the manual for the
`kill` binary instead of the built-in. Certain switches documented in the
manual, such as `-L` are not actually implemented in the built in, so you have
to explicitly run the binary using `/bin/kill` instead.

For example, on OpenBSD:
```
% kill -l
HUP INT QUIT ILL TRAP ABRT EMT FPE KILL BUS SEGV SYS PIPE ALRM TERM URG STOP TSTP CONT CHLD TTIN TTOU IO XCPU XFSZ VTALRM PROF WINCH INFO USR1 USR2 THR
% /bin/kill -l
HUP INT QUIT ILL TRAP ABRT EMT FPE KILL BUS SEGV SYS PIPE ALRM TERM URG
STOP TSTP CONT CHLD TTIN TTOU IO XCPU XFSZ VTALRM PROF WINCH INFO USR1 USR2 THR
```

Similarly, on Linux:
```
% kill -L
kill: unknown signal: SIGL
kill: type kill -l for a list of signals
% /bin/kill -L
 1 HUP      2 INT      3 QUIT     4 ILL      5 TRAP     6 ABRT     7 BUS
 8 FPE      9 KILL    10 USR1    11 SEGV    12 USR2    13 PIPE    14 ALRM
15 TERM    16 STKFLT  17 CHLD    18 CONT    19 STOP    20 TSTP    21 TTIN
22 TTOU    23 URG     24 XCPU    25 XFSZ    26 VTALRM  27 PROF    28 WINCH
29 POLL    30 PWR     31 SYS
```
