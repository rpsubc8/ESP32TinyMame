<h1>Hardware Sonido</h1>
La máquina Capcom Section Z dispone de:
<ul>
 <li>CPU Z80 6 Mhz para el juego</li>
 <li>
 CPU Z80 4 Mhz para el sonido
  <ul>
   <li>YM2203 a 1.5 Mhz (mono)</li>
   <li>YM2203 a 1.5 Mhz (mono)</li>
  </ul>
 </li> 
</ul>

Es decir, se necesitan 2 CPU's Z80, una para el juego, y la otra para el sonido:
>	for (activecpu = 0;activecpu < totalcpu;activecpu++)<br>
>	{<br>
>		int cycles;<br>
<br><br>


<h1>YM2203</h1>
El YM2203 es un chip OPN, que en la parte PSG es similar a un AY8210 con 3 canales, mientras que en la parte OPN dispone de 3 canales FM con osciladores.<br>
Los efectos, como disparon, bombas y explosiones, van a la parte PSG, es decir, a los 3 canales de una emulación básica AY8210.<br>
Los efectos del jugador 1 van a uno de los 2 AY8210, mientras que el jugador 2, al otro. Cuando ocurren explosiones, se usan ambos.<br>


<br><br>
<h1>Emulación</h1>
No solo nos podemos saltar la emulación PSG AY8210, sino también la parte FM YM2203, e incluso la emulación de la segunda CPU Z80, ya que tenemos acceso a los comandos de la posición de memoria 0xF80C.
