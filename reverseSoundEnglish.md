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

As we are working with 16 signed bits, the maximum is positive, while the minimum is negative. Since we are working with low values, no matter how much we add, we are not going to exceed the value of -32768 or 32767, so we do not need to clip.<br><br>

In the case of noise, it comes from register 6 of AY-3-8912, i.e. its noise channel (AY_NOISEPER):<br>
<pre>
 unsigned int noise= PSG->Regs[AY_NOISEPER];
 noise= noise ? AYClockFreq / AYSoundRate * 4 / noise : 0;
 //noise= noise ? 1500000 / (16*noise) : 0;
 noise=noise>>5;
</pre><br>
For the oscillator routine, although the <b>rand</b> function can be used, it is more optimised to make use of tricks.<br>
We know the frequency, but we must apply random values for the positive part, and the same for the negative part, respecting the crossover by 0, as well as the sampling frequency:<br><br>
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
<h1>ESP32 Sound</h1>
In the ESP32 we will make use of the DAC (GPIO 25), solving the I2S problems.<br>
The output of the DAC is always positive (0 to 255) and although with 32000 Hz for the mixer, we have enough, we will deal with 44100 Hz.<br>
The system is similar to the use of SDL oscillators, i.e. it is generated in real time (0 lag), changing only frequencies, but in addition, we do not make use of any buffer:<br><br>

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

The system is similar to SDL, using an intermediate latch register, so that concurrently every millisecond the status of this latch is checked, which is updated every time an AY-3-8912 register is written to.<br>
If the CPU emulation routine of a full frame is very fast, i.e. below 8 milliseconds, we may not need to use a timer with the latch every 1 millisecond.<br>
The use of the routine with real time timer for the oscillators, when using the DAC without I2S with DMA, consumes a little more CPU, but in exchange we solve the problem with the different Espressif frameworks (without solution) of using the internal DAC. However, if instead of using this system, we use a R2R resistor ladder with GPIO output or I2C communication with a chip (Atmega328) or external DAC, it would also be solved, and it would even be better, since it is faster than the internal DAC of the ESP32.<br>


<br><br>
<h1>Emulation</h1>
In <b>msdos.cpp</b> you can find the variable <b>play_sound</b>, so if it is set to 0, the second Z80 CPU will stop processing, and therefore, the YM2203 will stop emulating both the FM part and the PSG part of the AY-3-8912, that is, it will stop emulating the sound at 100%.<br>
If, on the other hand, <b>play_sound</b> is set to 1, the sound Z80 will be emulated, and therefore the PSG calls of the AY-3-8912 will be picked up. In this case, we can emulate the AY-3-8912 or not, since we will see the relationship between the SFX's and the VGM's.
Not only can we skip the PSG AY-3-8912 emulation, but also the FM YM2203 part, and even the emulation of the second Z80 CPU, as we have access to the sound commands sent by the main Z80 CPU to the sound CPU, via memory location <b>0xF80C</b> of the first CPU's context, which can be seen in the <b>lwingsdriver.cpp</b> driver.<br><br>

<pre>
 static struct MemoryWriteAddress writemem[] = {
 {
   { 0xc000, 0xdeff, MWA_RAM },
   ...
   { 0xf80c, 0xf80c, sound_command_w },
   ...
 }
</pre><br>

All this can be managed and traced from MAME code in <b>genericsndhrdw.cpp</b>, where the <b>sound_command_w</b> function call is located:<br>

>	 void sound_command_w (int offset,int data) <br>
>	 { <br>

Always 2 commands are sent, which is the VGM identifier (SAMPLE), followed by another command with the value 0xFF.<br>
Some of the commands for SFX effects, would be:<br>

| CMD  | Type | Descripción                    |
|------|------|--------------------------------|
| 0x01 | SFX  | Dying by enemy explosion       |
| 0x04 | SFX  | Fire shot                      |
| 0x07 | SFX  | Pump                           |
| 0x0A | SFX  | Enemy fire explosion           |

<br>
