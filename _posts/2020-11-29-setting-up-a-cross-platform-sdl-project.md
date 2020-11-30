---
layout: post
title: "Setting up a cross-platform SDL project"
tags: [programming, sdl, graphics]
date: 2020-11-29 19:15 -0800
description: A look at how to set up SDL everywhere.
---

One of the benefits of [SDL](https://www.libsdl.org/) is portability. You can drop in a couple of SDL headers and start writing code that renders images to the screen, captures input, and emits audio, like magic. The only problem is that when you take your code and try to build it on a different OS you may find the process a little confusing. Let's take a look at how to set up a C++ project with SDL you can build just about anywhere.

## macOS and Linux

Let's start with a POSIX environment like macOS or a Linux distribution. If you're running on macOS you wont have a package manager by default, but [Homebrew](http://brew.sh/) is a common choice and easy to use. Once installed. Once your environment has a package manager you can go ahead and install all of the SDL development libraries.

`$ brew install sdl2 sdl2_image sdl2_mixer sdl2_net sdl2_ttf`

Great, now we can begin. We'll use a simple `Makefile` for our project:

```
CXX=g++
SDL2CFLAGS=-I/usr/local/include/SDL2 -D_THREAD_SAFE

CXXFLAGS=-O2 -c --std=c++14 -Wall $(SDL2CFLAGS)
LDFLAGS=-L/usr/local/lib -lSDL2

exec: main.o
	$(CXX) $(LDFLAGS) -o exec main.o

main.o: main.cpp
	$(CXX) $(CXXFLAGS) main.cpp
```

The process here is simple, we build our `main.cpp` file into an object using our `SDL2` headers, then link it against the `SDL2` dynamic libraries. In `main.cpp` we can access `SDL_*` functions with a simple `#include <SDL2/SDL.h>` directive. The header files will tell the compiler how the API works, and the dynamic libraries will tell the linker how they're implemented.

But where did we get the `SDL2CFLAGS` and `LDFLAGS` values? Well we installed the SDL libraries we also got access to `sdl2-config`. You can use it like this:

```
$ sdl2-config --cflags
$ sdl2-config --libs
```

It's a handy way to find out where your package manager installed the files. Now, assuming you've got a C++ compiler, you can go ahead and run `make exec` to build your program.

## Windows

So we've got an SDL program that we wrote on macOS or Linux, and we want to share it with our friends running Windows. How do we make a build for them? Well we could install [MinGW](http://www.mingw.org/) which provides us with `g++` and `make`, but for my [project](https://github.com/rivergillis/chip-8) I found that the version of `g++` included here was lacking. It didn't support `std::thread` after installation. So much for standardization.

That leaves us with [Visual Studio](https://visualstudio.microsoft.com/), specifically Visual C++, or MSVC. Microsoft isn't good with naming these things. It's an IDE and a C++ compiler, tightly coupled. The process here is a little finnicky. You need to open your SDL project as a project without a "Solution" (which is going to replace our Makefile on Windows) so that VS can generate one for us. Skip past all of the dialogue (when prompted, indicate you're building a command line app) so that we can start configuring things.

Without a package manager on Windows you need to manually make your way [here](http://libsdl.org/download-2.0.php) and download the development libraries for Visual C++. Unzip those to a folder you'll remember later. As a tip, alter the SDL2 include folder structure to place all headers in a dir called `SDL2`. For some reason the Windows distribution of this package doesn't do that, which would mean your includes would look like `#include <SDL.h>` instead of `#include <SDL2/SDL.h>.`

Okay, last step. We need to tell VS how to use the libraries we just downloaded. Go to *Project -> Properties -> Configuration Properties -> VC++ Directories -> Include Directories -> Edit* and add the path to the `include` subdir from the SDL2 package you unzipped earlier. These are the header files. Then go to *Linker -> Additional Dependencies -> Edit* and add `SDL2.lib; SDL2main.lib;`. These are the dynamic libraries, but we still need to tell VS where those blobs are, so go to *VC++ Directories -> Include Directories -> Edit* and add the path to the `lib/x64` (or `x86` for 32-bit builds) subdir from the SDL2 package you unzipped earlier. Finally, save everything and build your project.

With that, you can sleep safely knowing your project will build just about anywhere. When you want to release on macOS or Linux, use the `Makefile`, and when you want to release on Windows, use the Visual Studio solution.