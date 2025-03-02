<h1>Código fuente</h1>
La versión mínima de MAME, que soporta el Legendary Wings, es la 0.29 del MAME (20 Oct 1997):<br><br>
<a href='https://sourceforge.net/projects/mame/files/mame/0.29/mame029s.zip/download'>https://sourceforge.net/projects/mame/files/mame/0.29/mame029s.zip/download</a> (código fuente)<br>
<a href='https://sourceforge.net/projects/mame/files/mame/0.29/mame029b.zip/download'>https://sourceforge.net/projects/mame/files/mame/0.29/mame029b.zip/download</a> (binario emulador)<br>
<a href='https://www.mamedev.org/oldrel.html'>https://www.mamedev.org/oldrel.html</a> (versiones viejas)<br>
<br>
Se trata de una versión de msdos, preparada para el DJGPP, es decir, que se necesita un extensor de memoria protegida, como el CWSDPMI.<br>
Este gestor, era muy lento en la época, y usando el DOSBOX, aunque aceleremos las esperas, seguirá siendo lento, por lo que compilarlo se puede hacer muy pesado desde el DOSBOX, pero garantizará que se compilará bien.<br>
El DJGPP sólo se puede ejecutar de manera nativa, hasta la versión de Windows XP con soporte HAL 32 bits. Para 64 bits y sistemas operativos posteriores, se puede decir adios a la manera nativa.<br>
La opción más cómoda es usar un Windows 95 o 98 desde VMWARE o VirtualBOX, de manera que si preparamos una ISO CD con el código fuente del MAME 0.29, podremos copiarlo fácilmente al disco duro virtual.<br>
Compilar desde Windows, al saltarse el CWSDPMI, incluso por medio de máquina virtual, es muchísimo más rápido que desde DOSBOX, por lo que en un minuto debería estar compilado.<br>

<br><br>
<h1>Pasos a seguir</h1>
Si se sigue esta guía, aunque no estaba preparado para la misma versión, sirve de sobra:<br><br>
<a href='https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/compile.html'>https://arcarc.xmission.com/Web%20Archives/ionpool.net%20(Dec-31-2020)/arcade/mame/compile.html</a><br>
<br>
Por tanto, se necesita:
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

Para evitar problemas, cosas a destacar:
<pre>
El nasm has que usar el nasm098.zip de msdos.
Rrenombrar a nasmw.exe para que funciona en dosbox para soporte CPU 68000

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

El MAMEOS tiene que ir en mayúsculas.
En virtual Windows 98 y Windows ME se requiere activar memoria virtual manejada por SO y de esta forma compila.
Si se tiene desactivada la memoria virtual, o es muy pequeña por no tener activada la opción de manejada por el SO, dará fallo.

El mame01 hay que cambiar en el makefile la libreria de sonido seal audiodjf.
En codigo en src msdos.c se quita en codigo todo el sonido.
En el makefile hay que quitar el DEFS -DX86_ASM
Con todo esto, se compila bien.
 
Lanzar mame con opciones -nosound -nojoy
</pre>
De esta guía, es muy importante disponer de las librerías allegro, así como mamesealnew para el sonido.


<br><br>
<h1>Portar a otras plataformas</h1>
Lo que se ha visto antes, es para compilar el MAME 0.29 original desde y para MSDOS, aunque sea desde máquina virtual.<br>
Con esto, se podrá ver el alcance del mismo, depurar o incluso realizar modificaciones de código para nuestras necesidades, todo bajo MSDOS.<br>
Si queremos que funcione en otras plataformas, como puede ser el caso de Windows con SDL, ESP32 o RP2040, debemos modificar este código, quitando la dependencia de ALLEGRO y VESA para video, así como mamesealnew para el sonido.<br>
