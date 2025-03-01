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
Los efectos del jugador 1, van a uno de los 2 AY-3-8912, mientras que el jugador 2, al otro. Cuando ocurren explosiones, se usan ambos.<br>
Disponemos por tanto de (3+3)+(3+3)= 12 canales. El canal de ruido si se sigue la emulación extricta de un AY-3-8912, saldrá por 1 de los 3 canales, pero si disponemos de CPU host suficiente y queremos darle más calidad lo podemos añadir a la mezcla, de manera que nos daría 14 canales.<br>
El YM2203 MAME lo gestiona como un AY-3-8912, siempre que la escritura a los registros sea inferior a 16, por medio de la función <b>AYWriteReg</b> en el <b>psg.cpp</b>. En cuanto sea un registro superior, se gestionaría la parte FM (osd_ym2203_write).
<br>
La parte PSG de AY-3-8912, consta de:<br><br>

 | Registro | Funcion                        | Nombre      | Rango        |
 |----------|--------------------------------|-------------|--------------|
 | 0        | Channel A fine tone period     | AY_AFINE    | 8-bit(0-255) |
 | 1        | Channel A coarse tone period   | AY_ACOARSE  | 4-bit(0-15)  |
 | 2        | Channel B fine tone period     | AY_BFINE    | 8-bit(0-255) |
 | 3        | Channel B coarse tone period   | AY_BCOARSE  | 4-bit(0-15)  |
 | 4        | Channel C fine tone period     | AY_CFINE    | 8-bit(0-255) |
 | 5        | Channel C coarse tone period   | AY_CCOARSE  | 4-bit(0-15)  |
 | 6        | Noise period                   | AY_NOISEPER | 5-bit(0-31)  |
 | 7        | Mixer                          | AY_ENABLE   | 8-bit        |
 | 8        | Channel A volume               | AY_AVOL     | 4-bit(0-15)  |
 | 9        | Channel B volume               | AY_BVOL     | 4-bit(0-15)  |
 | 10       | Channel C volume               | AY_CVOL     | 4-bit(0-15)  |
 | 11       | Envelope fine period           | AY_EFINE    | 8-bit(0-255) |
 | 12       | Envelope coarse period         | AY_ECOARSE  | 8-bit(0-255) |
 | 13       | Envelope shape                 |             | 4-bit(0-15)  |

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
El <b>gbVolMixer_now</b> controla el mezclador de cada canal, de forma que si está a 0, ese canal está en silencio, es decir, no se mezcla, ni se procesa.<br>
El <b>gbVol_canal_now</b> controla el volumen de cada canal, de forma que es algo parecido al gbVolMixer_now.<br><br>
En esta función de relleno de buffer no se calcula cuantas muestras son positivas ni negativas dada una fecuencia, puesto que ya se le pasa ese cálculo. Para saberlo, se tiene que calcular previamente, en el <b>_AYUpdateChip</b> del <b>psg.cpp</b>:<br><br>
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

Para el caso del canal B y C, es similar.<br>

Para la mezcla en los osciladores SDL,es tan sencillo como hacer una simple suma, teniendo en cuenta:<br><br>

> flipflop 1 (parte positiva onda) - Valor máximo. <br>
> flipflop 0 (parte negativa onda) - Valor mínimo. <br>
<br>

El mezclador del AY-3-8912, es el registro <b>AY_ENABLE</b>, y controla los 3 canales (0 activo, 1 silencio) en lógica negada:<br>
<pre>
 A - AY_ENABLE & 0x01 - gbVolMixer_now[0]= ((~AY_ENABLE) & 0x01)
 B - AY_ENABLE & 0x02 - gbVolMixer_now[1]= ((~AY_ENABLE) & 0x02)
 C - AY_ENABLE & 0x04 - gbVolMixer_now[2]= ((~AY_ENABLE) & 0x04)
</pre>

Como estamos con 16 bits con signo, el máximo es positivo, mientras que el mínimo es negativo. Dado que trabajamos con valores bajos, por mucho que sumemos, no vamos a sobrepasar el valor de -32768 o 32767, por lo que no necesitamos realizar un clippping (recorte).<br><br>

Para el caso del ruido, nos viene por el registro 6 del AY-3-8912, es decir, su canal de ruido (AY_NOISEPER):<br>
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
En el ESP32 haremos uso del DAC (GPIO 25), solucionando los problemas del I2S.<br>
La salida del DAC es siempre positiva (0 a 255) y aunque con 32000 Hz para el mezclador, tenemos de sobra, trataremos con 44100 Hz.<br>
El sistema es similar al uso de osciladores de SDL, es decir, se genera en tiempo real (0 lag), cambiando sólo frecuencias, pero además, no hacemos uso de ningún buffer:<br><br>

