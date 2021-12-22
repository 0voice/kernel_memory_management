In this tutorial I will share different methods and tools to detect and find memory leaks with different processes in Linux. As a developer we often face scenarios when proess such as httpd apache, java starts consuming high amount of memory leading to OOM (Out Of memory) situations.

So it is always healthy to keep monitoring the memory usage of critical process. I work for an application which is memory intensive so it is my job to make sure other processes are not eating up the memory unnecessarily. In this process I use different tools in real time environments to detect memory leaks and then report it to the responsible developers.

 

## What is Memory Leak?

- Memory is allocated on demand—using `malloc()` or one of its variants—and memory is freed when it’s no longer needed.
- A memory leak occurs when memory is allocated but not freed when it is no longer needed.
- Leaks can obviously be caused by a `malloc()` without a corresponding `free()`, but leaks can also be inadvertently caused if a pointer to dynamically allocated memory is deleted, lost, or overwritten.
- Buffer overruns—caused by writing past the end of a block of allocated memory—frequently corrupt memory.
- Memory leakage is by no means unique to embedded systems, but it becomes an issue partly because targets don't have much memory in the first place and partly because they often run for long periods of time without rebooting, allowing the leaks to become a large puddle.
- Regardless of the root cause, memory management errors can have unexpected, even devastating effects on application and system behavior.
- With dwindling available memory, processes and entire systems can grind to a halt, while corrupted memory often leads to spurious crashes.

 

