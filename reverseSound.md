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
El YM2203 es un chip OPN, que en la parte PSG es similar a un AY-3-8912 con 3 canales, mientras que en la parte OPN dispone de 3 canales FM con osciladores.<br>
Los efectos, como disparos, bombas y explosiones, van a la parte PSG, es decir, a los 3 canales de una emulación básica AY-3-8912<br>
Los efectos del jugador 1 van a uno de los 2 AY-3-8912, mientras que el jugador 2, al otro. Cuando ocurren explosiones, se usan ambos.<br>
Disponemos por tanto de (3+3)+(3+3)= 12 canales. El canal de ruido si se sigue la emulación extricta de un AY-3-8912, saldrá por 1 de los 3 canales, pero si disponemos de CPU host suficiente y queremos darle más calidad lo podemos añadir a la mezcla, de manera que nos daría 14 canales.


<br><br>
<h1>Emulación</h1>
No solo nos podemos saltar la emulación PSG AY-3-8912, sino también la parte FM YM2203, e incluso la emulación de la segunda CPU Z80, ya que tenemos acceso a los comandos de sonido que envia la CPU Z80 principal a la de sonido, por medio de la posición de memoria 0xF80C del contexto de la primera CPU.<br>
Todo ello, se puede gestionar desde código MAME en genericsndhrdw.cpp:<br>

>	 void sound_command_w (int offset,int data) <br>
>	 { <br>

Se envian siempre 2 comandos, que equivale al identificador del SAMPLE, seguido de otro comando con el valor 0xFF.<br>
Algunos de los comandos para los efectos SFX, serían:
<ul>
 <li>0x01 - SFX: Morir por explosion de enemigo</li>
 <li>0x04 - SFX: Disparo fuego</li>     
 <li>0x07 - SFX: Bomba</li>
 <li>0x0A - SFX: Explosion fuego enemigo</li> 
</ul>
Los efectos SFX, al final, aunque podriamos tenerlos sampleados, se traducen en llamadas a escrituras de puerto en los PSG AY-3-8912.<br>

Para las melodias, que podemos tener en SAMPLES WAV o crudos, serían:
<ul>
 <li>0x23 - Melodia 23.Screen Change</li>
 <li>0x25 - Melodia 01.Credit</li>
 <li>0x26 - Melodia 22.Game Over</li>
 <li>0x31 - Melodia 10.Underground</li>
 <li>0x36 - Melodia 21.Continue</li>
</ul>


<br><br>
<h1>Lista</h1>
Las melodías VGM, son una especie de MIDI:
<ul>
 <li>01. Credit 0:02</li>
 <li>02. Start Demo 0:06</li>
 <li>03. Game Start 0:08</li>
 <li>04. Area 1 1:13 + 0:59</li>
 <li>05. Area 2 1:05 + 0:59</li>
 <li>06. Area 3 1:30 + 1:27</li>
 <li>07. Area 4 2:02 + 1:44</li>
 <li>08. Area 5 2:04 + 1:47</li>
 <li>09. Bonus Area 1:12 + 1:10</li>
 <li>10. Underground 0:28 + 0:28</li>
 <li>11. Sanctuary 1:43 + 1:17</li>
 <li>12. Underground Boss 0:47 + 0:47</li> 	
 <li>13. Area Boss 0:25 + 0:09</li>
 <li>14. Sanctuary Boss 0:41 + 0:41</li>
 <li>15. Area Clear 1 0:06</li>
 <li>16. Area Clear 2 0:16</li>
 <li>17. Ranking 1 2:00</li>
 <li>18. Ranking 2 0:44 + 0:23</li> 	
 <li>19. Ranking Display 1 0:06</li>
 <li>20. Ranking Display 2 0:06</li>
 <li>21. Continue 0:12</li>
 <li>22. Game Over 0:07</li>
 <li>23. Screen Change 0:05</li>
 <li>24. Extend 0:02</li>		
</ul>
En Total: 16:58 + 11:46<br>
Se pueden extraer todas de:<br><br>
<a href='https://vgmrips.net/packs/pack/legendary-wings-arcade'>https://vgmrips.net/packs/pack/legendary-wings-arcade</a><br><br>
Así mismo, las melodías, como tienen calidad MIDI, se podrían resamplear algunas a 2000 Hz, y otras a 4000 Hz, sin perder calidad. Si no tenemos problemas de espacio, como es el caso de un ESP32, es decir, por ejemplo en un PC, se pueden dejar en 8000 Hz o más.<br>
Muchas melodias, son secuenciales, por ejemplo, después del    22. Game Over 0:07   viene la    21. Continue 0:12, salvo que consigamos Ranking.



<br><br>
<h1>Nivel</h1>
De lo que he podido depurar, el nivel de juego se puede detectar por 3 posiciones de memoria:
<ul>
 <li>0xC06C - Resetear (1 avanza el nivel)</li>
 <li>0xC06D - El nivel</li>
 <li>0xC06E - Finalizar (0 continuar, 1 morir)</li>
</ul>

El nivel propiamente dicho está en 0xC06D, pero si no se envia el set a 1 de las posiciones 0xC06C y 0 de 0xC06E, no se hará ninguna acción.<br>
Saber el nivel, es cómodo para poder saber que melodía debemos poner y para más situaciones.



<br><br>
<h1>Trucos</h1>
La información está sacada de:<br><br>
<center></center><a href='https://ryiron.wordpress.com/2019/10/14/legendary-wings-reversing-a-1980s-arcade-game/'>https://ryiron.wordpress.com/2019/10/14/legendary-wings-reversing-a-1980s-arcade-game</a></center><br><br>

| Memoria | Valor | Accion      |
|---------|-------|-------------|
| 0xC11B  | 1     | Invencible  |
| 0xC118  | 0x05  | Potenciador |

Con escribir en dicha posición de memoria desde las capas más altas del emulador, incluso cada segundo, se tendría.<br>
