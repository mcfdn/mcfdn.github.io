---
layout: post
title: Patching the Bash Shellshock bug on Debian 6 Squeeze
description: News of the Shellshock bug is spreading fast. If you haven't heard about it already, you should definitely read up on it
---
News of the Shellshock bug is spreading fast. If you haven't heard about it already, you should definitely [read up on it](http://www.troyhunt.com/2014/09/everything-you-need-to-know-about.html).

Until more information is released, all we can do is make sure we patch Bash as best we can. Debian 7 (Wheezy) users have should already have received the Bash patch having run the standard:

    $ apt-get update
    $ apt-get install bash

However, since Debian 6 (Squeeze) is no longer supported, a simple `apt-update` will not suffice. We'll need to add locations of the Debian Squeeze archives so we can get Bash patched.

These are the steps to get the most recent patch for Bash on Debian 6 (Squeeze):

    $ sudo vim /etc/apt/sources.list

Add the following lines:

    deb http://http.debian.net/debian/ squeeze main contrib non-free
    deb-src http://http.debian.net/debian/ squeeze main contrib non-free

    deb http://security.debian.org/ squeeze/updates main contrib non-free
    deb-src http://security.debian.org/ squeeze/updates main contrib non-free

    deb http://http.debian.net/debian squeeze-lts main contrib non-free
    deb-src http://http.debian.net/debian squeeze-lts main contrib non-free

Write the file and run:

    $ apt-get update
    $ apt-get install bash

Test by running:

    $ env x='() { :;}; echo vulnerable' bash -c 'echo hello'

You should receive the following:

    bash: warning: x: ignoring function definition attempt
    bash: error importing function definition for `x'
    hello

### Futher Reading

[https://wiki.debian.org/LTS/Using](https://wiki.debian.org/LTS/Using)
