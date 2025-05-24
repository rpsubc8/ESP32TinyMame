<h1>Hardware Sound</h1>
The Capcom Section Z machine features:
<ul>
 <li>Z80 6 Mhz CPU for the game</li>
 <li>
 Z80 4 Mhz CPU for sound
  <ul>
   <li>YM2203 at 1.5 Mhz (mono)</li>
   <li>YM2203 at 1.5 Mhz (mono)</li>
  </ul>
 </li> 
</ul>

That is, you need 2 Z80 CPUs, one for the game, the other for the sound:

>	for (activecpu = 0;activecpu < totalcpu;activecpu++) <br>
>	{ <br>
>		int cycles; <br>
<br><br>


<h1>YM2203</h1>
The YM2203 is an OPN chip, which on the PSG side is similar to an AY-3-8912 with 3 channels, while on the OPN side it has 3 FM channels with oscillators.<br>
Effects, such as gunshots, bombs and explosions, go to the PSG part, i.e. the 3 channels of a basic AY-3-8912 emulation.<br>
Player 1's effects go to one of the 2 AY-3-8912, while Player 2's effects go to the other. When explosions occur, both are used.<br>
We therefore have (3+3)+(3+3)= 12 channels. The noise channel, if we follow the strict emulation of an AY-3-8912, will come out of 1 of the 3 channels, but if we have enough CPU host and we want to give it more quality, we can add it to the mix, so that we would have 14 channels.<br>
The YM2203 MAME handles it as an AY-3-8912, as long as the write to registers is less than 16, by means of the function <b>AYWriteReg</b> in the <b>psg.cpp</b>. As soon as it is a higher register, the FM part (osd_ym2203_write) would be handled.
<br>
The PSG part of AY-3-8912, consisting of:<br><br>

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

I have measured by statistics, and from what I have looked at, the game does not make use of the envelope, so we can skip its recreation.

<br><br>
<h1>Sound x86 SDL</h1>
On PCs, the sound card always makes use of an internal DMA buffer that cannot be modified, as it would block the rest of the hardware. What is allowed is to use another buffer and a periodic call that fills our auxiliary buffer, which is then sent to the internal buffer of the sound card.<br>
This buffer will depend on the sampling rate, as well as on the number of times it is called to fill, so that the dreaded problem of sound failures can occur, due to mismatching or mismatching times, or even filling, due to under- or over-filling.<br>
To solve this, which is the problem of most emulators, I have decided to go to the most practical, that is, the use of oscillators, so that every time <b>SDL_audio_callback</b> is automatically invoked, we don't have to worry about padding or time, since we pass it the parameters to be generated in the buffer.<br>
If you want a 6-channel mixer, i.e. 2 AY-3-8912 chips, something similar works:<br><br>

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

In <b>Legendary Wings</b>, no use is made of the sound duty cycle, as could be the case with the NES APU, i.e. the AY-3-8912 sound wave, besides being square, has the same duration of the positive part as the negative part, which makes the calculations easier.<br>
If, for example, we want to:<br><br>

> Frequency 1000 Hz <br>
>  44100 Hz / 1000 Hz = 44 samples <br>
>  44 / 2 = 22 positive and 22 negative samples. <br>
<br>
The positive and negative switching is handled internally by the <b>gb_flipflop_ch</b>. So we only have to control the loop with the number of channels, which in this case is 6, both the flipflop switching and the mixing.<br>
The <b>gbVolMixer_now</b> controls the mixer for each channel, so if it is set to 0, that channel is muted, i.e. no mixing, no processing.<br>
The <b>gbVol_channel_now</b> controls the volume of each channel, so it is somewhat similar to the gbVolMixer_now.<br><br>
En esta función de relleno de buffer no se calcula cuantas muestras son positivas ni negativas dada una fecuencia, puesto que ya se le pasa ese cálculo. To know this, it has to be previously calculated, in the <b>_AYUpdateChip</b> from <b>psg.cpp</b>:<br><br>
<pre>
 unsigned int a= (PSG->Regs[AY_AFINE]+((unsigned int)(PSG->Regs[AY_ACOARSE]&0xF)<<8));
 //_AYUpdateChip clk:1500000000 rate:43920
 a = a ? AYClockFreq / AYSoundRate * 4 / a : 0;
 a=a>>2; //Para que suene más grave.
</pre><br>
This example is to have the frequency of channel A, specifically 1 of the 2 AY-3-8912. But we need to convert that frequency into the data for our oscillator:<br><br>

> gb_max_cont_pos_ch[0]= (a!=0)? SAMPLE_RATE/a/2 : 0;  //44100/a/2 <br>
> gb_max_cont_neg_ch[0]= gb_max_cont_pos_ch[0]; <br>
<br>

For channel B and C, it is similar.<br>

For mixing in SDL oscillators, it is as simple as doing a simple addition, taking into account:<br><br>

> flipflop 1 (positive part wave) - Maximum value. <br>
> flipflop 0 (negative part wave) - Minimum value. <br>
<br>

The mixer of the AY-3-8912, is the registry <b>AY_ENABLE</b>, and controls the 3 channels (0 active, 1 mute) in negated logic:<br>
<pre>
 A - AY_ENABLE & 0x01 - gbVolMixer_now[0]= ((~AY_ENABLE) & 0x01)
 B - AY_ENABLE & 0x02 - gbVolMixer_now[1]= ((~AY_ENABLE) & 0x02)
 C - AY_ENABLE & 0x04 - gbVolMixer_now[2]= ((~AY_ENABLE) & 0x04)
</pre>
