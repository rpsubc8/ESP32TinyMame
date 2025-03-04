<h1>Source code</h1>
The minimum version of MAME, which supports Legendary Wings, is MAME version 0.29 (20 Oct 1997):<br><br>
<a href='https://sourceforge.net/projects/mame/files/mame/0.29/mame029s.zip/download'>https://sourceforge.net/projects/mame/files/mame/0.29/mame029s.zip/download</a> (source code)<br>
<a href='https://sourceforge.net/projects/mame/files/mame/0.29/mame029b.zip/download'>https://sourceforge.net/projects/mame/files/mame/0.29/mame029b.zip/download</a> (binary emulator)<br>
<a href='https://www.mamedev.org/oldrel.html'>https://www.mamedev.org/oldrel.html</a> (old versions)<br>
<br>
This is a DJGPP-ready version of msdos, i.e. a protected memory extender, such as the CWSDPMI, is required.<br>
This handler was very slow at the time, and using DOSBOX, even if we speed up the waits, it will still be slow, so compiling it can be a pain in the ass from DOSBOX, but it will guarantee that it will compile well.<br>
DJGPP can only be run natively, up to the Windows XP version with 32-bit HAL support. For 64-bit and later operating systems, you can say goodbye to the native way.<br>
The most convenient option is to use a Windows 95 or 98 from VMWARE or VirtualBOX, so if we prepare an ISO CD with the MAME 0.29 source code, we can easily copy it to the virtual hard disk.<br>
Compiling from Windows, by skipping the CWSDPMI, even via virtual machine, is much faster than from DOSBOX, so it should be compiled within a minute.<br>

<br><br>
<h1>Steps to follow</h1>
If you follow this guide, even though it was not prepared for the same version, it is more than enough:<br>
<a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/compile.html'>https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/compile.html</a><br>
<br>
Therefore, it is needed:
<ul>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/unzip.exe'>unzip.exe</a> (140 kB)</li>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/djdev203.zip'>djdev203.zip</a> (1502 kB)</li> 
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/bnu2951b.zip'>bnu2951b.zip</a> (2508 KB)</li>  
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/gcc2952b.zip'>gcc2952b.zip</a> (1888 KB)</li>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/mak379b.zip'>mak379b.zip</a> (263 KB)</li>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/gnufut21.zip'>gnufut21.zip</a> (761 kB)</li>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/all3934.zip'>all3934.zip</a> (1789 KB)</li>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/zlib113.zip'>zlib113.zip</a> (214 KB)</li>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/mamesealnew.zip'>mamesealnew.zip</a> (69 KB)</li>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/nasm098.zip'>nasm098.zip</a> (160 KB)</li>
 <li><a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/zips/compile/upx107w.zip'>upx107w.zip</a> (117 KB)</li>
</ul>

To avoid problems, things to note:
<pre>
The nasm must use the msdos nasm098.zip.
Rename nasmw.exe to run on dosbox for 68000 CPU support.

unzip djdev203.zip -d c:\djgpp\
unzip bnu2951b.zip -d c:\djgpp\
unzip gcc2952b.zip -d c:\djgpp\
unzip mak379b.zip -d c:\djgpp\
unzip gnufut21.zip -d c:\djgpp\bin\
 
unzip upx107w.zip upx.exe -d c:\djgpp\bin\
unzip nasm098.zip nasmw.exe -d c:\djgpp\bin\
unzip mamesealnew.zip -d c:\djgpp\

unzip all3934.zip -d c:\djgpp\
 cd \djgpp\allegro
 make lib
 make install

unzip zlib113.zip -d zlib\
 cd zlib
 make -fmsdos\makefile.dj2
 xcopy libz.a c:\djgpp\lib\
 xcopy zlib.h c:\djgpp\include\
 xcopy zconf.h c:\djgpp\include\

make MAMEOS=msdos
make I686=1 MAMEOS=msdos
make K6=1 MAMEOS=msdos

MAMEOS has to be in capital letters.
In virtual Windows 98 and Windows ME it is required to activate OS managed virtual memory and thus compile.
If virtual memory is disabled, or if it is too small because the option managed by the OS is not activated, it will fail.

In mame01 you have to change in the makefile the sound library seal audiodjf.
In code in src msdos.c all sound is removed in code.
In the makefile the DEFS -DX86_ASM must be removed.
All in all, it compiles well.
 
Launch mame with options -nosound -nojoy
</pre>
From this guide, it is very important to have the allegro libraries, as well as mamesealnew for sound.


<br><br>
<h1>Porting to other platforms</h1>
What has been seen before, is to compile the original MAME 0.29 from and for MSDOS, even from a virtual machine.<br>
With this, you will be able to see the scope of it, debug or even make code modifications for your needs, all under MSDOS.<br>
If we want it to work on other platforms, such as Windows with SDL, ESP32 or RP2040, we must modify this code, removing the dependency on ALLEGRO and VESA for video, as well as mamesealnew for sound.<br>