Before you go ahead I would recommend you to also read about **[Linux memory management](https://www.golinuxcloud.com/tutorial-linux-memory-management-overview/)** so that you are familiar with the different terminologies used in Linux kernel in terms of memory.

 

## 1. Memwatch

- MEMWATCH, written by Johan Lindh, is an open-source memory error-detection tool for C.
- It can be downloaded from https://sourceforge.net/projects/memwatch
- By simply adding a header file to your code and defining MEMWATCH in your `gcc` command, you can track memory leaks and corruptions in a program.
- MEMWATCH supports ANSI C; provides a log of the results; and detects double frees, erroneous frees, unfreed memory, overflow and underflow, and so on.

I have downloaded and extracted memwatch on my Linux server as you can check in the screenshot:

![Extract MEMWATCH](https://www.golinuxcloud.com/wp-content/uploads/2020/08/memwatch.jpg)

 

Next before we compile the software, we must comment the below line from test.c which is a part of memwatch archive.

```
/* Comment out the following line to compile. */
// error "Hey! Don't just compile this program, read the comments first!"
```

Next compile the software:

```
[root@server memwatch-2.71]# make
cc -DMEMWATCH -DMW_STDIO test.c memwatch.c
```

**HINT:**

Make sure you have all the compiling software installed on your Linux server such as `gcc`, `make` etc

Next we will create a dummy C program `memory.c` and add the `memwatch.h` include on *line 3* so that MEMWATCH can be enabled. Also, two compile-time flags `-DMEMWATCH` and `-DMW_STDIO` need to be added to the compile statement for each source file in the program.

```
[root@server memwatch-2.71]# cat memory.c
  1  #include 
  2  #include 
  3  #include "memwatch.h"
  4
  5  int main(void)
  6  {
  7    char *ptr1;
  8    char *ptr2;
  9
 10   ptr1 = malloc(512);
 11   ptr2 = malloc(512);
 12
 13   ptr2 = ptr1;
 14   free(ptr2);
 15   free(ptr1);
 16 }
```

 

The code shown in allocates two 512-byte blocks of memory (lines 10 and 11), and then the pointer to the first block is set to the second block (line 13). As a result, the address of the second block is lost, and a memory leak occurs.

Now compile the `memwatch.c` file, which is part of the MEMWATCH package with the sample source code (`memory1.c`). The following is a sample `makefile` for building `memory1.c`. `memory1` is the executable produced by this `makefile`:

```
[root@server memwatch-2.71]# gcc -DMEMWATCH -DMW_STDIO memory1.c memwatch.c -o memory1
```

Next we execute the program `memory1` which captures two memory-management anomalies

```
[root@server memwatch-2.71]# ./memory1
MEMWATCH detected 2 anomalies
```

MEMWATCH creates a log called `memwatch.log`. as you can see below, which is created by running the `memory1` program.

![memwatch log file with memory leak information](https://www.golinuxcloud.com/wp-content/uploads/2020/08/memwatch-log.jpg)memwatch log file with memory leak information

MEMWATCH tells you which line has the problem. For a free of an already freed pointer, it identifies that condition. The same goes for unfreed memory. The section at the end of the log displays statistics, including how much memory was leaked, how much was used, and the total amount allocated.

In the above figure you can see that the memory management errors occur on *line 15*, which shows that there is a double free of memory. The next error is a memory leak of 512 bytes, and that memory is allocated in *line 11*.

 

## 2. Valgrind

- Valgrind is an Intel x86-specific tool that emulates an x86-class CPU to watch all memory accesses directly and analyze data flow
- One advantage is that you don't have to recompile the programs and libraries that you want to check, although it works better if they have been compiled with the -g option so that they include debug symbol tables.
- It works by running the program in an emulated environment and trapping execution at various points.
- This leads to the big downside of Valgrind, which is that the program runs at a fraction of normal speed, which makes it less useful in testing anything with real-time constraints.

Valgrind can detect problems such as:

- Use of uninitialized memory
- Reading and writing memory after it has been freed
- Reading and writing from memory past the allocated size
- Reading and writing inappropriate areas on the stack
- Memory leaks
- Passing of uninitialized and/or unaddressable memory
- Mismatched use of `malloc`/new/new() versus free/delete/delete()

Valgrind is available in most Linux distributions so you can directly go ahead and install the tool

Advertisement

```
# rpm -Uvh /tmp/valgrind-3.15.0-11.el7.x86_64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:valgrind-1:3.15.0-11.el7         ################################# [100%]
```

Next you can execute `valgrind` with the process for which you wish to check for memory leak. For example I wish to check memory leak for `amsHelper` process which is executed with `-f` option. Press `Ctrl+C` to stop monitoring

```
# valgrind amsHelper -f
==30159== Memcheck, a memory error detector
==30159== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==30159== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==30159== Command: amsHelper -f
==30159==
NET-SNMP version 5.7.3 AgentX subagent connected
^C==30159==
==30159== HEAP SUMMARY:
==30159==     in use at exit: 853,777 bytes in 11,106 blocks
==30159==   total heap usage: 21,779 allocs, 10,673 frees, 28,226,706 bytes allocated
==30159==
==30159== LEAK SUMMARY:
==30159==    definitely lost: 73 bytes in 2 blocks
==30159==    indirectly lost: 0 bytes in 0 blocks
==30159==      possibly lost: 32,341 bytes in 120 blocks
==30159==    still reachable: 821,363 bytes in 10,984 blocks
==30159==         suppressed: 0 bytes in 0 blocks
==30159== Rerun with --leak-check=full to see details of leaked memory
==30159==
==30159== For lists of detected and suppressed errors, rerun with: -s
==30159== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

To save the output to a log file and to collect more details on the leak use `--leak-check-full` along with `--log-file=/path/og/log/file`. Press `Ctrl+C` to stop monitoring

```
# valgrind --leak-check=full --log-file=/tmp/mem-leak-amsHelper.log amsHelper -f
NET-SNMP version 5.7.3 AgentX subagent connected
```

Now you can check the content of `/tmp/mem-leak-amsHelper.log`. When it finds a problem, the Valgrind output has the following format:

```
==771== 72 bytes in 1 blocks are definitely lost in loss record 2,251 of 3,462
==771==    at 0x4C2B975: calloc (vg_replace_malloc.c:711)
==771==    by 0x6087603: ??? (in /usr/lib64/libnss3.so)
==771==    by 0x60876A8: ??? (in /usr/lib64/libnss3.so)
==771==    by 0x6087268: ??? (in /usr/lib64/libnss3.so)
==771==    by 0x607C11A: ??? (in /usr/lib64/libnss3.so)
==771==    by 0x60808C5: ??? (in /usr/lib64/libnss3.so)
==771==    by 0x602A269: ??? (in /usr/lib64/libnss3.so)
==771==    by 0x602AA80: NSS_InitContext (in /usr/lib64/libnss3.so)
==771==    by 0x550F3BA: rpmInitCrypto (in /usr/lib64/librpmio.so.3.2.2)
==771==    by 0x52CBF8D: rpmReadConfigFiles (in /usr/lib64/librpm.so.3.2.2)
==771==    by 0x473C74: ??? (in /usr/sbin/amsHelper)
==771==    by 0x441E24: ??? (in /usr/sbin/amsHelper)
```

 

## 3. Memleax

One of the drawbacks of Valgrind is that you cannot check memory leak of an existing process which is where memleax comes for the rescue. I have had instances where the memory leak was very sporadic for amsHelper process so at times when I see this process reserving memory I wanted to debug that specific PID instead of creating a new one for analysis.

[memleax](https://github.com/WuBingzheng/memleax) debugs memory leak of a running process by attaching it. It hooks the target process's invocation of memory allocation and free, and reports the memory blocks which live long enough as memory leak, in real time. The default expire threshold is 10 seconds, however you should always set it by -e option according to your scenarios.

You can download memleax from the official [github repository](https://github.com/WuBingzheng/memleax)
Next memleax expects few dependencies which you must install before installing the memleax rpm

```
# rpm -Uvh /tmp/memleax-1.1.1-1.el7.centos.x86_64.rpm
error: Failed dependencies:
        libdwarf.so.0()(64bit) is needed by memleax-1.1.1-1.el7.centos.x86_64
        libunwind-x86_64.so.8()(64bit) is needed by memleax-1.1.1-1.el7.centos.x86_64
        libunwind.so.8()(64bit) is needed by memleax-1.1.1-1.el7.centos.x86_64
```

So I have manually copied these rpms from my official repository as this is a private network I could not use yum or dnf

```
# rpm -Uvh /tmp/libdwarf-20130207-4.el7.x86_64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:libdwarf-20130207-4.el7          ################################# [100%]

# rpm -Uvh /tmp/libunwind-1.2-2.el7.x86_64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:libunwind-2:1.2-2.el7            ################################# [100%]
```

Now since I have installed both the dependencies, I will go ahead and install memleax rpm:

```
# rpm -Uvh /tmp/memleax-1.1.1-1.el7.centos.x86_64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:memleax-1.1.1-1.el7.centos       ################################# [100%]
```

Next you need the PID of the process which you wish to monitor. You can get the PID of your process from `ps -ef` output

```
root      2102     1  0 12:29 ?        00:00:01 /sbin/amsHelper -f
root     45256     1  0 13:13 ?        00:00:00 amsHelper
root     49372 44811  0 13:23 pts/0    00:00:00 grep amsH
```

Now here we wish to check the memory leak of `45256` PID

```
# memleax 45256
Warning: no debug-line found for /usr/sbin/amsHelper
== Begin monitoring process 45256...
CallStack[1]: memory expires with 688 bytes, backtrace:
    0x00007f0bc87010d0  libc-2.17.so  calloc()+0
    0x00000000004079d3  amsHelper
    0x0000000000409249  amsHelper
    0x0000000000407077  amsHelper
    0x00000000004a7a60  amsHelper
    0x00000000004a8c4c  amsHelper
    0x00000000004afd90  amsHelper
    0x00000000004ac97a  amsHelper  table_helper_handler()+2842
    0x00000000004afd90  amsHelper
    0x00000000004bae09  amsHelper
    0x00000000004bb707  amsHelper
    0x00000000004bb880  amsHelper
    0x00000000004bbca2  amsHelper
    0x00000000004e7eb1  amsHelper
    0x00000000004e8a3e  amsHelper
    0x00000000004e98a9  amsHelper
    0x00000000004e98fb  amsHelper
    0x00000000004051b4  amsHelper
    0x00007f0bc869d555  libc-2.17.so  __libc_start_main()+245
    0x00000000004053e2  amsHelper
CallStack[1]: memory expires with 688 bytes, 2 times again
CallStack[1]: memory expires with 688 bytes, 3 times again
CallStack[1]: memory expires with 688 bytes, 4 times again
CallStack[1]: memory expires with 688 bytes, 5 times again
CallStack[2]: memory expires with 15 bytes, backtrace:
    0x00007f0bc87006b0  libc-2.17.so  malloc()+0
    0x00007f0bc8707afa  libc-2.17.so  __GI___strdup()+26
    0x00007f0bc8731141  libc-2.17.so  tzset_internal()+161
    0x00007f0bc8731b03  libc-2.17.so  __tz_convert()+99
```

You may get output similar to above in case of a memory leak in the application process. Press `Ctrl+C` to stop monitoring

 

## 4. Collecting core dump

It helps for the developer at times we can share the core dump of the process which is leaking memory. In Red Hat/CentOS you can collect core dump using `abrt` and `abrt-addon-ccpp`
Before you start make sure the system is set up to generate application cores by removing the core limits:

```
# ulimit -c unlimited
```

Next install these rpms in your environment

```
# yum install abrt abrt-addon-ccpp abrt-tui
```

Ensure the ccpp hooks are installed:

```
# abrt-install-ccpp-hook install
# abrt-install-ccpp-hook is-installed; echo $?;
```

Ensure that this service is up and the ccpp hook to capture core dumps is enabled:

```
# systemctl enable abrtd.service --now
# systemctl enable abrt-ccpp.service --now
```

Enable the hooks

```
# abrt-auto-reporting enabled
```

To get a list of crashes on the command line, issue the following command:

```
# abrt-cli list
```

But since there are no crash the output would be empty. Next get the PID for which you wish to collect core dump, here for example I will collect for PID 45256

```
root      2102     1  0 12:29 ?        00:00:01 /sbin/amsHelper -f
root     45256     1  0 13:13 ?        00:00:00 amsHelper
root     49372 44811  0 13:23 pts/0    00:00:00 grep amsH
```

Next we have to send `SIGABRT` i.e. `-6` kill signal to this PID to generate the core dump

```
# kill -6 45256
```

Next you can check the list of available dumps, now you can see a new entry for this PID. This dump will contain all the information required to analyse the leak for this process

```
# abrt-cli list
id 2b9bb9702d83e344bc940b813b43262ede9d9521
reason:         amsHelper killed by SIGABRT
time:           Thu 13 Aug 2020 02:29:27 PM +0630
cmdline:        amsHelper
package:        hp-ams-2.10.0-861.6.rhel7
uid:            0 (root)
Directory:      /var/spool/abrt/ccpp-2020-08-13-14:29:27-45256
```

 

## 5. How to identify memory leak using default Linux tools

We discussed about third party tools which can be used to detect memory leak with more information in the code which can help the developer analyse and fix the bug. But if our requirement is just to look out for process which is reserving memory for no reason then we will have to rely on system tools such as `sar`, `vmstat`, `pmap`, `meminfo` etc

So let's learn about using these tools to identify a possible memory leak scenario. Before you start you must be familiar with below areas

1. How to check the actual memory consumed by individual process
2. How much memory reservation is normal for your application process

If you have answers to above questions then it will be easier to analyse the problem.

For example in my case I know the memory usage of amsHelper should not be more than few MB but in case of memory leak the memory reservation goes way higher than few MB. if you have read my [earlier article where I explained different tools to check actual memory usage](https://www.golinuxcloud.com/check-memory-usage-per-process-linux/), you would know that Pss gives us an actual idea of the memory consumed by the process.

 

`**pmap**` would give you more detailed output of memory consumed by individual address segments and libraries of the process as you can see below

```
# pmap -X $(pgrep amsHelper -f)
15046:   /sbin/amsHelper -f
         Address Perm   Offset Device  Inode   Size   Rss   Pss Referenced Anonymous Swap Locked Mapping
        00400000 r-xp 00000000  fd:02   7558   1636  1152  1152       1152         0    0      0 amsHelper
        00799000 r--p 00199000  fd:02   7558      4     4     4          4         4    0      0 amsHelper
        0079a000 rw-p 0019a000  fd:02   7558     52    48    48         48        20    0      0 amsHelper
        007a7000 rw-p 00000000  00:00      0    356    48    48         48        48    0      0
        01962000 rw-p 00000000  00:00      0   9716  9716  9716       9716      9716    0      0 [heap]
    7fd75048b000 r-xp 00000000  fd:02   3406    524   320    44        320         0    0      0 libfreeblpriv3.so
    7fd75050e000 ---p 00083000  fd:02   3406   2048     0     0          0         0    0      0 libfreeblpriv3.so
    7fd75070e000 r--p 00083000  fd:02   3406      8     8     8          8         8    0      0 libfreeblpriv3.so
    7fd750710000 rw-p 00085000  fd:02   3406      4     4     4          4         4    0      0 libfreeblpriv3.so
    7fd750711000 rw-p 00000000  00:00      0     16    16    16         16        16    0      0

<output trimmed>

    7fd75ba5f000 rw-p 00022000  fd:02   4011      4     4     4          4         4    0      0 ld-2.17.so
    7fd75ba60000 rw-p 00000000  00:00      0      4     4     4          4         4    0      0
    7ffdeb75d000 rw-p 00000000  00:00      0    132    32    32         32        32    0      0 [stack]
    7ffdeb79a000 r-xp 00000000  00:00      0      8     4     0          4         0    0      0 [vdso]
ffffffffff600000 r-xp 00000000  00:00      0      4     0     0          0         0    0      0 [vsyscall]
                                             ====== ===== ===== ========== ========= ==== ======
                                             196632 15896 13896      15896     10384    0      0 KB
```

Alternatively you can get the same information with more details using `smaps` of the respective process. Here I have written a small script to combine the memory and get the total but you can also remove the pipe and break down the command to get more details

```
# cat /proc/$(pgrep amsHelper)/smaps | grep -i pss |  awk '{Total+=$2} END {print Total/1024" MB"}'
14.4092 MB
```

So you can put a cron job or create a daemon to timely monitor the memory consumption of your application using these tools to figure out if they are consuming too much memory over time.

 

## Conclusion

In this tutorial I shared different commands, tools and methods to detect and monitor memory leak across different types of applications such as C or C++ programs, Linux applications etc. The selection of tools would vary based on your requirement. There are many other tools such as YAMD, Electric fence, gdb core dump etc which can help you capture memory leak.

Lastly I hope the steps from the article to check and monitor memory leak on Linux was helpful. So, let me know your suggestions and feedback using the comment section.