<pre>
  hw_timer_t *gb_timerSound = NULL;
  volatile unsigned char gb_spk_data= 0x80;
  volatile unsigned char gb_spk_data_before= 0x80;

  hw_timer_t *gb_timerPlayPoll = NULL;

  void IRAM_ATTR onTimerSoundDAC(void); 
  void IRAM_ATTR onTimerPlayPoll(void);
 
 void setup()
 {
  dac_output_enable(DAC_CHANNEL_1);
  CLEAR_PERI_REG_MASK(SENS_SAR_DAC_CTRL2_REG, SENS_DAC_CW_EN1_M);
  SET_PERI_REG_BITS(RTC_IO_PAD_DAC1_REG, RTC_IO_PDAC1_DAC, 0x7f, RTC_IO_PDAC1_DAC_S);

  gb_timerSound= timerBegin(0, 80, true); 
  timerAttachInterrupt(gb_timerSound, &onTimerSoundDAC, true);

  gb_timerPlayPoll = timerBegin(1,80,true);
  timerAttachInterrupt(gb_timerPlayPoll, &onTimerPlayPoll, true);

  //timerAlarmWrite(gb_timerSound, 125, true); //1000000 1 segundo  125 es 8000 hz
  //timerAlarmWrite(gb_timerSound, 90, true); //1000000 1 segundo  125 es 11025 hz
  //timerAlarmWrite(gb_timerSound, 62, true); //1000000 1 segundo  62 es 16000 hz
  //timerAlarmWrite(gb_timerSound, 45, true); //1000000 1 segundo  45 es 22050 Hz
  timerAlarmWrite(gb_timerSound, 22, true); //1000000 1 segundo  22 es 44100 Hz

  timerAlarmWrite(gb_timerPlayPoll, 1000, true); //1000000 1 segundo  1000 es 1000 Hz 1 ms
   
  timerAlarmEnable(gb_timerSound);
  timerAlarmEnable(gb_timerPlayPoll);  
 }

 

 void IRAM_ATTR onTimerPlayPoll()
 {
  Sonido_poll_play();
 }

 void Sonido_poll_play()
 {
  //Canal A
  gbVol_canal_now[0]= gb_latch_vol_pulse[0];  
  gb_max_cont_pos_ch[0]= gb_latch_pos_max_pulse[0];
  gb_max_cont_neg_ch[0]= gb_latch_neg_max_pulse[0];

  //Canal B
  gbVol_canal_now[1]= gb_latch_vol_pulse[1];  
  gb_max_cont_pos_ch[1]= gb_latch_pos_max_pulse[1];
  gb_max_cont_neg_ch[1]= gb_latch_neg_max_pulse[1];

  ...

  for (unsigned char i=0;i<6;i++)
  {
   gbVolMixer_now[i]= gb_latch_vol_mix[i];
  }
 }
 
 void IRAM_ATTR onTimerSoundDAC()
 {  
  int iSum;
  int vol;
  unsigned int auxMax;

  if (gb_spk_data != gb_spk_data_before)
  {
   SET_PERI_REG_BITS(RTC_IO_PAD_DAC1_REG, RTC_IO_PDAC1_DAC, gb_spk_data, RTC_IO_PDAC1_DAC_S);   //dac_output
   gb_spk_data_before= gb_spk_data;
  }

  iSum= 0;

  //Canal A
  gb_cur_cont_ch[0]++;
  auxMax= (gb_flipflop_ch[0]==0) ? (gb_max_cont_pos_ch[0]): (gb_max_cont_neg_ch[0]);
  if (gb_cur_cont_ch[0] >= auxMax)
  {                          
   gb_cur_cont_ch[0]=0;
        
   gb_flipflop_ch[0]++;
   gb_flipflop_ch[0]= (gb_flipflop_ch[0] & 0x01);
  }

  //Canal B
  gb_cur_cont_ch[1]++;
  auxMax= (gb_flipflop_ch[1]==0) ? (gb_max_cont_pos_ch[1]): (gb_max_cont_neg_ch[1]);
  if (gb_cur_cont_ch[1] >= auxMax)
  {                          
   gb_cur_cont_ch[1]=0;
        
   gb_flipflop_ch[1]++;
   gb_flipflop_ch[1]= (gb_flipflop_ch[1] & 0x01);
  }
  
  ...


  //MIXER
  //Canal A
  if ((gbVolMixer_now[0]!=0) && (gbVol_canal_now[0]!=0))
  {
   vol= (int)(gbVol_canal_now[0])<<1;
   iSum+= (gb_flipflop_ch[0]==1)? vol:-vol;
  }

  //Canal B
  if ((gbVolMixer_now[1]!=0) && (gbVol_canal_now[1]!=0))
  {
   vol= (int)(gbVol_canal_now[1])<<1;
   iSum+= (gb_flipflop_ch[1]==1)? vol:-vol;
  }    
  
  ...

  //Clipping
  if (iSum>127) {iSum=127;}
  else
  {
   if(iSum<-127) {iSum=-127;}
  }
  
  gb_spk_data= (iSum+0x80);
    
 }
