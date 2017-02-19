---
layout: post
title:  "Adventures in /usr/bin and the likes"
date:   2017-02-19 14:00:00 +0200
categories: linux adventures commands
image: /images/linux.png
description: Digging up some useful and not so popular commands in Linux
---

# Introduction

I just love Linux! For me it makes interacting with your computer fun and educational. I think if someone needs to learn about the core principles underlying an operating system and hardware, Linux is a great place to start.

As we all know interacting with Linux is achieved mostly through programs (commands). One of the things which differentiates Linux and the \*nix based systems is the command line and the concept of pipes. Combining the input and output of programs provides a very powerful platform for manipulating data.

Most of the Linux commands/programs/binaries (however you decide to call them) reside under /usr/bin, /usr/sbin, /bin and /usr/local/bin. Looking through the contents of these folders we can see a lot of programs. This got me thinking, what are all those binaries meant to do, in my daily work I hardly use any of them. So I decided to go on a treasure hunt and write about my findings in this blog post.

As a start, I wanted to see some data, a simple count on my machine for /usr/bin gives:

```bash
ablagoev ➜  ~  ls -la /usr/bin | wc -l
2420
```

This is of course my everyday system and I have a lot of things there. Running the same on a vanilla server Centos 6.8 installation (no X server), with just a little bit of additional software:

```
[root@XXXXXX ~]# ls -la /usr/bin | wc -l
1054
```

Half that amount, but still quite a lot. In my daily work I probably use around 20 commands on average. Here's a rough way to check it under zsh:

```
fc -li -146 | awk '{print $4}' | sort | uniq -c
```

This will give you the last 146 commands. I used fc to see the timestamps and filter out commands only from today.

With this data in mind, I decided to dig inside /usr/bin and the likes and see if there aren't any hidden gems. All of these were present on my vanilla Centos 6.8 installation, different distributions will most likely have differences.

I learned about each program as I wrote this post, so please excuse my ignorance, where it blatantly shines through.

# Contents

