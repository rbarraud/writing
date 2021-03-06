![header](https://filebox.tymoon.eu//file/TVRJM05RPT0=)  
As announced in my previous post, I've decided to do a write-up that illustrates my adventures in developing [Portacle](https://shinmera.github.io/portacle), a cross-platform, portable, installer-less, self-contained Common Lisp development environment.

The reason why I started with portacle in the first place is that, some day, probably in the distant future, I want to write a long series of long articles that should eventually accumulate into a book that introduces complete novices and interested people to programming in Common Lisp. As a prerequisite for that, I deem it absolutely necessary that the readers have an easy way to try out examples. Forcing them to perform a strange setup process that has absolutely nothing to do with the rest of the book and is likely to be error-prone in multiple ways is completely out of the question to me. They should be able to download an archive, extract it, and then just click on an icon to get a fully-featured IDE with which they can start out and play around with.

So, right off the bat this sets a few constraints that are rather hard to deal with.

* It needs to be cross-platform. I can't constrain the users to a certain operating system. Requiring them to set up a virtual machine would be madness.
* It needs to be cross-distribution. I can't know which distribution of Linux, or which version of it users are going to have. Requiring a specific one or a specific version would, too, be madness.
* It needs to be self-contained and should not poison the rest of the environment unless necessary or intended. This is necessary for it to be truly portable and work from a single folder.
* It needs to be reproducible and upgradable in order to be future-proof. This means I cannot simply assemble a package by hand once and call it a day once it works. It needs to be automatically buildable.
* It needs to be able to run on all platforms simultaneously as a single distribution. Otherwise, you would not be able to put it onto a USB stick and use it from any machine.
* It needs to be fully-featured enough to provide for both quick experiments and larger projects. As such, it will need to package a few different components and make them all work alongside, with the above constraints on top.

Since I want the eventual article series to be for absolute beginners as well, I also had some serious concerns about the usability aspects. It shouldn't be too confusing or hindering to use. Despite this, I still settled for Emacs+SLIME for the IDE, simply because there isn't really any alternative out there that is free and supports Lisp to a sufficient degree. I'm still not entirely set on the way the Emacs customisation is currently laid out, though. I might have to make more or less severe changes to make it easier for people to get started with. But, that's something for another time.

Now, considering the last constraint listed above, I decided on the following packages to be included in Portacle:

* Emacs, for the primary editor.
* Emacs customisation files, since the default is horrid.
* SBCL, for a capable Lisp implementation.
* ASDF, for building and system management.
* Quicklisp, for package management and library access.
* Git, for version control and project sharing.

ASDF was included specifically so that I would not have to rely on the implementation updating its internally shipped version and could instead deliver one that is possibly ahead of it, and thus less buggy/more feature-rich.

The first goal was to figure out how to build everything on OS X, Linux, and Windows. This alone took its fair share of experimentation and scouring as I had to look for the right combination of feature flags and build flags to make things work minimally.

In order to make this process work, I've developed a series of bash scripts that take care of downloading the source files of the various parts, configuring them, building them, and installing them in the proper locations in the resulting Portacle distribution tree. I settled on the following layout:

```
portacle      -- Base Portacle directory
  build       -- Build scripts and artefacts
    $package  -- Build artefacts for the package
  config      -- Configuration files and user files
  projects    -- User projects directory
  all         -- Cross-platform packages and resources
    emacsd    -- Emacs configuration package
    quicklisp -- Quicklisp installation
  $platform   -- Platform-specific packages and resources
    $package  -- Installed package files
    lib       -- Shared library files
    bin       -- Shared executable files
```

The layout used to be different at first, namely the `$platform` and `$package` directories used to be flipped around in the hierarchy. That proved to be less than ideal, however. I won't go into the details as to why. Suffice to say that complications that cropped up along the way ended up poisoning the hierarchy. Having a platform directory for all the specific files means you can create a Portacle distribution that works on all platforms simultaneously too.

Now, in order to properly illustrate the problems that cropped up, I'm going to talk about the development process for each platform separately. Pretty much every one of them had unique problems and complications that lead to a lot of wasted time and questions about the universe.

## Linux
Linux was the first platform I started developing for, and ironically enough the last one I finished for good. The first phase of just getting everything built was comparatively painless. Things just worked.

However, as soon as I tried to run a complete installation on another Linux setup, things just segfaulted left and right. No wonder, too. After all, every component has shared library dependencies. Those are going to have different versions, and possibly even different names on different distributions. The first idea to deal with this mess was to simply copy every shared library the applications touched to a directory and set `LD_LIBRARY_PATH` in a wrapper script before the respective application was launched.

In order to do this, I extended the build scripts with tools that would statically analyse all the executables that were shipped and recursively trace a full list of shared library dependencies. It would then copy them all over to the shared library files directory.

After doing this, things worked a bit better. But only a tiny bit. SBCL worked now, at least. Everything else still crashed. Unfortunately however, SBCL also only worked on my particular system. Much later I would find out that on others it would still fail. The root of the problem lay in the fact that `LD_LIBRARY_PATH` does not cover everything. The system will still pick some shared libraries from elsewhere. Particularly, `libc` and so forth.

So I scoured the web in search of a solution that would somehow let me constrain where linux looked for shared libraries. After a long while, I finally found something: `ld-linux.so`. This file, which lives on every Linux system, is responsible for setting up the shared libraries an application depends on, and then running the application. It can be called directly, and it accepts a library-path argument, with which I can force it to look in a particular place first. So I changed the wrappers around to also start things with `ld-linux.so`. Doing this allowed me to get Emacs working.

However, now SBCL didn't work anymore. For some reason that is still completely beyond me, if I try to run SBCL under `ld-linux.so`, it acts as if the argument that is the SBCL binary path was simply not there. You can try it for yourself: `/lib64/ld-linux.so $(which sbcl)` will show you the help text, as if that argument was just not there. If you pass it the path twice it does something, but it won't work right either. Fortunately, since SBCL doesn't have many dependencies to speak of, I've been able to get by without wrapping it in `ld-linux.so`. I can't wait for the day for that to break as well, though.

Anyway, back to where I was. Emacs now worked. Kind of. Running other binaries with it did not, because the `LD_LIBRARY_PATH` would poison their loading. Regular binaries on the system somewhere would not execute. So I had to do away with that. Thanks to the `ld-linux.so` trick though, it wasn't really needed, at least for Emacs. For Git, the situation was worse. Git does some weird stuff where it has a multitude of `git-*` binaries that each perform certain tasks. Some of those tasks call other Git binaries. Some of them call bash or things like that. I couldn't set `LD_LIBRARY_PATH` because that would crash bash and others, and I couldn't use `ld-linux.so` because that doesn't work when a new process is created. Back to Google, then.

After even more endless searching I finally found a solution. Using `LD_PRELOAD` I could inject a shared library to be loaded before any other. This library could then replace the `exec*` family of libc system functions and ensure that the process you wanted to call was actually called under `ld-linux.so`. You can do this kind of symbol replacement by using `libdl`, fetching the function locations of the functions you want to replace using `dlsym` in the library's `init` function, and then defining your own versions that call out to the original functions. You can find a few interesting blog articles out there that use `LD_PRELOAD` and this kind of technique for malicious purposes. This culminated in something I called [`ld-wrap.so`](https://github.com/Shinmera/portacle-launcher/blob/master/ld-wrap.c). Getting all the `exec*` calls to work again was its own adventure. On that path I discovered that `execv` will actually call out to a shell if it finds a shell file, and, even worse than that, `excecvp*` will actually call a shell any time if they can't seem to execute the file regularly. I think it's pretty insane for a "low-level system call" to potentially invoke a shell to run a script file, rather than "just" executing an ELF binary. In fact, there is *no way* to "just" execute a binary without implementing your own system calls directly. That's bonkers.

Anyway, getting `ld-wrap.so` to work right involved multiple iterations, the last of which finally got Git working proper. I had to incorporate tests to check whether the binary was in the Portacle distribution, as well as checks to see whether the binary was static or not. Apparently launching a static binary under `ld-linux.so` just results in a segfault. How very C-like. Anyway, it has not been the least bit of fun to go through this and pretty much every step of getting all of this worked out cost me weeks.

Aside from the SBCL problem mentioned above, there's an outstanding issue regarding the usage of fonts in Emacs. Since there's absolutely no guarantee for any particular font to exist on the system, and since I'd like to provide for some nice-looking default, I need to ship and install fonts automatically. I haven't gotten that to work quite yet, but I've been very disappointed to find that Emacs apparently has no support for loading a font from a File, and that Linux and OS X don't really support "temporarily" loading a font either. You have to install it into a user directory for it to be visible, so I have to touch things outside of the Portacle distribution folder.

Finally, apparently if you try to compile Emacs on some Linux systems, it will fail to run under `ld-linux.so` and just crashes immediately with a "Memory Exhausted" warning. I have no clue what that is about and haven't received any feedback at all from the Emacs mailing lists either. So far this is not a problem, since the virtual system I build on works, but it is a major hassle because it means that building a distribution is out of the question for a lot of people. You can find out more about this bug on the [issue tracker](https://github.com/Shinmera/portacle/issues/18).

## Windows
Windows has been an interesting system to work with. On one hand it has given me a lot of problems, but on the other it has also avoided a lot of them. The biggest pleasure was that shared library deployment "just worked". No need for complicated `ld-linux` shenanigans, Windows is just consistent and faithful to loading the first library that it can find on its `PATH`. Given that most of the libraries I need are already either provided by the Windows system or from the self-contained MSYS installation, there really haven't been any issues with getting things running on a deployed system.

However, it makes up for this in other areas.

While Emacs and SBCL have been very easy to get compiled and running, Git has once again proven to be a problem child. First, Git insists on keeping around `git-*` prefixed hard-links to its binary because apparently a lot of things both internal and external still directly reference those. Hard links are difficult to ship because most archiving systems don't support them, making the resulting archive humongous. The current size of ~80 megabytes is already a lot by my measures, but the hard link issues exploded it to well over 100. Thus I had to go hunt for an archiver that allowed both self-extracting SFX executables --after all I couldn't ask people to install an archiver first-- and was able to compress well and handle hard links. 7zip was up to the task, but it required complicating the deployment quite a bit in order to get the SFX working.

Next, Git is primarily a system written for Linux. As such, it has some "interesting" ideas about what the filesystem really looks like. I'm still not entirely sure how the path translation happens, but apparently Git interprets the root of the filesystem somewhere relative to its application location. This is fortunate for me, but took a long while to figure out. It is fortunate, because Git needs CURL to work, and CURL needs a certificate file to be able to do HTTPS. Naturally, Windows does not provide this by itself, so I have to do it. Git does allow you to set the path of the certificate file through a configuration variable, but it was one hell of a journey to get the entire setup working right. I'll try to explain why.

Because of -- or rather thanks to Git's weird interpretation of the filesystem root, I can create a "fake" Git system configuration. Usually, Git looks up `/etc/gitconfig` as part of the configuration files. Now, since the root is sort of relative to the Git binary, I can create an `etc` subdirectory with the configuration file in the Git platform directory. This allows me to specify the certificate file path without having to disturb any of the other platforms. Then, thanks again to this root interpretation I can specify the path to the certificate file as an absolute path within the configuration file. Since the Git interpreted root moves with the Git binary, it becomes effectively relative. Naturally this would not work on Linux or OS X, but thankfully there I don't need to resort to such tricks.

Finally, if I try to compile Git without gettext on Windows, it fails to run properly and just exits with `vsnprintf is broken`. I did find some issues related to `vsnprintf` on Google, but nothing conclusive. Either way, if I do compile with gettext it seems to work fine. It doesn't work with gettext on OS X though, so I can't just enable it everywhere either.

Last but not least, Windows is primarily responsible for me not shipping a spell checker system. I really wanted to include that, as a spell checker is a really great thing to have when you're writing lots of comments and documentation, or just using Emacs for prose. However, I was simply not able to get `ispell`, `aspell`, or `hunspell` to compile and run no matter how much I tried. I was also unable to find any other compatible alternatives to those three that could be used as a replacement. If anyone else knows of a solution that I can compile successfully, and for which I can generate working dictionaries, I'd be very grateful.

Actually, I suppose it's also worth mentioning that for a while, before I wrote the [launcher](https://github.com/Shinmera/portacle-launcher), I was trying to use batch scripts to launch things. Please, don't ever try to do that. Batch files are unbelievably cryptic to read and a huge pain to work with. There's no easy way to pass the arglist to another program for example, as it'll just reinterpret spaces within an argument as an argument separator. If you can, use PowerShell, which is supposedly much better, or just write a proper launcher application that does that kind of logic.

## Mac OS X
Finally, OS X. This is a bit of a problematic system for me because Apple has, for reasons beyond me, decided to make it as difficult as possible to test for. Since I needed Portacle to work on multiple versions of OS X if possible, and even beyond that just ensure that it works outside of a development environment, I had to get my hands on virtual machines somehow. I first bought a MacBook Air just for this purpose --that's a thousand dollars wasted in the wind-- only to realise that trying to run any kind of Virtual Machine on it is futile because it's just unbearably slow. Fortunately there are ways to get VMWare Workstation on Linux to run OS X as well if you use specially prepared images and some nefarious tools to unlock OS X as an option. However, only versions 10.11+ seem to run at any usable speed. 10.10 is so slow that you can pretty much forget trying to use it for anything.

Anyway, even just setting up a suitable build and test environment proved to be a major hassle. Thanks to Homebrew and MacPorts the building aspect wasn't too bad, though there too I've found weird inconsistencies like the aforementioned gettext problem. Aside from the compilation though, it really bothers me a lot that the OS X coreutils lack so many useful features. The build scripts have several special cases just for OS X because some Unix utility lacks some option that I need to get it done succinctly or efficiently.

Aside from the virtualisation thing, Apple also seems to try their hardest to prevent anyone from developing software for their system without also paying them a substantial fee every year. If you want to launch Portacle as a normal user, you just get a "security" error message that blocks the application from running. To get it to launch, you have to start the systems settings application, navigate to the security tab, unlock it, then click on "run this application" in order to finally run it. They completely removed the option to allow foreign apps by default, too, so there's no to me visible option to make it shut up. Windows has a security popup similar to that too, but at least you can immediately tell it to launch it, rather than having to waste multiple minutes in menus to do so.

In addition to the "security" popup, Apple has also recently decided to activate a feature that makes it virtually impossible for me to properly ship additional libraries, or versions of the libraries that Portacle requires, thus forever constraining the possible versions it can run on. OS X will now automatically clear out `DYLD_LIBRARY_PATH` whenever you execute a new application. You can only disable this lunatic behaviour by booting into recovery mode and changing a security option-- definitely not something I can ask people to do just to use Portacle. Thus, it seems it is impossible for me to, in the long term, cover a wider version scheme than a single one. This is very annoying, especially given that lots of people seem to stop upgrading now that Apple is screwing up ever more with every new version.

Ah well.

## Future Outlook
That about sums up most of the major issues that I can remember. You can find out more stories of me going borderline insane if you browse around in the [issues](https://github.com/Shinmera/portacle/issues?utf8=✓&q=is%3Aissue) or the [commit history](https://github.com/Shinmera/portacle/commits/master).

Now that most of the platform problems are finished, Portacle is almost done. There are a few more things left to do, however. The Emacs configuration as it is right now is somewhat passable, but I'd like to add and change a few more things to make it more comfortable, especially for beginners. I don't want to change too much either though, as that would make it hard for beginners to find help to a specific issue online.

Otherwise, I'd also like to work a bunch more on the included documentation and help file. As it stands it gives an overview over almost everything one needs to know to get started, but I haven't had it run by anyone yet, so I can't really give any estimates as to whether it's comprehensible, readable, or even useful in the first place.

I might write about Portacle again another time, hopefully when it has finally reached something I can call "version 1.0". Until then, I'll try to write some more articles about other subjects.