</pre>

El sistema es similar a SDL, usando un registro latch intermedio, de forma que concurrentemente cada milisegundo se va mirando estado de dicho latch, el cual se va actualizando cada vez que se escribe en un registro del AY-3-8912.<br>
Si la rutina de emulación de CPU de un frame completo, es muy rápida, es decir, por debajo de 8 milisegundos, puede que no necesitamos usar un timer con el latch cada 1 milisegundo.<br>
El uso de la rutina con timer de tiempo real para los osciladores, al usar el DAC sin I2S con DMA, consume un poco más de CPU, pero a cambio solucionamos el problema con los diferentes frameworks de Espressif (sin solución) del uso del DAC interno. No obstante, si en lugar de usar este sistema, usaramos una escalera de resistencias R2R con salida GPIO o bien una comunicación I2C con un chip (Atmega328) o DAC exterior, quedaría también solucionado, e incluso sería mejor, dado que es más rápido que el DAC interno del ESP32.<br>


<br><br>
<h1>Emulación</h1>
En <b>msdos.cpp</b> se encuentra la variable <b>play_sound</b>, de manera que si está en valor 0, se dejará de procesar la segunda CPU Z80, y por tanto, se dejará de emular tanto el YM2203 la parte FM, como la parte PSG del AY-3-8912, es decir, se dejará de emular el sonido al 100%.<br>
Si por el contrario, <b>play_sound</b> tiene el valor 1, se emulará el Z80 de sonido, y por tanto se recogerán las llamadas al PSG del AY-3-8912. En este caso, podemos emular el AY-3-8912 o no, dado que veremos la relación que existe entre los SFX's  y los VGM's.
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

Se envian siempre 2 comandos, que equivale al identificador del VGM (SAMPLE), seguido de otro comando con el valor 0xFF.<br>
Algunos de los comandos para los efectos SFX, serían:<br>

| CMD  | Tipo | Descripción                    |
|------|------|--------------------------------|
| 0x01 | SFX  | Morir por explosion de enemigo |
| 0x04 | SFX  | Disparo fuego                  |
| 0x07 | SFX  | Bomba                          |
| 0x0A | SFX  | Explosion fuego enemigo        |

<br>

Los efectos SFX, al final, aunque podríamos tenerlos sampleados, se traducen en llamadas a escrituras de puerto en los PSG AY-3-8912.<br>
Si depuramos  y dejamos traza, en concreto en <b>_AYUpdateChip</b> de <b>psg.cpp</b>, podemos o bien dejar todas la escrituras a registros del AY-3-8912, o mejor, sólo con los cálculos de la frecuencia (A,B,C y ruido), volumen (A,B,C) y canales de mezcla:<br>

