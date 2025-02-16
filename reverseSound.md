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

>	for (activecpu = 0;activecpu < totalcpu;activecpu++) <br>
>	{ <br>
>		int cycles; <br>
<br><br>


<h1>YM2203</h1>
El YM2203 es un chip OPN, que en la parte PSG es similar a un AY-3-8912 con 3 canales, mientras que en la parte OPN dispone de 3 canales FM con osciladores.<br>
Los efectos, como disparos, bombas y explosiones, van a la parte PSG, es decir, a los 3 canales de una emulación básica AY-3-8912<br>
Los efectos del jugador 1 van a uno de los 2 AY-3-8912, mientras que el jugador 2, al otro. Cuando ocurren explosiones, se usan ambos.<br>
Disponemos por tanto de (3+3)+(3+3)= 12 canales. El canal de ruido si se sigue la emulación extricta de un AY-3-8912, saldrá por 1 de los 3 canales, pero si disponemos de CPU host suficiente y queremos darle más calidad lo podemos añadir a la mezcla, de manera que nos daría 14 canales.<br>
El YM2203 MAME lo gestiona como un AY-3-8912, siempre que la escritura a los registros sea inferior a 16, por medio de la función <b>AYWriteReg</b> en el <b>psg.cpp</b>. En cuanto sea un registro superior, se gestionaria la parte FM (osd_ym2203_write).
<br>
La parte PSG de AY-3-8912, consta de:<br><br>

 | Registro | Funcion                        | Rango        |
 |----------|--------------------------------|--------------|
 | 0        | Channel A fine tone period     | 8-bit(0-255) |
 | 1        | Channel A coarse tone period   | 4-bit(0-15)  |
 | 2        | Channel B fine tone period     | 8-bit(0-255) |
 | 3        | Channel B coarse tone period   | 4-bit(0-15)  |
 | 4        | Channel C fine tone period     | 8-bit(0-255) |
 | 5        | Channel C coarse tone period   | 4-bit(0-15)  |
 | 6        | Noise period                   | 5-bit(0-31)  |
 | 7        | Mixer                          | 8-bit        |
 | 8        | Channel A volume               | 4-bit(0-15)  |
 | 9        | Channel B volume               | 4-bit(0-15)  |
 | 10       | Channel C volume               | 4-bit(0-15)  |
 | 11       | Envelope fine period           | 8-bit(0-255) |
 | 12       | Envelope coarse period         | 8-bit(0-255) |
 | 13       | Envelope shape                 | 4-bit(0-15)  |

He realizado una medición por estadísticas, y por lo que he mirado, en el juego no se hace uso de la envolvente, así que podemos saltarnos su recreación.

<br><br>
<h1>Sonido x86 SDL</h1>
En PC, la tarjeta de sonido siempre hace uso de un buffer interno DMA que no se puede modificar, dado que bloquearía el resto de hardware. Lo que si se permite es utilizar otro buffer y una llamada periódica que rellena nuestro buffer auxiliar, que posteriormente se envía al buffer interno de la tarjeta de sonido.<br>
Este buffer dependerá de la velocidad de muestreo, así como de las veces que vaya llamándose para rellenar, de manera que puede aparecer el temido problema de fallos de sonidos, por no coincidir ni los tiempos o incluso el relleno, por quedarse corto o excederse.<br>
Para solucionar esto, que es el problema de la mayoría de emuladores, he decidido ir a lo más práctico, es decir, el uso de osciladores, de manera, que cada vez que se invoque automáticamente <b>SDL_audio_callback</b>, no tengamos que estar preocupándonos del relleno ni del tiempo, dado que le pasamos los parámetros a generar en el buffer.<br>
Para el caso de querer un mezclador de 6 canales, es decir, tener 2 chips AY-3-8912, nos sirve algo similar:<br><br>

<pre>
 volatile unsigned int gb_cur_cont_ch[8]={0,0,0,0,0,0,0,0};
 volatile unsigned char gb_flipflop_ch[8]={0,0,0,0,0,0,0,0};
 volatile unsigned int gb_max_cont_ch[8]={1,1,1,1,1,1,1,1};
 
 void SDL_InitAudio()
 { 
  want.freq = 44100;
  want.format = AUDIO_S16SYS;
  want.channels = 1;
  want.samples = 1024;
  want.callback = SDL_audio_callback;
  want.userdata = &sample_nr;
  ...
 }



 void SDL_audio_callback(void *user_data, Uint8 *raw_buffer, int bytes)
 {
  if (bytes == 0){
   return;      
  }

  Sint16 *buffer = (Sint16*)raw_buffer;
  unsigned int length = (bytes>>1);
  unsigned int &sample_nr(*(unsigned int*)user_data); 
  double auxMix=0;

  for(unsigned int i = 0; i < length; i++, sample_nr++) 
  {
   for (unsigned char ch=0;ch<6;ch++)
   {        
    gb_cur_cont_ch[ch]++;
    unsigned int auxMax= (gb_flipflop_ch[ch]==0) ? (gb_max_cont_pos_ch[ch]-0): (gb_max_cont_neg_ch[ch]-0);
       
    if (gb_cur_cont_ch[ch] >= auxMax)
    {                              
     gb_cur_cont_ch[ch]=0;
     
     gb_flipflop_ch[ch]++;
     gb_flipflop_ch[ch]= (gb_flipflop_ch[ch] & 0x01);     
    }
   }// fin for ch

   auxMix=0;
   for (unsigned char ch=0;ch<6;ch++)
   {
    if ((gbVolMixer_now[ch]!=0) && (gbVol_canal_now[ch]!=0))
    {
     int vol= (int)(gbVol_canal_now[ch])*250;
     auxMix+= (gb_flipflop_ch[ch]==1)? vol:-vol;
    }
   }
                            
   buffer[i]= (Sint16)(auxMix);
  } 
 }
</pre>

En el <b>Legendary Wings</b>, no se hace uso del duty cycle de sonido, como podría ser el caso de la APU de la NES, es decir, que la onda de sonido AY-3-8912, además de ser cuadrada, tiene la misma duración de la parte positiva, que la negativa, lo que nos facilita los cálculos.<br>
Si por ejemplo queremos:<br><br>

> Frecuencia 1000 Hz <br>
>  44100 Hz / 1000 Hz = 44 muestras <br>
>  44 / 2 = 22 muestras positivo y 22 negativo. <br>
<br>
El cambio de positivo y negativo se encarga internamente el <b>gb_flipflop_ch</b>. Así que nosotros sólo tenemos que controlar el bucle con el número de canales, que en este caso son 6, tanto del cambio de flipflop, como de la mezcla.<br>
El <b>gbVolMixer_now</b> controla el mexclador de cada canal, de forma que si está a 0, ese canal está en silencio, es decir, no se mezcla, ni se procesa.<br>
El <b>gbVol_canal_now</b> controla el volumen de cada canal, de forma que es algo parecido al gbVolMixer_now.<br><br>
En esta función de relleno de buffer no se calcula cuantas muestras son positivas ni negativas dada una fecuencia, dado que ya se le pasa ese cálculo. Para saberlo, se tiene que calcular previamente, en el <b>_AYUpdateChip</b> del <b>psg.cpp</b>:<br><br>
<pre>
 unsigned int a= (PSG->Regs[AY_AFINE]+((unsigned int)(PSG->Regs[AY_ACOARSE]&0xF)<<8));
 //_AYUpdateChip clk:1500000000 rate:43920
 a = a ? AYClockFreq / AYSoundRate * 4 / a : 0;
 a=a>>2; //Para que suene más grave.
</pre><br>
Este ejemplo es para tener la frecuencia del canal A, en concreto de 1 de los 2 AY-3-8912. Pero nosotros necesitamos convertir esa frecuencia en los datos para nuestro oscilador:<br><br>

> gb_max_cont_pos_ch[0]= (a!=0)? SAMPLE_RATE/a/2 : 0;  //44100/a/2 <br>
> gb_max_cont_neg_ch[0]= gb_max_cont_pos_ch[0]; <br>
<br>

Para la mezcla,es tan sencillo como hacer una simple suma, teniendo en cuenta:<br><br>

> flipflop 1 (parte positiva onda) - Valor máximo. <br>
> flipflop 0 (parte negativa onda) - Valor mínimo. <br>
<br>

Como estamos con 16 bits con signo, el máximo es positivo, mientras que el mínimo es negativo. Dado que trabajamos con valores bajos, por mucho que sumemos, no vamos a sobrepasar el valor de 32767, por lo que no necesitamos realizar un clippping (recorte).<br><br>

Para el caso del ruido, nos viene por el registro 6 del AY-3-8912, es decir, su canal de ruido:<br>
<pre>
 unsigned int noise= PSG->Regs[AY_NOISEPER];
 noise= noise ? AYClockFreq / AYSoundRate * 4 / noise : 0;
 //noise= noise ? 1500000 / (16*noise) : 0;
 noise=noise>>5;
</pre><br>
Para la rutina de osciladores, aunque puede usarse la función <b>rand</b>, lo más optimizado es hacer uso de trucos.<br>
Nosotros sabemos la frecuencia, pero debemos aplicar valores aleatorios para la parte positiva, y lo mismo para la negativa, respetando el cruce por 0, así como la frecuencia de muestreo:<br><br>
<pre>
 unsigned char gb_aRand[16]={5,1,9,1,4,1,2,1,16,1,7,1,13,1,6,1}; //0 a 15
 unsigned char gb_contRand=0;
 static unsigned int g_seed=0;
 
 inline unsigned int fast_rand()
 {
  g_seed = ((214013 * g_seed) + 2531011);
  return (g_seed>>16)&0x7FFF;
 }

 
 ...

 
 //En la rutina de mezcla de canales
 
  int vol= (int)(gbVol_canal_now[ch]) * (250/8) * ((gb_aRand[gb_contRand])+1); //De 1 a 14
  if ((i&0x07)==0)      
  {//44100 DIV 8000 = 5 lo dejo en cada 6 el cambio de aleatorio
   gb_contRand++;
   if (gb_contRand>15)
   {
    gb_contRand= gb_contRand + fast_rand() & 0x0F;
    gb_contRand= (gb_contRand & 0x0F);
   } 
  }

  auxMix+= (gb_flipflop_ch[ch]==1)? vol:-vol;
 
</pre>


<br><br>
<h1>Sonido ESP32</h1>



<br><br>
<h1>Emulación</h1>
No solo nos podemos saltar la emulación PSG AY-3-8912, sino también la parte FM YM2203, e incluso la emulación de la segunda CPU Z80, ya que tenemos acceso a los comandos de sonido que envia la CPU Z80 principal a la de sonido, por medio de la posición de memoria <b>0xF80C</b> del contexto de la primera CPU, que puede verse en el driver <b>lwingsdriver.cpp</b>.<br><br>

<pre>
 static struct MemoryWriteAddress writemem[] = {
 {
   { 0xc000, 0xdeff, MWA_RAM },
   ...
   { 0xf80c, 0xf80c, sound_command_w },
   ...
 }
</pre><br>

Todo ello, se puede gestionar y realizar traza, desde código MAME en <b>genericsndhrdw.cpp</b>, en donde se encuentra la llamada a la función <b>sound_command_w</b>:<br>

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
