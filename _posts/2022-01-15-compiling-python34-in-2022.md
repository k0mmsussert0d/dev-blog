---
layout: post
title: "Compiling Python <3.5.3 in 2022"
date: 2022-01-15 00:00:00
categories:
 - blog
tags:
 - linux
 - python
---

Despite Python 3.6 reaching end-of-life just 3 weeks ago[\[0\]][0], the amount of projects depending on older versions of Python 3 is exorbitant. Whilst working on a part of my media library workflow I found myself in need of using an old stalled project based on Python 3.4. Sadly, the code does not execute properly in newer runtimes and no one has ever undertook the task of refactoring it. Therefore, users are forced to run it with 3.4 version interpreter.

<!--break-->

**tldr; use openssl 1.0 and gcc 10**

## Short introduction to *pyenv*

Before getting right into the the crux of the matter, I'd like to interject for a moment in order to familiarize dear reader with Python runtime management tool called *pyenv*. This entry is not closely connected with it, however chances of anyone still willing to manually juggle Python runtimes after learning about *pyenv* are near to none. It's just so convenient to use.

My case isn't one of its own. Python is notorious for providing poor interoperability between its different versions and the tool I've been using for years to mitigate caused inconveniences is [pyenv\[1\]][1]. Basically it allows you to install multiple version of Python runtime and switch between them with ease. Let's say you want to use version `3.7.12` – run `pyenv install 3.7.12`. As Python offers source code tarballs as the only official way of distribution, *pyenv* downloads the archive, builds the version of your choice and copies the output binaries to its own runtime directory. Then, switching to the specific version is as simple as executing `pyenv global 3.7.12`. From now on, `python` command should be resolved with `3.7.12` binary.

```
$ python --version
Python 3.7.12
```

*pyenv* is very powerful utility on its own, and paired with extension called `pyevn-virtualenv` has served me as a manager of runtimes for multiple Python projects. 

This time wasn't supposed to be different and I was very surprised when `pyenv install 3.4.10` greeted me with `ERROR: The Python ssl extension was not compiled. Missing the OpenSSL lib`.

## OpenSSL 1.0
As per [pyenv docs\[2\]][2], old Python versions (for CPython, <3.5.3 and <2.7.13) require OpenSSL 1.0, while most of the newer systems ship with 1.1.x version by default. However, most also provide 1.0.2 version available as a separate package to cover incompatibility issues like this. For my distribution, which is Arch Linux, the package is called `openssl-1.0`. Debian stretch and Ubuntu bionic users might use `libssl1.0-dev`.

```
$ pacman -S openssl-1.0
```

There is no need to uninstall package providing new version of OpenSSL. The compiler just needs to be told where to look for the headers. Run `pyenv install` again with the following compilation flags:

```
$ CFLAGS="-I/usr/include/openssl-1.0" LDFLAGS="-L/usr/lib/openssl-1.0" CPPFLAGS="-I/usr/include/openssl-1.0" pyenv install 3.4.10
```

Verify the paths of OpenSSL 1.0 library beforehand. The command above includes the ones applicable for default Arch Linux setup. They might look different for other distributions.

## SEGFAULT and other compilation errors
Compiling older version of Python on Arch Linux and other bleeding edge distros might be interrupted by unexpected compilation error. In my case it was classic segmentation fault. The key is to use older version of gcc. For *3.4.10* version, it's enough to go back to gcc 10, still available as a regular package in Arch Linux.

Install it:
```
$ pacman -S gcc10
```

And add another flag to `pyenv install` call – `CC=gcc-10`:
```
$ CC=gcc-10 CFLAGS="-I/usr/include/openssl-1.0" LDFLAGS="-L/usr/lib/openssl-1.0" CPPFLAGS="-I/usr/include/openssl-1.0" pyenv install 3.4.10
```

Compilation should now succeed.

## Summary
To wrap everything up – compiling older version of Python on modern systems usually require two tweaks:
- providing OpenSSL 1.0 library as an alternative to any other version installed on the system
- using GCC 10 instead of recently released GCC 11

In Arch Linux it all narrows down to just two commands:
```
pacman -S openssl-1.0 gcc10
CC=gcc-10 CFLAGS="-I/usr/include/openssl-1.0" LDFLAGS="-L/usr/lib/openssl-1.0" CPPFLAGS="-I/usr/include/openssl-1.0" pyenv install 3.4.10
```

Happy coding, scripting or whatever you need old Python for. Don't forget to consider refactoring the code to run correctly in newer runtime, as OpenSSL 1.0 embedded in legacy Python executable is EOL and any internet-facing applications should not rely on it anymore.

## Links
~~~
[0]: https://devguide.python.org/devcycle/#end-of-life-branches
[1]: https://github.com/pyenv/pyenv
[2]: https://github.com/pyenv/pyenv/wiki/Common-build-problems#2-your-openssl-version-is-incompatible-with-the-python-version-youre-trying-to-install
~~~

[0]: https://devguide.python.org/devcycle/#end-of-life-branches
[1]: https://github.com/pyenv/pyenv
[2]: https://github.com/pyenv/pyenv/wiki/Common-build-problems#2-your-openssl-version-is-incompatible-with-the-python-version-youre-trying-to-install
~~~