<pre>
 soundw off:0x00 d:0x04 pend:0x00 QUEUE:0x0A (SAMPLE Fire)
 soundw off:0x00 d:0xFF pend:0x00 QUEUE:0x0A
         A    B    C   Noise       A  B  C
  freq: 0411 0371 0853 0000  vol: 14 14 14  mix: 07 time: 12366  ms: 0
  freq: 0344 0355 0711 0000  vol: 14 14 14  mix: 07 time: 12382  ms: 16
  freq: 0296 0341 0609 0000  vol: 14 14 14  mix: 07 time: 12398  ms: 32
  freq: 0260 0328 0533 0000  vol: 14 14 14  mix: 07 time: 12414  ms: 48
  freq: 0232 0319 0481 0000  vol: 14 14 14  mix: 07 time: 12430  ms: 64
  freq: 0242 0322 0494 0000  vol: 14 13 13  mix: 07 time: 12446  ms: 80
  freq: 0282 0331 0559 0000  vol: 14 13 13  mix: 07 time: 12462  ms: 96
  freq: 0338 0344 0644 0000  vol: 14 13 13  mix: 07 time: 12478  ms: 112
  freq: 0294 0352 0578 0000  vol: 13 12 13  mix: 07 time: 12495  ms: 129
  freq: 0279 0359 0559 0000  vol: 13 12 13  mix: 07 time: 12510  ms: 144
  freq: 0313 0371 0644 0000  vol: 13 12 13  mix: 07 time: 12526  ms: 160
  freq: 0264 0363 0726 0000  vol: 13 12 11  mix: 07 time: 12542  ms: 176
  freq: 0238 0355 8538 0000  vol: 12 12 10  mix: 07 time: 12558  ms: 192
  freq: 0191 0338 0975 0000  vol: 12 12 10  mix: 07 time: 12574  ms: 208
  freq: 0000 0000 0000 0000  vol: 00 00 00  mix: 00 time: 12590  ms: 224
</pre>
<br>
La ventaja de interceptar tanto los SFX's, como los VGM's, es que no necesitamos tener activo <b>play_sound</b>, y por tanto ni emular el Z80 de sonido, ni ninguno los chips de sonido.<br>
Lo he simplificado todo, para mostrar 1 chip del AY-3-8912, dado que en este caso sólo hay un jugador disparando. Se puede ver como llega el comando 0x04, seguido de un 0xFF, es decir, el sonido del disparo (SAMPLE Fire).<br>
El time es el medidor de milisegundos actual, mientras que ms, son los milisegundos desde que comienza el sonido, de manera, que se puede ver, que más o menos cada cambio es entre 16 o 17 milisegundos, con una duración total del mismo de 224 milisegundos.<br>
El volumen de cada canal, va de 0 a 15, y el mix está en hexadecimal, en lógica normal, de manera, que si tenemos:<br><br>

| MIX  | Bin | C B A |
|------|-----|-------|
| 0x00 | 000 | 0 0 0 |
| 0x01 | 001 | 0 0 1 |
| 0x02 | 010 | 0 1 0 |
| 0x03 | 011 | 0 1 1 |
| 0x04 | 100 | 1 0 0 |
| 0x05 | 101 | 1 0 1 |
| 0x06 | 110 | 1 1 0 |
| 0x07 | 111 | 1 1 1 |

<br><br>
Por tanto, si vamos enviando las frecuencias, con los volumenes, el canal de mezcla, todo ello siguiendo el intervalo de milisegundos que está establecido, contra el oscilador en tiempo real, generaremos el sonido de disparo. Otra opción es sustituirlo por un SAMPLE en formato RAW del WAV, pero enviar sólo 15 datos para generar 224 milisegundos de SAMPLE de sonido, es bastante tentador, para ahorrar memoria.<br><br>


https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/disparo.wav?raw=true

<br><br>
Otro ejemplo, es el caso del SFX de la Bomba (0x07):<br>

