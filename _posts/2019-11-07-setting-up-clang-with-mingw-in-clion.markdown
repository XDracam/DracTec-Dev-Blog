---
layout: post
title:  "Setting up Clang with MinGW-w64 in Clion on Windows"
date:   2019-11-07 14:40:13 +0100
categories: cpp
---

# Why CLion and Clang?

Not everyone develops C++ for Windows only. And not everyone is a huge fan of Visual Studio's 'intelli'sense, which punishes a typo by randomly filling in some class you never wanted to hear of. However, Windows 10 still provides an *excellent* environment for development, thanks to the amazing WSL (soon to [run on an actual Linux kernel](https://chariotsolutions.com/blog/post/running-linux-on-windows-with-wsl-2-and-a-native-kernel/)).

For developing in most languages, I use Visual Studio Code: it has an elegant WSL integration and plugins for almost everything. This is fine, as long as you stick to the default tool chains. But as soon as you leave your comfort zone, numerous new technologies come into play. For example: when you develop Java in an enterprise context, you'll quickly decide for an application server and/or framework (like JEE, Spring or Seam). Without a special toolchain, developing for an application server quickly becomes tedious. Ideally you want one button to redeploy your code to the server and open the main page. IDEs like Eclipse and IntelliJ provide simple, preconfigured toolchains for many servers and frameworks, among other support for less common environments.

C++ is another story entirely. I feel like people are used to developing for their favorite platform only. Linux disciples like to stick to vim or emacs, use gcc as their toolchain of choice and install dependencies globally through their favorite package manager. Windows fans open up Visual Studio, create a new solution, `#pragma once` everything and use vcpkg for dependencies. Mac users have XCode etc. These approaches are obviously each not very portable. So what *do* you do when you want a portable, modern C++ 17 project? 

The first step would be to pick an IDE that's properly supported on every platform. In my humble opinion, Clion is a great choice. It doesn't matter what I try: in the end I always come back to trusty JetBrains IDEs.

So what about the compiler? I guess everyone has their favorite choice, but I feel like Clang is the way to go: It is dashingly fast, provides nice hints through Clang Tidy and is designed to support custom tooling from the ground up. And it can even produce binaries which are compatible to MSVC binaries. It is also the default compiler for Max OSX.

# Apparently Clang support is not perfect yet

