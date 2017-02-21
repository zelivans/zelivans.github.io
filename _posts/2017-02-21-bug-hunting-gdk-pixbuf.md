---
layout: post
title:  Bug hunting GDK-PixBuf
---

GDK-PixBuf is an image loading library, mainly used by GTK+. It was originally a part of [GDK](https://en.wikipedia.org/wiki/GDK) but it was split up to [it's own repository](https://git.gnome.org/browse/gdk-pixbuf/).

## A little on fuzzing
I recently had some time to read about a great deal of security vulnerabilities and other bugs found with fuzzing, and it occurred to me to try and fuzz some libraries and applications I use regularly on my Linux machine. After a while I got to gdk-pixbuf. It is a great fuzzing target, because it is both a widely used library and it deals with various types of inputs.

My fuzzer of choice for this was [american fuzzy lop (afl)](http://lcamtuf.coredump.cx/afl/). It is very straight forward, it is well documented, and among other benefits it should be able to generate different file formats by itself, to reach full code coverage ([for example](https://lcamtuf.blogspot.co.il/2014/11/pulling-jpegs-out-of-thin-air.html)). There's more to it, but afl is not the topic of this post.

To start fuzzing, I had to either make a binary that uses gdk-pixbuf or pick something that uses it. I used gdk-pixbuf-thumbnailer which is a part of the gdk-pixbuf repository (originally a part of gnome-desktop that [had been moved to an external thumbnailer binary](https://git.gnome.org/browse/gdk-pixbuf/commit/?id=06cf4c78067203b78acbfb29862350cdb8200b73)). It is called by gnome-desktop to generate thumbnails for images. 

Before [this commit](https://git.gnome.org/browse/gnome-desktop/commit/?id=b69fde6f4a709f8c4fa3087941cee5b4c4169a8d) gnome-desktop fell back to thumbnail the file only if no external thumbnailer was found for the file's format (in other words, gdk-pixbuf-thumbnailer exists because the developers wanted to move the internal gdk-pixbuf thumbnailing process to an external binary). Vulnerabilities had been found in one of these external thumbnailers, evince-thumbnailer.[^evince]

To really get the best out of afl, the source code of the target must be compiled with an afl wrapped compiler (afl-gcc or afl-clang). These compilers inject afl instrumentation during compilation. Even better is afl-clang-fast which does "true compiler-level instrumentation, instead of the more crude assembly-level rewriting approach taken by afl-gcc and afl-clang".[^afl-clang-fast] 

The best way to compile Gnome projects from trunk is with [JHbuild](https://developer.gnome.org/jhbuild/stable/introduction.html.en). It's great because it allows compilation of the most recent Gnome internals without breaking anything (everything it builds go to the configured JHBuild path). I added `os.environ['CC'] = '/usr/bin/afl-clang-fast'` and `os.environ['CXX'] = '/usr/bin/afl-clang-fast++'` to my `~/.config/jhbuildrc` and compiled it with `jhbuild build gdk-pixbuf`. Now I had a gdk-pixbuf-thumbnailer binary that is ready to be fuzzed.

There are some other steps I did before actually starting afl with the binary, but as this is getting long already, I won't elaborate. If you are interested in using optimizing and afl though, read [this post about afl](https://foxglovesecurity.com/2016/03/15/fuzzing-workflows-a-fuzz-job-from-start-to-finish/).

## First bug
A crash! and another! Not a second passed before afl found 4 unique crashes. A quick investigation with gdb revealed there was a bug in the gnome-thumbnailer-skeleton.c file in the main function. It would try and print `error->message` when error was not set (it was NULL). In other words, it tried to print whatever was in address 0x8. As far as I know, this is harmless on a user space program such as this. But it's definitely a problem, so [I filed a bug](https://bugzilla.gnome.org/show_bug.cgi?id=778204).

## First interesting bug
I let afl run for a while until it found a new unique crash. It was a segmentation fault. I examined it with gdb:

{% highlight plaintext %}
Program received signal SIGSEGV, Segmentation fault.
DecodeHeader (Data=0x806caf8 "", Bytes=<optimized out>, State=<optimized out>, error=<optimized out>) at /home/tester/jhbuild/checkout/gdk-pixbuf/gdk-pixbuf/io-ico.c:362
362			if ((BIH[16] != 0) || (BIH[17] != 0) || (BIH[18] != 0)
{% endhighlight %}

This time the error is on io-ico.c. In the gdk-pixbuf folder there are different io-*.c files, each of these is a loader for a different file format. That is the decoder for Windows icon file format. I didn't have to investigate much to find out what the problem was:

{% highlight c %}
BIH = Data+entry->DIBoffset; 

/* A compressed icon, try the next one */
if ((BIH[16] != 0) || (BIH[17] != 0) || (BIH[18] != 0)
	|| (BIH[19] != 0)) {
{% endhighlight %}

I thought that's about it, and proceeded to try and figure out if this bug could be used for anything besides crashing the binary. But something then bothered me - when I compiled the binary for debugging (with -O0 flag), I was suddenly no longer seeing any segmentation faults.

I was sure I got it wrong, so I tried a few more times, compiled it back with optimization flags (the default is -O2) and it crashed. Otherwise I would get this error: `** (gdk-pixbuf-thumbnailer:12409): WARNING **: Could not thumbnail '/home/tester/crashes/test.ico': Invalid header in icon (header size)`. So somehow compiler optimizations allowed the bug to happen!

I began isolating individual compilation flags (that are added by -O2) to identify exactly what was the cause of the trouble. After a short while, I got it right. `-O1 -fstrict-overflow -ftree-vrp` were the flags that, when combined, let the crash happen.[^optimization-flags] If you read the gcc documentation on `-fstrict-overflow`, it explains this flag is used to allow the compiler to assume integer overflow never happens. `ftree-vrp` "allows the optimizers to remove unnecessary range checks like array bound checks and null pointer checks." (from the gcc manual).

Let's see what caused the error print on the good runs:
{% highlight c %}
State->HeaderSize = entry->DIBoffset + INFOHEADER_SIZE;

if (State->HeaderSize < 0) {
	g_set_error (error,
	             GDK_PIXBUF_ERROR,
	             GDK_PIXBUF_ERROR_CORRUPT_IMAGE,
	             _("Invalid header in icon (%s)"), "header size");
	return;
}
{% endhighlight %}

This now partly makes sense. If the compiler assumes that integer overflow never happens, we get undefined behavior by definition when an overflow does happen. This is possible on the first operation that sets `State->HeaderSize`, since `INFOHEADER_SIZE` is defined to 40 and we have full control of `entry->DIBoffset`. From a quick disassembly of the binary, it seems the overflow check is just left out.

I tested this on Arch Linux and later on Ubuntu 16.04.1 and it crashed eog and nautilus. Luckily, the code doesn't write anything to BIH but only reads from it. So to conclude this bug, it may lead to an out-of-bounds read to an address which we control, or otherwise lead to undefined behavior.

[Link to bug report](https://bugzilla.gnome.org/show_bug.cgi?id=779012)

## Finding more bugs
With afl running in the background, I started roughly reviewing the code manually. After some playing around with the loaders I found a bug in io-icns.c, the loader for Macintosh icons. There is a possible integer overflow in the size variable that is used later in a call to `gdk_pixbuf_loader_write`. This function is the general loading function, it finds the correct loader and calls it to decode the image in the buffer given to it. Let's take a look at it's signature:

{% highlight c %}
/**
 * gdk_pixbuf_loader_write:
 * @loader: A pixbuf loader.
 * @buf: (array length=count): Pointer to image data.
 * @count: Length of the @buf buffer in bytes.
 * @error: return location for errors
 ...
 **/
gboolean
gdk_pixbuf_loader_write (GdkPixbufLoader *loader,
			 const guchar    *buf,
			 gsize            count,
                         GError         **error)
{% endhighlight %}

The underflow is in the `load_resources` function which is called to decode the icns. There are multiple possible underflows in the `*mlen =` and `*plen =` lines. The first case:

{% highlight c %}
if (memcmp (header->id, "ic08", 4) == 0	/* 256x256 icon */
	|| memcmp (header->id, "ic09", 4) == 0)	/* 512x512 icon */
{
	*picture = (gpointer) (current + sizeof (IcnsBlockHeader));
	*plen = blocklen - sizeof (IcnsBlockHeader);
}
{% endhighlight %}

`sizeof(IcnsBlockHeader)` will always be 8, so if we have `blocklen < 8`, plen is underflown. Later `gdk_pixbuf_loader_write` is called with all the data after the icns header, and the plen/mlen as read to `count`.[^why_loader] So it is possible to call any loader with a huge (or negative, depending on type) count.

A malicious file would start like this:

{% highlight plaintext %}
0000h: 69 63 6E 73 00 00 00 10 69 63 30 38 00 00 00 07  icns....ic08....
{% endhighlight %}

`blocklen` here becomes 7, so `plen` will end up with the value of -1. If used as an unsigned variable (assuming 4 bytes) it's value will be 2<sup>32</sup>-1.

From a quick trial and error of this flaw with different image types I've found two out out-of-bounds reads. Both io-bmp.c and io-ico.c (which is actually based on io-bmp.c), when given a big count, continue to read from buf beyond it's real size, resulting in a segmentation fault.

## Infinite loop
Another interesting behavior is when trying the icns bug with a tiff image. It would hang, indefinitely. After digging into io-tiff.c, I found the culprit. The `make_available_at_least` function is called to allocate a buffer as large as the size we give it. Let's take a look on few lines this function:

{% highlight c %}
...
if (need_alloc > context->allocated) {
        guint new_size = 1;
        while (new_size < need_alloc)
                new_size *= 2;
...
{% endhighlight %}

The problem is that when new_size is large enough, multiplying it by 2 will overflow itself. Specifically, since it starts with 1, when it reaches 2<sup>31</sup>, the multiplication should result in 2<sup>32</sup> but that's too big for a 32 bit integer, so the result of the operation is 0.

Since this can be reproduced with a regular huge tiff file (without the icns vulnerability), [I filed a specific bug for this issue.](https://bugzilla.gnome.org/show_bug.cgi?id=779020)

## Other possibillites
By stacking two icns headers in a file, I was able to reach another out-of-bounds read. With the right filesize on the second header, I also caused another infinite loop (thanks to the out-of-bounds read returning zeros). With the same setup I was also able to cause the program to try allocate a buffer with the size I give it. 

The latter is also possible when a gif image or a tga image is placed after the icns header. Otherwise, if the image data is less than 4096, it would lead to another out-of-bounds read of the data.

[Link to bug report](https://bugzilla.gnome.org/show_bug.cgi?id=779016)

## What is affected?
There are many Linux user space applications that rely on gdk-pixbuf to load and display images. This also affects desktop managers themselves (the obvious is Gnome) and some file managers. To see the full list of programs affected run `apt-cache rdepends libgdk-pixbuf2.0-0 libgdk-pixbuf2.0-dev`  (on apt based distributions). I originally intended adding a demonstration of using one of these bugs maliciously, but I decided to leave that out for now.

I also found out these bugs can crash both Firefox and Chromium by trying to load a bad file from their file manager.

## Until next time
In my next post I will probably write about libwmf. This is another widely used library and I've already found some vulnerabilities in it. In the meantime I reccomend simply disabling it.

[^evince]: CVE-2010-2640, CVE-2010-2641, CVE-2010-2642, CVE-2010-2643
[^afl-clang-fast]: afl-2.39b/llvm_mode/README.llvm
[^optimization-flags]: Using `-O1 -fstrict-overflow` or `-O1 -ftree-vrp` wouldn't allow the crash.
[^why_loader]: The developer's intention was to load a JPEG 2000 image that usually follows the header for icons of size 256. However, no check is made that a real JPEG 2000 follows so we can abuse this call.