<pre>
 soundw off:0x00 d:0x07 pend:0x00 QUEUE:0x0A (SAMPLE Bomb) 
 soundw off:0x00 d:0xFF pend:0x00 QUEUE:0x0A
         A    B    C   Noise       A  B  C
  freq: 0853 0379 0196 0533  vol: 11 14 15  mix: 0F  time: 18014  ms: 0
  freq: 0474 0279 0165 0474  vol: 11 14 15  mix: 0F  time: 18030  ms: 16
  freq: 0328 0221 0143 0426  vol: 11 14 15  mix: 0F  time: 18046  ms: 32
  freq: 0251 0183 0126 0388  vol: 11 14 15  mix: 0F  time: 18063  ms: 49
  freq: 0204 0157 0113 0388  vol: 11 14 15  mix: 0F  time: 18078  ms: 64
  freq: 0171 0137 0102 0355  vol: 11 14 15  mix: 0F  time: 18094  ms: 80
  freq: 0147 0121 0093 0328  vol: 11 14 15  mix: 0F  time: 18110  ms: 96
  freq: 0129 0109 0086 0304  vol: 12 14 15  mix: 0F  time: 18126  ms: 112
  freq: 0116 0099 0079 0304  vol: 12 14 15  mix: 0F  time: 18142  ms: 128
  freq: 0104 0090 0074 0284  vol: 12 14 15  mix: 0F  time: 18158  ms: 144
  freq: 0095 0083 0069 0266  vol: 12 14 15  mix: 0F  time: 18174  ms: 160
  freq: 0087 0077 0065 0251  vol: 12 14 15  mix: 0F  time: 18191  ms: 177
  freq: 0081 0072 0061 0251  vol: 12 14 15  mix: 0F  time: 18206  ms: 192
  freq: 0075 0067 0058 0237  vol: 12 14 15  mix: 0F  time: 18222  ms: 208
  freq: 0070 0063 0055 0224  vol: 12 14 15  mix: 0F  time: 18238  ms: 224
  freq: 0066 0060 0052 0213  vol: 13 14 15  mix: 0F  time: 18254  ms: 240
  freq: 0062 0057 0050 0213  vol: 13 14 15  mix: 0F  time: 18270  ms: 256
  freq: 0058 0054 0047 0203  vol: 13 14 15  mix: 0F  time: 18286  ms: 272
  freq: 0055 0051 0045 0194  vol: 13 14 15  mix: 0F  time: 18302  ms: 288
  freq: 0053 0049 0043 0185  vol: 13 14 15  mix: 0F  time: 18318  ms: 304
  freq: 0050 0047 0042 0185  vol: 13 14 15  mix: 0F  time: 18334  ms: 320
  freq: 0048 0045 0040 0177  vol: 13 14 15  mix: 0F  time: 18350  ms: 336
  freq: 0046 0043 0039 0170  vol: 13 14 15  mix: 0F  time: 18366  ms: 352
  freq: 0044 0041 0037 0164  vol: 14 14 15  mix: 0F  time: 18383  ms: 369
  freq: 0042 0040 0036 0164  vol: 14 14 15  mix: 0F  time: 18398  ms: 384
  freq: 0040 0038 0035 0158  vol: 14 14 15  mix: 0F  time: 18414  ms: 400
  freq: 0039 0037 0034 0152  vol: 14 14 15  mix: 0F  time: 18430  ms: 416
  freq: 0038 0036 0033 0147  vol: 14 14 15  mix: 0F  time: 18447  ms: 433
  freq: 0036 0034 0032 0147  vol: 14 14 15  mix: 0F  time: 18462  ms: 448
  freq: 0000 0000 0000 0000  vol: 00 00 00  mix: 00  time: 18478  ms: 464
</pre>
<br>
Se ve como llega el comando 0x07, y luego llegan 30 datos, con el canal A,B,C y ruido, con hasta un total de 464 milisegundos de duración. Esta vez, el mezclador, al tener frecuencia de ruido, tenemos además de los 3 bits de los registros ABC, los 3 bits superiores que nos indican en que canal saldrá la frecuencia de ruido.<br><br>

| MIX  | Bin    | NC | NB | NA |
|------|--------|----|----|----|
|      | 000xxx | 0  | 0  | 0  |
|      | 001xxx | 0  | 0  | 1  |
|      | 010xxx | 0  | 1  | 0  |
|      | 011xxx | 0  | 1  | 1  |
|      | 100xxx | 1  | 0  | 0  |
|      | 101xxx | 1  | 0  | 1  |
|      | 110xxx | 1  | 1  | 0  |
|      | 111xxx | 1  | 1  | 1  |

<br>
https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/bomba.wav?raw=true
<br><br>


Para las melodías, que podemos tener en SAMPLES WAV o crudos, serían:<br>

| CMD  | Tipo | Descripción                  |
|------|------|------------------------------|
| 0x25 | VGM  | Melodía 01.Credit            |
| 0x37 | VGM  | Melodía 02.Start Demo        |
| 0x20 | VGM  | Melodía 03.Game Start        |
| 0x2B | VGM  | Melodía 04.Area 1            |
| 0x2C | VGM  | Melodía 05.Area 2            |
| 0x2D | VGM  | Melodía 06.Area 3            |
| 0x2E | VGM  | Melodía 07.Area 4            |
| 0x2F | VGM  | Melodía 08.Area 5            |
| 0x30 | VGM  | Melodía 09.Bonus Area        |
| 0x31 | VGM  | Melodía 10.Underground       |
| 0x32 | VGM  | Melodía 11.Sanctuary         |
| 0x33 | VGM  | Melodía 12.Underground Boss  |
| 0x34 | VGM  | Melodía 13.Area Boss         |
| 0x35 | VGM  | Melodía 14.Sanctuary Boss    |
| 0x21 | VGM  | Melodía 15.Area Clear 1      |
|      |      | Melodía 16.Area Clear 2      |
| 0x27 | VGM  | Melodía 17.Ranking 1         |
| 0x28 | VGM  | Melodía 18.Ranking 2         |
| 0x29 | VGM  | Melodía 19.Ranking Display 1 |
| 0x2A | VGM  | Melodía 20.Ranking Display 2 |
| 0x36 | VGM  | Melodía 21.Continue          |
| 0x26 | VGM  | Melodía 22.Game Over         |
| 0x23 | VGM  | Melodía 23.Screen Change     |
|      |      | Melodía 24.Extend            |