Chances are that you got here because you are already trying to set up Clang in Clion. And you might have come to the point where everything seemed to work, but the debugger casually ignored every single breakpoint you placed. This happens because Clion [does not support a Windows installation of Clang](https://intellij-support.jetbrains.com/hc/en-us/community/posts/360006497860-Clion-) yet:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Ah yes, you need to install Clang from MinGW. Another options are not supported yet</p>&mdash; JetBrains CLion IDE (@clion_ide) <a href="https://twitter.com/clion_ide/status/1192101960700305408?ref_src=twsrc%5Etfw">November 6, 2019</a></blockquote>

Actually, installing LLVM with the Windows installer does work, as long as you can live without a working debugger and with freezes in regular intervals (hint: you can't). Right now, Clion supports running Clang with:

- MinGW
- Cygwin
- The WSL

Searching the internet quickly yields [this link](https://www.jetbrains.com/help/clion/quick-tutorial-on-configuring-clion-on-windows.html), describing how to configure Clion on Windows using one of the above environments (or MSVC). However, it doesn't explicitly mention Clang. Worse, it only shows how to install MinGW 32 bit, which limits application memory to 4GB. Oof. We also don't want Cygwin, since it's a rather weavyweight POSIX emulator. And we *also* don't want to use the WSL, since (at least in my case) our application needs to run on Windows (and compilation is slower on an emulated OS). 

# Setting up Clang with MinGW-w64

Angrily google for a few hours and you might find [this support forum tutorial](https://intellij-support.jetbrains.com/hc/en-us/community/posts/206606735-Using-Clang-With-CLion-on-Windows?page=1#community_comment_115000631284). Don't worry, you don't have to read it. Especially since it is slightly outdated. But it provides a good start.

# Step 1: Using MYSYS2 to install MinGW-64 and dependencies

We start by installing MYSYS2.

> MSYS2 is a software distro and building platform for Windows

Okay, whatever, as long as I can get started with my C++ project. MSYS2 conveniently enables us to install MinGW-w64 and dependent spftware using the `pacman` package manager. You might know pacman from being the default package manager on Arch Linux. This is in fact what people mean when they say:

> install Foo through MinGW

So, let's get started!

First, install MSYS2 from the [official page](http://www.msys2.org/). Make sure you install the `x86_64` installer, and **not** the `i686` installer. There is nothing special about this installer — I am sure that you can complete the installation without a series of screenshots 😉. In the last step of the installer, you are asked to open the MSYS2 console after finishing. You should. But don't worry: the installer registers MSYS2 to the start menu. You can just hit the Windows key, type `MSYS2` and start the console that way.

We now want to install Clang and other dependencies which Clion needs, using MSYS2 and its builtin pacman package manager. Now, using pacman is not very intuitive, as it relies on single-letter flags. You can enter `pacman -Sh` for a help page. But for basic functionality, you just need to enter the following command:

```
pacman -Sy mingw-w64-x86_64-llvm mingw-w64-x86_64-clang mingw-w64-x86_64-make mingw64/mingw-w64-x86_64-gdb
```

This installs the newest versions of MinGW-w64, LLVM, Clang, Make and GDB. I assume that you will be using Clion's bundled CMake, so there is no need to install that here. To upgrade to newer versions at a later date, you may open MYSYS2 again and type `pacman -Syu`. 

# Step 2: Configuring Clion to generate proper Configurations

I will just assume that you are using CMake to configure and build your project, as it is the de-facto portable standard and the default format for projects created in Clion. I am telling you this because the tutorial post linked above asks you to modify your CMake settings, whereas some online sources even prompt you to modify CMakeLists.txt files directly. But editing the CMake settings in CLion does nothing, and hard-coding local paths to compiler installation in your portable build files is a **terrible** idea. 

What we are going to do instead is to tell CLion which toolchain to use. For this, we open up the CLion settings (`File -> Settings...`) and search for `Toolchains`. You will be greeted by a list of toolchain configurations on the left and environment-dependent settings on the right. You might even already have a toolchain configuration (like `Visual Studio`) in there. 

What you want to do now is create a new configuration. For this, just click the `+` button next to the list. This copies the currently selected configuration, which we now edit:

1. Call the new configuration `MinGW-w64 Clang`.
2. From the `Environment` dropdown, pick `MinGW`.
3. Next to the dropdown, insert the path to your MinGW installation. If you kept the default path, this should be `C:\msys64\mingw64`.
4. Make sure that `CMake` is set to `Bundled`.
5. Leave the `Make` field empty and make sure it is autodetected.
6. Set the `C Compiler` to `C:\msys64\mingw64\bin\clang.exe` and the `C++ Compiler` to `C:\msys64\mingw64\bin\clang++.exe` (default is gcc).
7. Set the `Debugger` to `C:\msys64\mingw64\bin\gdb.exe` if it is not already found automatically. 
8. Select your new configuration and click the up-arrow until it is at the top of the list and is postfixed with `(default)`.

If you followed all of these steps correctly, your settings page should look pretty much like this:

![CLion Toolchains Settings for Clang](/DracTec-Dev-Blog/assets/clion-clang-toolchain.png)

Apply and Close. Then execute `Tools -> CMake -> Reset Cache and Reload Project`.

Congratulations! 🎉 Clion should now generate run configurations for each CMake target. Happy coding!

# One last issue tho...

At the time of writing, the C++ standard library headers supplied with LLVM 9.2.0 have a bug on Windows: `#include <filesystem>` causes a compilation failure. To fix this, You need to move a few lines in a file. [Just follow this stackoverflow answer](https://stackoverflow.com/questions/57963460/clang-refuses-to-compile-libstdcs-filesystem-header). 


# Someone on the internet is wrong! I need to correct him!

People make mistakes, and so do I. If you have a suggestion, you can contact me via email or [leave me an issue on GitHub](https://github.com/XDracam/DracTec-Dev-Blog/issues). Thanks!