Title: Systemtap Guru (Dangerous) Mode
Date: Thu May 17 21:36:22 +08 2018
Category: Tracing
Tags: tracing, kernel, systemtap, stap, monitoring

Systemtap has a guru mode, activated via `-g` option of `stap`, which allows
the stap script to modify kernel variables and perform feats like [live patching the kernel.](https://access.redhat.com/blogs/766093/posts/1976643)

I decided to try out guru mode on a fresh ubuntu/xenial vagrant VM:

```bash
vagrant init ubuntu/xenial64
vagrant up && vagrant ssh
```

Install Systemtap on [xenial](https://wiki.ubuntu.com/Kernel/Systemtap)

```bash
apt update && apt install -y systemtap
```

Systemtap needs kernel debug symbols in order to work with the probe points.
Installing that is another ~3Gb:

```bash
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C8CAB6595FDFF622
codename=$(lsb_release -c | awk  '{print $2}')
tee /etc/apt/sources.list.d/ddebs.list << EOF
deb http://ddebs.ubuntu.com/ ${codename}      main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-security main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-updates  main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-proposed main restricted universe multiverse
EOF

apt-get update
apt-get install linux-image-$(uname -r)-dbgsym
```

Lastly, grab the kernel source (~750Mb) so you know which functions to trace
```
apt-get source linux
```

This grabs and extracts the source for your kernel into your current working
directory.

Verify that Systemtap works:
```bash
stap -ve 'probe begin { print("hello world")}'
```

Since guru mode can be dangerous, I wanted to just try something simple like
modify the `write` system call to write only the first byte and exit.  And even
then I will only modify system calls issued by `echo` processes owned by the
`uid=1000(vagrant)` user 

At first I tried:
```awk
probe syscall.write {
        if(uid() == 1000 && execname() == "echo" && $count > 1){
			$count = 1
        }
}
```

The `$` syntax references variables from the code being traced, in this case,
the `count` variable that is passed to the `write` system call (see `man 2
write`).

Run that with `stap -g orig.stp` (`-g` for guru mode) and the result:

```shell
vagrant@ubuntu-xenial:~$ /bin/echo 'hello'
hello
vagrant@ubuntu-xenial:~$ 
```

That didn't seem to work. The expected output is just `h`.

Add in a debug print statement and try again:

```awk
probe syscall.write {
        if(uid() == 1000 && execname() == "echo" && $count > 1){
                $count = 1
                printf("count:%d, buf:%s\n", $count, kernel_string($buf))
        }
}
```

Note: Since `buf` is a char pointer, to dereference it we have to use
`kernel_string`. Simply using `$buf` alone just gives you the address pointed
to.

The debug stap trace from a single `/bin/echo 'hello'`:
```bash
root@ubuntu-xenial:~/learn_stap# stap -g orig.stp
count:1, buf:hello

count:1, buf:ello

count:1, buf:llo

count:1, buf:lo

count:1, buf:o
```

At this point, it starts to make sense. The `write` system call always returns
the number of bytes written, which is not guaranteed to equal to `count`. It is
the duty of the caller, in this case, `echo`, to check the return value and
retry `write` accordingly, which it did. Since the hacked `write` only writes 1
byte at a time `echo` simply issues the system call once for each byte.

In order to "fix" this "bug", I have to also modify the return value of
`write`. 

At first I tried to poke the return value using the trace point
`syscall.write.return` like so:

```awk
global orig_count
probe syscall.write {
        if(uid() == 1000 && execname() == "echo" && $count > 1){
                orig_count = $count
                $count = 1
        }
}
probe syscall.write.return{
        if(uid() == 1000 && execname() == "echo" && $count == 1){
                $ret = orig_count
        }
}
```

But alas Systemtap does not support it:
`semantic error: write to target variable not permitted in .return probes: identifier '$ret' at orig.stp:11:3`

And I can't set `$ret` in the `syscall.write` handler either as that is too
early. So I need to find a more specific place in the source code to probe.

The source code of a system call is decorated with the [SYSCALL_DEFINE macro.](https://stackoverflow.com/a/10149838)
Going back to the kernel source code extracted earlier, we can see that
`fs/read_write.c` contains the source code for `write`:

```bash
~/linux-4.4.0# grep -lPr 'SYSCALL_DEFINE[0-6]\(write' *
fs/read_write.c
~/linux-4.4.0#
```

Specifically lines 601-616 of `read_write.c`
```c
 601 SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
 602                 size_t, count)
 603 {
 604         struct fd f = fdget_pos(fd);
 605         ssize_t ret = -EBADF;
 606 
 607         if (f.file) {
 608                 loff_t pos = file_pos_read(f.file);
 609                 ret = vfs_write(f.file, buf, count, &pos);
 610                 if (ret >= 0)
 611                         file_pos_write(f.file, pos);
 612                 fdput_pos(f);
 613         }
 614 
 615         return ret;
 616 }
```

Now a good place to poke `ret` will be line `615`. To do that, we can use this
trace point `kernel.statement("SyS_write@fs/read_write.c:615")` , which
basically means to trace the kernel function called `SyS_write` defined in
`fs/read_write.c` line 615. 

Note: Because the kernel function name in this case is a macro, I have to use
`ppfunc()` to get stap to print out the name that the macro eventually defines.
`stap -e 'probe syscall.write { printf("%s\n", ppfunc())}` prints `SyS_write`
for each `write` intercepted.

Unfortunately, line 615 cannot be traced for some reason:

```bash
~# stap -l 'kernel.statement("SyS_write@fs/read_write.c:615")'
Tip: /usr/share/doc/systemtap/README.Debian should help you get started.
~# echo $?
1
~# 
```

In fact, only 605, 607, 608, 609, 610 can be traced. I don't know why.

```bash
~# for line in `seq 604 615`; do
> stap -l "kernel.statement(\"SyS_write@fs/read_write.c:$line\")"
> done
Tip: /usr/share/doc/systemtap/README.Debian should help you get started.
kernel.statement("SyS_write@/build/linux-DqQ95A/linux-4.4.0/fs/read_write.c:605")
Tip: /usr/share/doc/systemtap/README.Debian should help you get started.
kernel.statement("SyS_write@/build/linux-DqQ95A/linux-4.4.0/fs/read_write.c:607")
kernel.statement("SyS_write@/build/linux-DqQ95A/linux-4.4.0/fs/read_write.c:608")
kernel.statement("SyS_write@/build/linux-DqQ95A/linux-4.4.0/fs/read_write.c:609")
kernel.statement("SyS_write@/build/linux-DqQ95A/linux-4.4.0/fs/read_write.c:610")
Tip: /usr/share/doc/systemtap/README.Debian should help you get started.
Tip: /usr/share/doc/systemtap/README.Debian should help you get started.
Tip: /usr/share/doc/systemtap/README.Debian should help you get started.
Tip: /usr/share/doc/systemtap/README.Debian should help you get started.
Tip: /usr/share/doc/systemtap/README.Debian should help you get started.
~# 
```
 
Of the usable lines, only 610 is suitable since `ret` is already set by then.

```awk
global orig_count
probe syscall.write {
        if(uid() == 1000 && execname() == "echo" && $count > 1){
                orig_count = $count
                $count = 1
        }
}
probe kernel.statement("SyS_write@fs/read_write.c:610"){
        if(uid() == 1000 && execname() == "echo" && $count == 1){
                $ret = orig_count
        }
}
```

And finally it works:
```shell
vagrant@ubuntu-xenial:~$ /bin/echo 'hello'
hvagrant@ubuntu-xenial:~$ 
```