<br>
Las melodías (VGM), a diferencia de los SFX, se van encolando, de manera, que hasta que no termina, no suena la siguiente. Esto es algo que se nota sobre todo al empezar partida en primer nivel desde 0, que van encolándose:<br><br>

<ul>
 <li>1.- (SAMPLE Melodía 02.Start Demo)</li>
 <li>2.- (SAMPLE Melody 03.Game Start)</li>
 <li>3.- (SAMPLE Melody Area 1)</li>
</ul>

Para convertir un VGM en un SAMPLE, existen varios caminos, pero el más cómodo, usar el <b>vgmplay</b>:<br><br>

> vgmplay -c General.LogSound=1 -w 01 Credit.vgz <br>

Una vez generado el WAV, lo podemos convertir con el <b>goldwave</b> o el <b>audacity</b> a formato RAW de 8 bits con signo. Antes hay que resamplearlo para que ocupe menos, que tal y como se comentó, cada VGM, permite un rango en algunos casos especiales de 2000 Hz, pero en la mayoría, mejor de 8000 Hz para arriba.<br>
Usar signo, es muy útil para en el caso del ESP32 poder sumar en la mezcla sin tener que convertir los signos.<br><br>

Y por último sólo queda mezclar el sample con el resto, que dependerá del resampleo, pero sería tan sencillo como mezclarlo con el mezclador normal.<br>

>auxMix+= (int)(gb_vgz_data[gb_idPlay][gb_cont_vgz])*250; <br>

Esto sería para el caso de SDL. Si estamos con ESP32, apuntaría al buffer de FLASH o bien a un buffer intermedio de SRAM que mediante otro buffer se fuera rellenando a intervalos.<br>





<br><br>
<h1>Lista</h1>
Las melodías VGM, son una especie de MIDI, sobre todo por el resultado de audio final:<br><br>

| ID | Nombre           | Duración    |
|----|-------------------|-------------|
| 01 | Credit            | 0:02        |
| 02 | Start Demo        | 0:06        |
| 03 | Game Start        | 0:08        |
| 04 | Area 1            | 1:13 + 0:59 |
| 05 | Area 2            | 1:05 + 0:59 |
| 06 | Area 3            | 1:30 + 1:27 |
| 07 | Area 4            | 2:02 + 1:44 |
| 08 | Area 5            | 2:04 + 1:47 |
| 09 | Bonus Area        | 1:12 + 1:10 |
| 10 | Underground       | 0:28 + 0:28 |
| 11 | Sanctuary         | 1:43 + 1:17 |
| 12 | Underground Boss  | 0:47 + 0:47 |
| 13 | Area Boss         | 0:25 + 0:09 |
| 14 | Sanctuary Boss    | 0:41 + 0:41 |
| 15 | Area Clear 1      | 0:06        |
| 16 | Area Clear 2      | 0:16        |
| 17 | Ranking 1         | 2:00        |
| 18 | Ranking 2         | 0:44 + 0:23 |
| 19 | Ranking Display 1 | 0:06        |
| 20 | Ranking Display 2 | 0:06        |
| 21 | Continue          | 0:12        |
| 22 | Game Over         | 0:07        |
| 23 | Screen Change     | 0:05        |
| 24 | Extend            | 0:02        |

<br>
En Total: 16:58 + 11:46<br>
Se pueden extraer todas de:<br><br>
<a href='https://vgmrips.net/packs/pack/legendary-wings-arcade'>https://vgmrips.net/packs/pack/legendary-wings-arcade</a><br><br>
Así mismo, las melodías, como tienen calidad MIDI, se podrían resamplear algunas a 2000 Hz, y otras a 4000 Hz, sin perder calidad. Si no tenemos problemas de espacio, como es el caso de un ESP32, es decir, por ejemplo en un PC, se pueden dejar en 8000 Hz o más.<br>
Muchas melodias, son secuenciales, por ejemplo, después de una, viene la otra:<br>
<ul>
 <li>22. Game Over 0:07</li>
 <li>21. Continue 0:12, salvo que consigamos Ranking.</li>
