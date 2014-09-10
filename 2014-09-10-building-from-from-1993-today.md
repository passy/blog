# Building Vim from 1993 today

Because of a conversation I had on Twitter earlier today, I felt compelled to compile the oldest version of Vim I could find.

<blockquote class="twitter-tweet" lang="en"><p><a href="https://twitter.com/sindresorhus">@sindresorhus</a> <a href="https://twitter.com/eugeniyoz">@eugeniyoz</a> I&#39;m too curious to leave at this. Gonna try to compile the first release from 1991 this afternoon. </p>&mdash; Pascal Hartig (@passy) <a href="https://twitter.com/passy/status/509672650609033216">September 10, 2014</a></blockquote>

As it turns out, compiling a C program from more than 20 years ago is actually a lot easier than getting a Rails app from last year to work. If you want to try it yourself, these are the steps you need to take:


```bash
$ docker run -i -t mugen/ubuntu-build-essential bin/bash
$ apt-get install -y wget gdb file
$ wget ftp://[ftp.vim.org/pub/vim/old/vim-1.24.tar.gz](http://ftp.vim.org/pub/vim/old/vim-1.24.tar.gz)
```


According to [Wikipedia](http://en.wikipedia.org/wiki/Vim_(text_editor)), vim 1.22, released in 1992, was the first version of vim to run on Unix and thus compete with vi. I couldn’t find a tarball of that release, so I went with vim 1.24 instead which was released a year later.


```bash
$ tar -xf vim-1.24.tar.gz
$ cd vim-1.24/src
$ make -f makefile.unix

unix.c: In function 'mch_settmode':
unix.c:361:23: error: storage size of 'ttybold' isn't known
unix.c:362:20: error: storage size of 'ttybnew' isn't known
unix.c:366:12: error: 'TIOCGETP' undeclared (first use in this function)
unix.c:366:12: note: each undeclared identifier is reported only once for each function it appears in
unix.c:368:25: error: 'CRMOD' undeclared (first use in this function)
unix.c:368:33: error: 'ECHO' undeclared (first use in this function)
unix.c:369:23: error: 'RAW' undeclared (first use in this function)
unix.c:370:12: error: 'TIOCSETP' undeclared (first use in this function)
```


Edit `makefile.unix` and change `CC` to `cc -g` so we can run gdb later. Remove `-DTERMCAP` from `DEFS` and `-ltermcap` from `LIBS`. A friend [found a version](https://twitter.com/dbanck/status/509701635098951680) of libtermcap that was almost working, but in the end I got a bunch of linking errors and decided to drop it altogether.


```bash
$ rm *.o # There’s no clean task.
$ make -f makefile.unix

unix.c: In function 'mch_settmode':
unix.c:361:23: error: storage size of 'ttybold' isn't known
unix.c:362:20: error: storage size of 'ttybnew' isn't known
unix.c:366:12: error: 'TIOCGETP' undeclared (first use in this function)
unix.c:366:12: note: each undeclared identifier is reported only once for each function it appears in
unix.c:368:25: error: 'CRMOD' undeclared (first use in this function)
unix.c:368:33: error: 'ECHO' undeclared (first use in this function)
unix.c:369:23: error: 'RAW' undeclared (first use in this function)
unix.c:370:12: error: 'TIOCSETP' undeclared (first use in this function)
make: *** [unix.o] Error 1
```


A new error, this time in `unix.c`. It turns out that the preprocessor condition that switches between Linux/BSD and System V includes is the wrong way around. By changing `#ifdef SYSV` to `#ifndef SYSV` we can make it work.


```bash
$ rm *.o
$ make -f makefile.unix

cc -o mkcmdtab mkcmdtab.o
mkcmdtab cmdtab.tab cmdtab.h
make: mkcmdtab: Command not found
make: *** [cmdtab.h] Error 127
```


Almost there! I didn’t know operating systems other than windows include the cwd in their `PATH`, but we can fix that easily:


```bash
$ PATH=.:$PATH make -f makefile.unix

cc -o ../vim alloc.o unix.o buffers.o charset.o cmdline.o csearch.o digraph.o edit.o fileio.o help.o linefunc.o main.o mark.o message.o misccmds.o normal.o ops.o param.o quickfix.o regexp.o regsub.o screen.o script.o search.o storage.o tag.o term.o undo.o version.o
$ file ../vim
../vim: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=0x4aed816b36b1b4b65cbbd3a599bd547a338f5b53, not stripped
```


Boom! We now have a 64bit binary of vim 1.24. However, the joy doesn’t last long, because as soon as we try to execute it, we see this:

[https://twitter.com/passy/status/509704282103222272](https://twitter.com/passy/status/509704282103222272)

Luckily, we compiled vim with debugging symbols, so lets turn to `gdb` for help:


```bash
$ gdb ./vim
(gdb) r
Program received signal SIGSEGV, Segmentation fault.
0x000000000040e326 in expand_env (src=0x42067e "$HOME/.vimrc", dst=0x62b010 "",
    dstlen=1025) at misccmds.c:694
694                     *tail = NUL;
(gdb) bt
#0  0x000000000040e326 in expand_env (src=0x42067e "$HOME/.vimrc", dst=0x62b010 "",
    dstlen=1025) at misccmds.c:694
#1  0x00000000004079c7 in dosource (fname=0x420683 "/.vimrc") at cmdline.c:2121
#2  0x000000000040c016 in main (argc=1, argv=0x7fffffffed60) at main.c:278
(gdb)
```


Vim is trying to expand `$HOME/.vimrc` to an absolute path and tries to overwrite a character of the string with `\0` which leads to a crash. I don’t exactly know why. My uneducated guess would be that this area of the memory is marked as read-only, which we could certainly turn off through a GCC flag. But if we go up the stacktrace to the main function, we can spot a nice shortcut:


```c

/*
 * Read the VIMINIT or EXINIT environment variable
 *              (commands are to be separated with '|').
 * If there is none, read initialization commands from "s:.vimrc" or "s:.exrc".
 */
        if ((initstr = getenv("VIMINIT")) != NULL || (initstr = getenv("EXINIT")) != NULL)
                docmdline((u_char *)initstr);
        else if (dosource(SYSVIMRC_FILE)) /* main.c:278 */
                dosource(SYSEXRC_FILE);
```


The crash happens when we source the `SYSVIMRC_FILE` which we can actually prevent from happening by setting the environment variable `VIMINIT`.


```bash
$ export VIMINIT="”
$ ./vim

~
~
~
~
Empty Buffer
```


That’s it! We have a running version of vim 1.24 on a modern Linux computer. You can convince yourself by typing `:version` and enjoying the fact that this version still went under the name of **Vi IMitation** before it was renamed to **Vi IMproved** with version 2.0.  And hey, it can even [load files bigger than 2MB](https://twitter.com/passy/status/509711664543838209).
