---
layout: post
title: "Fix JetBrains IDEA IntelliJ not working on bspwm (gray window issue)"
date: 2021-07-20 01:00:00
categories:
    - blog
tags:
    - linux
    - ricing
    - java
---

An easily reproducible issue I've recently been running into is **JetBrains IDEs not being displayed properly in [bspwm\[0\]](https://github.com/baskerville/bspwm) environment**. Instead of IDE graphical interface, only a gray solid window is being shown.

There's nothing innovatory about this fix, however it seems like it has never been mentioned in this particular context, meaning **running IntelliJ on bspwm and xorg**.

<!--break-->

This issue has been described in [Troubleshooting section of Java article on ArchLinux wiki\[1\]](https://wiki.archlinux.org/title/Java#Gray_window,_applications_not_resizing_with_WM,_menus_immediately_closing), although it doesn't mention the fix method for bspwm.

What needs to be done is **to export `_JAVA_AWT_WM_NONREPARENTING=1`**. Just like in the solution for Sway desktop environment.

The easiest way to achieve this in xorg environment is to add this line in `~/.xinitrc`:

```
export _JAVA_AWT_WM_NONREPARENTING=1
```

## Links
~~~
[0]: https://github.com/baskerville/bspwm
[1]: https://wiki.archlinux.org/title/Java#Gray_window,_applications_not_resizing_with_WM,_menus_immediately_closing
~~~

[0]: https://github.com/baskerville/bspwm
[1]: https://wiki.archlinux.org/title/Java#Gray_window,_applications_not_resizing_with_WM,_menus_immediately_closing