</ul>



<br><br>
<h1>Compactar</h1>
Las melodías en formato RAW o WAV, puede que ocupen demasiado,sobre todo para dispositivos de recursos reducidos como el ESP32. Sin embargo, mucha información sobra, por ejemplo, tenemos segundos de silencio, que al estar sampleados, ocupan demasiado.<br>
Recordemos, que 1 segundo con un sampleo de 8000 Hz, equivale a 8000 bytes.<br>
Un ejemplo, sería el VGM <b>22.Game Over</b>, que tiene un par de milisegundos de silencio al principio y 1 segundo al final.<br><br>
<center><img src='https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/gameoversnd.gif'></center><br>
Esos datos de silencio, se pueden quitar del almacenamiento, y dejar que se encarguen las partes de arriba de controlar, con sólo decirle que tiene 1 segundo de silencio al final.<br><br>

Los datos del archivo de audio, aunque se han dejado con 8 bits, se puede apreciar, que tanto sus valores mínimos y máximos no superan ni el -59, ni el 61:<br><br>

| VGM                  | Min | Max |
|----------------------|-----|-----|
| 01 Credit            | -38 | 25  |
| 02 StartDemo         | -42 | 46  |
| 03 GameStart         | -44 | 46  |
| 04 Area1             | -44 | 46  |
| 05 Area2             | -55 | 46  |
| 06 Area3             | -57 | 49  |
| 07 Area4             | -57 | 55  |
| 08 Area5             | -57 | 58  |
| 09 BonusArea         | -57 | 58  |
| 10 Underground       | -57 | 58  |
| 11 Sanctuary         | -57 | 58  |
| 12 Underground Boss  | -57 | 58  |
| 13 Area Boss         | -57 | 58  |
| 14 Sanctuary Boss    | -57 | 58  |
| 15 Area Clear        | -57 | 58  |
| 16 Area Clear        | -57 | 58  |
| 17 Ranking 1         | -57 | 58  |
| 18 Ranking 2         | -59 | 59  |
| 19 Ranking Display 1 | -59 | 59  |
| 20 Ranking Display 2 | -59 | 59  |
| 21 Continue          | -59 | 59  |
| 22 Game Over         | -59 | 59  |
| 23 Screen Change     | -59 | 61  |
| 24 Extend            | -59 | 61  |

<br>
Por tanto, con una codificación de 7 bits (-63, 63) nos serviría, aunque claro, sólo ahorraríamos 1 bit. Si hacemos una división a la mitad, es decir, un DIV 2, veríamos que con 6 bits (-31, 31) conseguimos mismos resultados, pero, debemos hacer una normalización, de manera que si el valor antes de hacer la división no era 0, y luego si, mejor hacer una división menos agresiva:<br><br>

<pre>
 divAgresiva= 8;
 
 aux= ptr[j];                
 auxAntes= aux;

 aux= aux/divAgresiva;
 if ((aux==0)&&(auxAntes!=0)) 
 {
  aux= (auxAntes/(divAgresiva/2)); 
 }
</pre>

Con 6 bits, ahorramos un 25% de espacio, muy útil para el ESP32. Pero podemos, seguir más agresivamente y aplicando algoritmos de compresión de bajos recursos con el muestreo, valorando la pérdida de calidad.<br>
Todos los algoritmos que apliquemos de reducción, a la hora de reproducir, debemos de hacer el proceso inverso.

 


<br><br>
<h1>Nivel</h1>
De lo que he podido depurar, el nivel de juego se puede detectar por 3 posiciones de memoria:<br><br>

| ADDR   | Descripción                      |
|--------|----------------------------------|
| 0xC06C | Resetear (1 avanza el nivel)     |
| 0xC06D | El nivel                         |
| 0xC06E | Finalizar (0 continuar, 1 morir) |
<br>

El nivel propiamente dicho está en 0xC06D, pero si no se envia el set a 1 de las posiciones 0xC06C y 0 de 0xC06E, no se hará ninguna acción.<br>
La dirección 0xC06D, sólo puede llegar a 5 y una vez superado, pasa a 0.<br>
Saber el nivel, es cómodo para poder saber que melodía debemos poner y para más situaciones.