1. [Misc](#misc)
* [at](#at)
* [cal](#cal)
* [mkfifo](#mkfifo)
* [resolveip](#resolveip)
* [timeout](#timeout)
* [w](#w)
* [watch](#watch)
2. [OS](#os)
* [blktrace](#blktrace)
* [chrt](#chrt)
* [lstopo](#lstopo)
* [numactl and taskset](#numactl-and-taskset)
* [numastat](#numastat)
* [peekfd](#peekfd)
* [pidstat](#pidstat)
* [pwdx](#pwdx)
3. [Debugging](#debugging)
* [addr2line](#addr2line)
* [gstack](#gstack)
4. [Data Manipulation](#data-manipulation)
* [column](#column)
* [colrm](#colrm)
* [comm](#comm)
* [csplit](#csplit)
* [cut](#cut)
* [join](#join)
* [paste](#paste)
* [replace](#replace)
* [look](#look)
* [nl](#nl)
* [tac](#tac)
* [expand](#expand)
* [unexpand](#unexpand)

# Misc

## at

> at and batch read commands from standard input or a specified file which are to be executed at a later time, using /bin/sh.

Nice! I usually mess up the cron syntax when trying to setup up a one off job and this one seems a lot simpler for these kinds of use cases. For example:

```
root@XXXX:~# at 9:26 PM
warning: commands will be executed using /bin/sh
at> date > /tmp/test_at.log
at>^D
```

`date > /tmp/test_at.log` will be executed at 9.26 PM on the same day. Pending jobs can be viewed:

```
root@XXXX:~# atq
1	Wed Feb  8 21:26:00 2017 a root
```

at appears to understand some cool time formats. For example it knows teatime (`at teatime`). Cool!

## cal

```
ablagoev ➜  ~  cal
   Февруари 2017
нд пн вт ср чт пт сб
          1  2  3  4
 5  6  7  8  9 10 11
12 13 14 15 16 17 18
19 20 21 22 23 24 25
26 27 28
```

Displays a nifty calendar with the current date highlighted. Pretty useful if you ask me.

## mkfifo

Another good one. `mkfifo` can be used for simple IPC (InterProcess Communication). It creates a `named pipe` which, like everything in Linux, is a file. What this means is that we can have processes writing to this file (writers) and processes reading from this file (readers). Consider the following two python scripts:

```
#!/usr/bin/python

# reader.py

i = 0
while True:
	fp = open('fifo', 'r')
	line = fp.readline()
	print "Item in fifo: " + line.rstrip() + " on iteration " + str(i) + "\n"
	fp.close
	i += 1
```

```
#!/usr/bin/python

# writer.py
import time

i = 0
while True:
	fp = open('fifo', 'a')
	fp.write("Hello " + str(i) + "!\n")
	fp.close()
	i += 1
	time.sleep(1)
```

Before using the scripts we create the fifo:

```
mkfifo fifo
```

we start the writer:

```
./writer.py
```

and we start the reader:

```
./reader.py
```

Notice the rate at which the reader outputs the contents of the fifo - it is in sync with the writer. If fifo was just an ordinary file, the reader will constantly read from the file, disregarding whether there are new items or not. This is because reading and writing on the fifo are blocking operations. Go ahead and change the fifo to an ordinary file and see the difference.

Another great example from <a href="https://en.wikipedia.org/wiki/Netcat#Proxying" target="_blank">Netcat's entry in Wikipedia</a>:

```
mkfifo backpipe
nc -l 12345 0<backpipe | nc www.google.com 80 1>backpipe
```

This is an example of a proxy, which will forward all requests on port 12345 to www.google.com port 80. It needs a fifo, because pipes are one directional.

## resolveip

Usually to get the ip of a host I use ping. `resolveip` is simpler:

```
ablagoev ➜  ~  resolveip en.wikipedia.org
IP address of en.wikipedia.org is 91.198.174.192
```

If you are like me, you often need to copy the ip, use the -s option which returns only the ip:

```
resolveip -s en.wikipedia.org | xsel -b
```

This can easily be made into a function, just put it into your .bash_aliases file:

```bash
# Copy an ip address from a domain
# Usage cip domain.com
function cip() {
	ip=$(resolveip -s $1)
	echo $ip | xsel -b
	echo $ip
}
```

```
ablagoev ➜  ~  cip en.wikipedia.org
91.198.174.192
```

## timeout

> run a command with a time limit

If you don't want to edit code this one is pretty useful. Example:

```
timeout 10 ./program
```

This will run the program 10 seconds and then send TERM signal to it. The signal can be modified, as well as the units for the duration (i.e minutes, hours and days).

## w
> Show who is logged on and what they are doing.

On a multiuser system this can be useful:

```
root ➜  /home/ablagoev  w ablagoev
 12:20:44 up 1 day, 56 min,  5 users,  load average: 0,06, 0,13, 0,21
 USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
 ablagoev :0       :0               сб11   ?xdm?   1:18m  0.59s upstart --user
 ablagoev pts/4    :0               сб11   24:51m 46.01s 45.70s /home/ablagoev/.rbenv/versions/2.3.1/bin/jekyll serve
 ablagoev pts/7    :0               сб23    4.00s  2.42s  4:04  gnome-terminal
 ablagoev pts/25   :0               сб11   12.00s  3:10   3:10  vim _posts/2017-02-10-adventures-in-usr-bin.markdown
 ablagoev pts/33   :0               сб12    1:35m  2.28s  4:04  gnome-terminal
```

## watch

`watch` is a nifty tool for running a program periodically and monitoring its output. A simple clock:

```
watch -n 1 -d date
```

-n tells watch to run the program every second. -d tells watch to highlight the differences in the output.

# OS

## blktrace

This one merits a full blog post and I have to admit I am not prepared to go into detail here. From what I understand it allows you to monitor the IO of your system on a lower level than iostat does. You can measure a lot of IO related metrics. A good post on the subject seems to be <a href="https://brooker.co.za/blog/2013/07/14/io-performance.html" target="_blank">this one</a>.

Only newer kernels have this, older ones need to be patched.

## chrt

We all know about setting the "niceness" of processes in our systems. What I did not know, is that you can also alter their CPU scheduling policies. Currently, modern kernels support the following policies:

* SCHED_OTHER the standard round-robin time-sharing policy;
* SCHED_BATCH for "batch" style execution of processes;
* SCHED_IDLE for running very low priority background jobs;

* SCHED_FIFO a first-in, first-out policy (only for real time processes);
* SCHED_RR a round-robin policy (only for real time processes);

Additionally, there are two types of processes: real time ones and normal ones. Real time processes have higher priority and are used for time sensitive applications.

By changing a process' policy we can alter the way Linux schedules the process.

User processes are usually not real time and with the SCHED_OTHER policy by default. I've checked the kernel processes and some of them have SCHED_FIFO, but none of them (at least on my machine) have SCHED_RR.

Using `chrt` the scheduling policy of a process can be changed and queried. For example, checking init:

```
ablagoev ➜  ~  chrt -p 1
pid 1's current scheduling policy: SCHED_OTHER
pid 1's current scheduling priority: 0
```

Consequently, a process' policy can be changed as follows:

```
ablagoev ➜  ~  chrt -b -p 0 PID
ablagoev ➜  ~  chrt -p PID
pid 970's current scheduling policy: SCHED_BATCH
pid 970's current scheduling priority: 0
```

I could not find practical use cases though and whether it is worth it to meddle with the policies in production. [This post](https://barro.github.io/2016/02/being-a-good-cpu-neighbor/) goes into depth with some benchmarks. I'd love it if someone can share his experience and knowledge regarding the policies.

More detailed reading here:

```
man sched_setscheduler
```

## lstopo

This one is very cool. It can be used to display the hardware layout of a machine (its topology). Supports a lot of different output formats, as well as ASCII art, which is awesome. Here's an example:

```
[root@centos-db2 ~]# lstopo --of txt
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Machine (31GB)                                                                                                                │
│                                                                                                                               │
│ ┌────────────────────────────────────────────────────────────────────────┐            ┌─────────────────┐                     │
│ │ Socket P#0                                                             │  ├┤╶─┬─────┤ PCI 8086:153a   │                     │
│ │                                                                        │      │     │                 │                     │
│ │ ┌────────────────────────────────────────────────────────────────────┐ │      │     │ ┌────────┐      │                     │
│ │ │ L3 (8192KB)                                                        │ │      │     │ │ eth0   │      │                     │
│ │ └────────────────────────────────────────────────────────────────────┘ │      │     │ └────────┘      │                     │
│ │                                                                        │      │     └─────────────────┘                     │
│ │ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │      │                                             │
│ │ │ L2 (256KB)   │  │ L2 (256KB)   │  │ L2 (256KB)   │  │ L2 (256KB)   │ │      │ 0,2       0,2           ┌─────────────────┐ │
│ │ └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘ │      ├─────┼┤╶───────┼┤╶───────┤ PCI 1a03:2000   │ │
│ │                                                                        │      │                         └─────────────────┘ │
│ │ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │      │                                             │
│ │ │ L1d (32KB)   │  │ L1d (32KB)   │  │ L1d (32KB)   │  │ L1d (32KB)   │ │      │ 0,2       0,2 ┌─────────────────┐           │
│ │ └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘ │      ├─────┼┤╶───────┤ PCI 8086:1533   │           │
│ │                                                                        │      │               │                 │           │
│ │ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │      │               │ ┌────────┐      │           │
│ │ │ L1i (32KB)   │  │ L1i (32KB)   │  │ L1i (32KB)   │  │ L1i (32KB)   │ │      │               │ │ eth1   │      │           │
│ │ └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘ │      │               │ └────────┘      │           │
│ │                                                                        │      │               └─────────────────┘           │
│ │ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │      │                                             │
│ │ │ Core P#0     │  │ Core P#1     │  │ Core P#2     │  │ Core P#3     │ │      │     ┌────────────────────┐                  │
│ │ │              │  │              │  │              │  │              │ │      └─────┤ PCI 8086:8c02      │                  │
│ │ │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │            │                    │                  │
│ │ │ │ PU P#0   │ │  │ │ PU P#1   │ │  │ │ PU P#2   │ │  │ │ PU P#3   │ │ │            │ ┌──────┐  ┌──────┐ │                  │
│ │ │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │            │ │ sda  │  │ sdb  │ │                  │
│ │ │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │            │ └──────┘  └──────┘ │                  │
│ │ │ │ PU P#4   │ │  │ │ PU P#5   │ │  │ │ PU P#6   │ │  │ │ PU P#7   │ │ │            └────────────────────┘                  │
│ │ │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │                                                    │
│ │ └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘ │                                                    │
│ └────────────────────────────────────────────────────────────────────────┘                                                    │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Host: centos-db2                                                                                                           │
│                                                                                                                               │
│ Indexes: physical                                                                                                             │
│                                                                                                                               │
│ Date: 18.02.2017 (сб) 18,19,39 CET                                                                                            │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

The command is part of the `hwloc` package.

## numactl and taskset

It appears that these are candidates for separate blog posts too. As far as I understand, both commands can be used to control a process'/thread's cpu affinity, i.e you can assign different processes or threads to different cores or sockets. Additionally, you can also control which memory nodes (NUMA pools) a process/thread should use. Practically, these are used when the Linux scheduler does not do an optimal job and places threads or similar processes across cores. In order to start you should review your numa architecture:

```
[root@chat ~]# numactl -H
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 8 9 10 11
node 0 size: 24566 MB
node 0 free: 23075 MB
node 1 cpus: 4 5 6 7 12 13 14 15
node 1 size: 24576 MB
node 1 free: 23309 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

This is a 48GB server, with two CPU sockets and 16 cores in total. Memory is divided into two nodes.

A simple example with numactl:

```
numactl --cpunodebind=0 ./program
```

this will run `program` on the first CPU socket (0).

To change a process' affinity at run time you can use `taskset`. Taskset works with cores and not with sockets, so if you want to assign a process to a single socket you will have to list all cores:

```
taskset -p -c 0 pid
```

Using -c 0 we are assigning the pid to core #0. Cores can be viewed with `cat /proc/cpuinfo`.

Additionally, to view the current CPU core of a process look for the psr column in ps:

```
ablagoev ➜  ~  ps -o pid,psr,comm -p 1
	PID PSR COMMAND
	1   1 init
```

This means that my init process is currently on core #1.

## numastat

> Show per-NUMA-node memory statistics for processes and the operating system

```
[root@chat ~]# numastat
                           node0           node1
numa_hit               502845176       522381709
numa_miss                      0               0
numa_foreign                   0               0
interleave_hit             22624           22649
local_node             502838421       522343368
other_node                  6755           38341
```

To me it seems logical, that if you have a lot of misses (a miss is a memory allocation to a node that is not preferred by a process) then there is non optimal affinity somewhere on the system.

## peekfd

This one can be used to monitor io on a given process. Before continuing, though, I have to quote this from the man page:

> BUGS
> Probably lots.  Don't be surprised if the process you are monitoring dies.

I'd be careful in using it in production. Perhaps someone can chime in regarding its stability. In any case, monitoring vim activity on save:

```bash
root ➜  /home/ablagoev  peekfd 20921

writing fd 1:
 [1b] [?5h [1b] [?5l [1b] [?25l [1b] [m [1b] [57;1H [1b] [K [1b] [57;1H:w [1b] [?12l [1b] [?25h
 [1b] [?25l"t"
writing fd 5:
Example IO
writing fd 1:
 [noeol] 1L, 10C written [1b] [1;14H [1b] [?12l [1b] [?25h
writing fd 7:
b0VIM 7.4 [10] 2 [a9] X [8a]  [14]  [16]  [b9] Qablagoevablagoev-VPCF22M1E~ablagoev/t [09] 3210#"!  [13]  [12] U

```

The cool thing is you can actually see what is written to the file. Of course, this can also be achieved with strace and some filtering of the output.

## pidstat

This one can be used to monitor various statistics for a given process or the whole system. It gives information similar to top/htop, but is capable of providing further details. For example:

```bash
[root@chat ~]# pidstat -p 19562 1
Linux 3.10.104-1.el6.elrepo.x86_64 (ldp3chat) 	19.02.2017 	_x86_64_	(16 CPU)

10,22,03          PID    %usr %system  %guest    %CPU   CPU  Command
10,22,04        19562   11,00    3,00    0,00   14,00    15  node
10,22,05        19562   10,00    1,00    0,00   11,00    10  node
10,22,06        19562   11,00    3,00    0,00   14,00    11  node
10,22,07        19562   16,00    2,00    0,00   18,00    10  node
10,22,08        19562   23,00    2,00    0,00   25,00    11  node
10,22,09        19562   16,00    1,00    0,00   17,00    14  node
10,22,10        19562   14,00    2,00    0,00   16,00    13  node
```

The "1" at the end tells pidstat to poll statistics every second. You can watch io statistics using the -d option:

```bash
[root@chat ~]# pidstat -d -p 19562 1
Linux 3.10.104-1.el6.elrepo.x86_64 (chat) 	19.02.2017 	_x86_64_	(16 CPU)

10,23,38          PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10,23,39        19562      0,00      0,00      0,00  node
10,23,40        19562      0,00      0,00      0,00  node
10,23,41        19562      0,00      0,00      0,00  node
10,23,42        19562      0,00      0,00      0,00  node
10,23,43        19562      0,00      0,00      0,00  node
```

Memory:

```bash
[root@chat ~]# pidstat -r -p 19562 1
Linux 3.10.104-1.el6.elrepo.x86_64 (chat) 	19.02.2017 	_x86_64_	(16 CPU)

10,24,16          PID  minflt/s  majflt/s     VSZ    RSS   %MEM  Command
10,24,17        19562      2,00      0,00 1576132 524288   1,06  node
10,24,18        19562     36,00      0,00 1574416 522588   1,05  node
10,24,19        19562     47,00      0,00 1574416 522588   1,05  node
10,24,20        19562     70,00      0,00 1575440 522840   1,05  node
10,24,21        19562    186,00      0,00 1575440 523632   1,06  node
10,24,22        19562     30,00      0,00 1575440 523632   1,06  node
```

Task (context) switching:

```bash
[root@chat ~]# pidstat -w -p 19562 1
Linux 3.10.104-1.el6.elrepo.x86_64 (chat) 	19.02.2017 	_x86_64_	(16 CPU)

10,25,48          PID   cswch/s nvcswch/s  Command
10,25,49        19562    481,00     97,00  node
10,25,50        19562    359,00     64,00  node
10,25,51        19562    401,00     71,00  node
10,25,52        19562    383,00     63,00  node
```

## pwdx

A simple and useful tool to get the working directory of a process:

```bash
ablagoev ➜  ~  pwdx 6038
6038: /home/ablagoev/workspace/ablagoev.github.io
```
# Debugging

## addr2line

This one is a bit tricky to use and it merits another full blog post. I know folks dealing with lower level programming are probably using this on a daily basis and in a range of different cases. For me, though, this one is useful for debugging segfaults.

Segfaults usually have the following form:

```
php-fpm[6048]: segfault at 10 ip 00007f46db77a8fb sp 00007fffa155e2d0 error 4 in xcache.so[7f46db763000+23000]
```

In this case the segfault is appearing in a shared library (xcache.so). We need to calculate the correct address for addr2line by subtracting the library offset (xcache.so[**7f46db763000**+23000] from the instruction pointer (ip part).

```
00007f46db77a8fb - 7f46db763000 = 178FB
```

Once we have this address we feed it into addr2line, along with the location of the shared object:

```
addr2line -e /usr/lib64/20131226/xcache.so 178FB
```

On this particular machine the output is:

```
/root/source/xcache-3.2.0/mod_cacher/xc_cacher.c:778
```

This is the offending line in the c source code, which generates the segfault. Awesome!

Like I said, though, this one needs a full blog post as there is a lot more to it. For example, for this all to work your binaries need to be compiled with debug symbols on, otherwise you will not be able to get the needed information. Also, there is theory behind the address calculation, which is very interesting (It's on my reading todo list). Additionally, there are other tools which allow further debugging (gdb).

## gstack

> print a stack trace of a running process

Again... Awesome! Being able to see the stacktrace of a rouge process can definitely help in tracking down a problem. Usage is pretty straightforward:

```
gstack pid
```

The command is a wrapper around gdb. Thus, if your distribution is missing it you can achieve the same with gdb.

# Data Manipulation

## column

This one is awesome, can't believe I did not know it. It can be used to format input in columns. For me the most useful application would be to format csv files. Among other stuff you can specify a delimiter for the different columns. Imagine you have the following file:

```
id;event;count
1;test;10
2;test2;20
3;test3;30
```

Using the following command:

```
ablagoev ➜  ~  column -s ";" -t test
id  event  count
1   test   10
2   test2  20
3   test3  30
```

This gets interesting in combination with the next command.

## colrm

This one can be used to remove a column from an input. I am used to using awk, but colrm seems easier for simpler use cases. With the result from the above command, we could do this to remove the first column:

```
ablagoev ➜  ~  column -s ";" -t test | colrm 1 4
event  count
test   10
test2  20
test3  30
```

## comm

> Compare two sorted files line by line

With `comm` you can compare two files and see which items (lines) are in the first or second file only and whether there are lines present in both files.
Consider the following two files:

```
ex1

Common line
In both files
Only in 1st file
```

```
ex2

Common line
In both files
Only in 2nd file
```

Running `comm` on them will give you the following output:

```
ablagoev ➜  ~  comm ex1 ex2
		Common line
		In both files
Only in 1st file
	Only in 2nd file
```

Output is separated in columns, first column contains lines only in the first file, second column contains lines only in the second file and third column contains lines present in both files. One thing to consider is that both files should be sorted alphabetically. Finally, there are options which control which columns to display.

## csplit

Similarly to split (if you don't know it you should definitely check it out), `csplit` can be used to split a big file into smaller chunks.
Unlike split, though, csplit can be used with a regular expression and sections can be of different sizes. Consider this example file (data.txt):

```
Item: Shoes
Price: 9.99
Quantity: 10
----
Item: Dresses
Price: 19.99
Quantity: 15
----
Item: Socks
Price: 1.99
Quantity: 30
```

Using the following command:

```
csplit --suppress-matched data.txt '/^----$/' {*}
```

you will get 3 files containing only the item blocks. `--suppress-matched` option tells csplit to exclude matching lines from the output. `{*}` tells csplit to repeat the matching pattern as many times as possible. Unfortunately, csplit operates line by line, i.e multiline regular expressions will not work.

## cut

Cut is considered popular, but I never got round to checking it out. By the looks of it, it is a very powerful tool. Similarly to colrm, cut can be used to extract portions (columns) from a file. Unlike colrm, though, you can specify delimiters and not just start/end offsets. This makes the command very powerful for parsing csv files, for example. Consider this data (data.txt):

```
id;name;count
1;Shoes;10
2;Dresses;15
3;Socks;30
```

Using cut, will give you:

```
ablagoev ➜  ~  cut -d';' -f1 data.txt
id
1
2
3
```

-d option specifies the delimiter (";" in our case) and -f option specifies which columns to output. If we wanted both columns 1 and 3:

```
ablagoev ➜  ~  cut -d';' -f1,3 data.txt
id;count
1;10
2;15
3;30
```

There are some other options which make the command even more flexible. One, worthy of mention, is that you can control the output delimiter, i.e:

```
ablagoev ➜  ~  cut -d';' -f1,3 --output-delimiter $'\t' data.txt
id	count
1	10
2	15
3	30
```

--output-delimter $'\t' tells cut to use a tab character.

## join

This one is also popular, but unknown to me, so here it goes. It allows joining data from two files based on a common field. Consider these two files:

```
ex1

Date Count
2017-02-02 10
2017-02-03 10
2017-02-04 10

```

```
ex2

Date Sum
2017-02-02 50
2017-02-03 60
2017-02-04 70
```

Using join you will get:

```
ablagoev ➜  ~  join ex1 ex2
Date Count Sum
2017-02-02 10 50
2017-02-03 10 60
2017-02-04 10 70
```

We've joined the files, based on the first column. The command provides the flexibility to choose which columns to use in both files and which delimiter to use (by default it is whitespace).

## paste

Paste can be used to merge files together. In the simplest case with 1 file, it can be used to join all lines of a file. Given this file (data):

```
1
2
3
4
5
6
7
8
9
10
```

running this:

```
ablagoev ➜  ~  paste -s -d ',' data
1,2,3,4,5,6,7,8,9,10
```

-d option specifies the delimiter and -s is used to join all the lines of the file. I am a bit confused by the description of the -s option and why it doesn't work without it on a single file:

> -s, --serial paste one file at a time instead of in parallel

Maybe without the -s the file is processed line by line.

Another use cases is to join two files. Consider the following (ex1 and ex2):

```
Name
Shoes
Skirs
Socks
Dresses
```

```
Quantity
10
20
30
40
```

```
ablagoev ➜  ~  paste ex1 ex2
Name	Quantity
Shoes	10
Skirs	20
Socks	30
Dresses	40
```

Of course delimiter can also be specified as in the first example.

## replace

I wish I knew this earlier, so that I didn't have to memorize sed's syntax for simple search and replace operations. This one can be used to replace strings in files. For example, given the file (data):

```
variable = 10
operation = variable + 12
print variable
```

running `replace` to change "variable" -> "x":

```
ablagoev ➜  ~  replace variable x -- data
data converted
ablagoev ➜  ~  cat data
x = 10
operation = x + 12
print x
```

Quick and easy. Multiple search and replace pairs can be used, as well as multiple files.

## look

This is one little gem. It filters out lines beginning with a specified string. Essentially it does what grep "^PREFIX" does, but without the regular expression. Given the file (data):

```
pref: 123
some other thing
pref: 234
another thing
pref: 222
```

```
ablagoev ➜  ~  look pref data
pref: 123
pref: 234
pref: 222
```

One interesting use case, that is described in the man page, is to use `look` for spellchecking. Apparently there is an english dictionary in /usr/share/dict/words and by default `look` uses this as an input. So:

```
ablagoev ➜  ~  look defini
defining
definite
definitely
definiteness
definiteness's
definition
definition's
definitions
definitive
definitively
```

## nl

A very handy small tool. Adds line numbers to a file. Given the file (data):

```
This is the first line.
This is the second line.
This is the third line.
```

and running `nl` on it:

```
ablagoev ➜  ~  nl data
     1	This is the first line.
     2	This is the second line.
     3	This is the third line.
```

## tac

Just like cat, but print the files in reverse order:

```
ablagoev ➜  ~  cat data
1
2
3
4
5
6
```

```
ablagoev ➜  ~  tac data
6
5
4
3
2
1
```

## expand

This one is simple, but seems handy. It converts tabs to spaces.

## unexpand

Reverse of the above. Converts spaces to tabs.

# Conclusion

And this is it. I learned a lot of stuff as I was writing this blog post, my love and appreciation for Linux has grown. It is amazing what kind of transparency and detail the system provides. If you have other gems which you would like to share, please do in the comments section below. Most of the commands were new to me, so I probably made a lot of factual errors, I'd appreciate your feedback and corrections.