<br><br>
<h1>Patrones</h1>
Un sistema no sólo alternativo,sino complementario para detección de VGM's, sería el uso no sólo de patrones visuales, sino de estados del emulador, ya que tenemos acceso al MAME directo.<br>
Para el caso de pulsar la tecla 3, que equivale a introducir moneda, ya podemos asociarle el VGM <b>Melodía 01.Credit</b>.<br>
Así mismo, una vez pulsemos la tecla 1, se sabe que se inicia la emulación, por lo que pasariamos a la secuencia:
<ul>
 <li>02.Start Demo</li>
 <li>03.Game Start</li>
 <li>04.Area 1</li>
</ul>
Así mismo, cuando morimos, la pantalla de GAME OVER, es:<br><br>
<center><img src='https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/screengameover.gif'></center>
Detectarla, es muy sencillo, dado que es casi todo negro, pero no haría falta analizar toda la imagen, sólo un par de pixels mínimo que lo diferencie del resto. Además, no hace falta analizarlo siempre, ni siquiera 50 o 60 veces por segundo, con muchísimo menos, ya sirve.<br><br>

Después del GAMEOVER, siempre viene el CONTINUE, así que la forma de detectarlo es más fácil, ya que ya partimos del GAMEOVER.<br>
<center><img src='https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/screencontinue.gif'></center>
Si se ha superado el record 1 o el 2, se reproducirá la melodía del ranking 1 o 2, así como una vez guardado, la melodía del Ranking Display 1 o 2.<br><br>

La relación de pantallas y melodías, principalmente es la siguiente, y sirve para analizar el patrón de pixels diferenciador:<br>
<table>
 <tr>
  <th>02. Start Demo</th>
  <th>03. Game Start</th>
  <th>04. Area 1</th>
  <th>23. Screen Change</th>  
 </tr>
 <tr>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/01startdemo.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/02gamestart.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/03area1.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/05screenchange.gif'></td> 
 </tr>
 <tr>
  <th>23. Screen Change</th>
  <th>12. Underground Boss</th>
  <th>13.	Area Boss</th>
  <th>11.	Sanctuary</th>
 </tr> 
 <tr>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/06screenchange.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/07undergroundboss.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/08areaboss.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/09sanctuary.gif'></td>
 </tr>
 <tr>
  <th>14. Sanctuary Boss</th>
  <th>15.	Area Clear 1</th>
  <th>05.	Area 2</th>
  <th>05.	Area 2</th>
 </tr> 
 <tr>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/10sanctuaryboss.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/11areaclear1.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/12area2.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/13area2.gif'></td>
 </tr>
 <tr>
  <th>13.	Area Boss</th>
  <th>15.	Area Clear 1</th>
  <th>06.	Area 3</th>
  <th>06.	Area 3</th>
 </tr> 
 <tr>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/14areaboss.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/15areaclear1.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/16area3.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/17area3.gif'></td>
 </tr>
 <tr>
  <th>13.	Area Boss</th>
  <th>15.	Area Clear 1</th>
  <th>06.	Area 3</th>
  <th>07. Area 4</th>
 </tr> 
 <tr>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/18areaboss.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/19areaclear1.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/20area3.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/21area4.gif'></td>
 </tr>
 <tr>
  <th>13.	Area Boss</th>
  <th>15.	Area Clear 1</th>
  <th>07. Area 4</th>
  <th>08.	Area 5</th>
 </tr> 
 <tr>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/22areaboss.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/23areaclear1.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/24area4.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/25area5.gif'></td>
 </tr>
 <tr>
  <th>13.	Area Boss</th>
  <th>15.	Area Clear 1</th>
  <th>09. Bonus Area</th>
  <th>18. Ranking 2</th>
 </tr> 
 <tr>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/26areaboss.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/27areaclear1.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/28bonusarea.gif'></td>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/200ranking2.gif'></td>
 </tr>
 <tr>
  <th>20. Ranking Display 2</th>
 </tr>
 <tr>
  <td><img src='https://raw.githubusercontent.com/rpsubc8/ESP32TinyMame/main/preview/201rankingdisplay2.gif'></td>
 </tr> 
</table>



<br><br>
<h1>Trucos</h1>
La información está sacada de:<br><br>
<center></center><a href='https://ryiron.wordpress.com/2019/10/14/legendary-wings-reversing-a-1980s-arcade-game/'>https://ryiron.wordpress.com/2019/10/14/legendary-wings-reversing-a-1980s-arcade-game</a></center><br><br>

| Memoria | Valor | Accion      |
|---------|-------|-------------|
| 0xC11B  | 1     | Invencible  |
| 0xC118  | 0x05  | Potenciador |

Con escribir en dicha posición de memoria desde las capas más altas del emulador, incluso cada segundo, se tendría.<br>
