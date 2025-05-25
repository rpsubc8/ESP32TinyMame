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

 | Registro | Function                       | Name        | Range        |
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

| CMD  | Type | Description                    |
|------|------|--------------------------------|
| 0x01 | SFX  | Dying by enemy explosion       |
| 0x04 | SFX  | Fire shot                      |
| 0x07 | SFX  | Pump                           |
| 0x0A | SFX  | Enemy fire explosion           |

<br>

SFX effects, in the end, although we could have them sampled, are translated into port write calls on the AY-3-8912 PSGs.<br>
If we debug and leave trace, specifically in <b>_AYUpdateChip</b> of <b>psg.cpp</b>, we can either leave all writes to AY-3-8912 registers, or better, just with the calculations of frequency (A,B,C and noise), volume (A,B,C) and mix channels:<br>

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
The advantage of intercepting both SFX's and VGM's is that we don't need to have <b>play_sound</b> active, and therefore neither emulate the Z80 sound, nor any of the sound chips.<br>
I have simplified everything to show 1 chip of the AY-3-8912, since in this case there is only one player firing. You can see how the command 0x04 arrives, followed by 0xFF, i.e. the sound of the firing (SAMPLE Fire).<br>
The time is the actual millisecond meter, while ms is the milliseconds from the start of the sound, so you can see that roughly every change is between 16 or 17 milliseconds, with a total duration of 224 milliseconds.<br>
The volume of each channel goes from 0 to 15, and the mix is in hexadecimal, in normal logic, so if we have:<br><br>

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
So, if we send the frequencies, with the volumes, the mix channel, all following the millisecond interval that is set, against the oscillator in real time, we will generate the trigger sound. Another option is to replace it with a SAMPLE in RAW format of the WAV, but sending only 15 data to generate 224 milliseconds of SAMPLE sound, is quite tempting, to save memory.<br><br>


https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/disparo.wav?raw=true

<br><br>
Another example is the case of the Bomb SFX (0x07):<br>

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
We can see how the 0x07 command arrives, and then 30 pieces of data arrive, with channel A, B, C and noise, with a total of 464 milliseconds of duration. This time, the mixer, having noise frequency, we have in addition to the 3 bits of the ABC registers, the 3 upper bits that tell us in which channel the noise frequency will be output.<br><br>

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

| CMD  | Type | Description                  |
|------|------|------------------------------|
| 0x25 | VGM  | Melody 01.Credit             |
| 0x37 | VGM  | Melody 02.Start Demo         |
| 0x20 | VGM  | Melody 03.Game Start         |
| 0x2B | VGM  | Melody 04.Area 1             |
| 0x2C | VGM  | Melody 05.Area 2             |
| 0x2D | VGM  | Melody 06.Area 3             |
| 0x2E | VGM  | Melody 07.Area 4             |
| 0x2F | VGM  | Melody 08.Area 5             |
| 0x30 | VGM  | Melody 09.Bonus Area         |
| 0x31 | VGM  | Melody 10.Underground        |
| 0x32 | VGM  | Melody 11.Sanctuary          |
| 0x33 | VGM  | Melody 12.Underground Boss   |
| 0x34 | VGM  | Melody 13.Area Boss          |
| 0x35 | VGM  | Melody 14.Sanctuary Boss     |
| 0x21 | VGM  | Melody 15.Area Clear 1       |
|      |      | Melody 16.Area Clear 2       |
| 0x27 | VGM  | Melody 17.Ranking 1          |
| 0x28 | VGM  | Melody 18.Ranking 2          |
| 0x29 | VGM  | Melody 19.Ranking Display 1  |
| 0x2A | VGM  | Melody 20.Ranking Display 2  |
| 0x36 | VGM  | Melody 21.Continue           |
| 0x26 | VGM  | Melody 22.Game Over          |
| 0x23 | VGM  | Melody 23.Screen Change      |
|      |      | Melody 24.Extend             |

<br>
The melodies (VGM), unlike the SFX, are glued, so that the next one doesn't play until the end of the game. This is something that is especially noticeable when starting a game on the first level from 0, which are glued:<br><br>

<ul>
 <li>1.- (SAMPLE Melodía 02.Start Demo)</li>
 <li>2.- (SAMPLE Melody 03.Game Start)</li>
 <li>3.- (SAMPLE Melody Area 1)</li>
</ul>

To convert a VGM into a SAMPLE, there are several ways, but the most comfortable way is to use the <b>vgmplay</b>.:<br><br>

> vgmplay -c General.LogSound=1 -w 01 Credit.vgz <br>

Once the WAV is generated, we can convert it with <b>goldwave</b> or <b>audacity</b> to 8-bit RAW format with sign. Before we have to resample it so that it occupies less, that as it was commented, each VGM, allows a range in some special cases of 2000 Hz, but in the majority, better of 8000 Hz upwards.<br>
Using sign, it is very useful in the case of ESP32 to be able to add in the mixture without having to convert the signs.<br><br>

And finally, the only thing left to do is to mix the sample with the rest, which will depend on the resampling, but it would be as simple as mixing it with the normal mixer.<br>

>auxMix+= (int)(gb_vgz_data[gb_idPlay][gb_cont_vgz])*250; <br>

This would be in the case of SDL. If we are with ESP32, it would point to the FLASH buffer or to an intermediate SRAM buffer that would be filled at intervals by means of another buffer.<br>





<br><br>
<h1>List</h1>
VGM tunes are a kind of MIDI, especially in terms of the final audio output:<br><br>

| ID | Name              | Duration    |
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
In Total: 16:58 + 11:46<br>
They can all be extracted from:<br><br>
<a href='https://vgmrips.net/packs/pack/legendary-wings-arcade'>https://vgmrips.net/packs/pack/legendary-wings-arcade</a><br><br>
Also, the melodies, as they have MIDI quality, could be resampled at 2000 Hz, and others at 4000 Hz, without losing quality. If we don't have space problems, as is the case with an ESP32, i.e. for example on a PC, they can be left at 8000 Hz or more.<br>
Many melodies are sequential, i.e. after one comes another:<br>
<ul>
 <li>22. Game Over 0:07</li>
 <li>21. Continue 0:12, unless we get Ranking.</li>
</ul>



<br><br>
<h1>Compact</h1>
The tunes in RAW or WAV format may take up too much space, especially for low-resource devices like the ESP32. However, a lot of information is left over, for example, we have seconds of silence, which, being sampled, take up too much space.<br>
Recall that 1 second at 8000 Hz sampling is equivalent to 8000 bytes.<br>
An example would be VGM <b>22.Game Over</b>, which has a couple of milliseconds of silence at the beginning and 1 second at the end.<br><br>
<center><img src='https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/gameoversnd.gif'></center><br>
That silence data can be removed from storage, and let the upstream parts take care of controlling it, just by telling it that it has 1 second of silence at the end.<br><br>

There is also repetition of blocks, being scores, in particular, we can see it in VGM 10. Underground:<br>
<center><img src='https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/compress10Underground.gif'></center>
There are 28 seconds, which is repeated for another 28 seconds, ending with a final repetition of the block of only 7 seconds with a fade out effect.
So we go from 00:01:04 of data to just 00:00:28, which is repeated 3 times, i.e. from 512000 bytes (500 KB) to 224000 bytes (218 KB).<br>
<br>
We can continue to reduce further, given that within block 1, we have a 7.5 second repetition:<br><br>
<center><img src='https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/compress10Underground02.gif'></center>
<br>
The data of the audio file, although it has been left with 8 bits, it can be seen that both its minimum and maximum values do not exceed neither -59 nor 61:<br><br>

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
Therefore, a 7-bit encoding (-63, 63) would work, although of course, we would only save 1 bit. If we make a division by half, that is, a DIV 2, we would see that with 6 bits (-31, 31) we get the same results, but we must make a normalisation, so that if the value before making the division was not 0, and then it was, it is better to make a less aggressive division:<br><br>

<pre>
 //Codificacion agresiva simple 4 bits (-7,7)
 divAgresiva= 8;
 
 aux= ptr[j];                
 auxAntes= aux;

 aux= aux/divAgresiva;
 if ((aux==0)&&(auxAntes!=0)) 
 {
  aux= (auxAntes/(divAgresiva/2)); 
 }
</pre>

With 6 bits, we save 25% space, very useful for ESP32. But we can, however, go more aggressively and apply low-resource compression algorithms with sampling, valuing the loss of quality.<br>
All the algorithms that we apply for reduction, at the time of reproduction, we must do the inverse process.

 


<br><br>
<h1>Level</h1>
From what I have been able to debug, the level of play can be detected by 3 memory locations:<br><br>

| ADDR   | Descripción                      |
|--------|----------------------------------|
| 0xC06C | Reset (1 advances the level)     |
| 0xC06D | The level                        |
| 0xC06E | End (0 continue, 1 die)          |
<br>

The level itself is at 0xC06D, but if the set is not sent to 1 of positions 0xC06C and 0 of 0xC06E, no action will be taken.<br>
Address 0xC06D, can only go to 5 and once exceeded, it goes to 0.<br>
Knowing the level is useful to know what melody to play and for more situations.



<br><br>
<h1>Patterns</h1>
An alternative, but complementary system for VGM detection would be the use not only of visual patterns, but also of emulator states, since we have direct access to MAME.<br>
In the case of pressing key 3, which is equivalent to entering currency, we can associate the VGM <b>Melody 01.Credit</b> to it.<br>
Likewise, once we press the 1 key, we know that the emulation starts, so we move on to the sequence:
<ul>
 <li>02.Start Demo</li>
 <li>03.Game Start</li>
 <li>04.Area 1</li>
</ul>
Likewise, when we die, the GAME OVER screen, is:<br><br>
<center><img src='https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/screengameover.gif'></center>
Detecting it is very simple, given that it is almost all black, but it would not be necessary to analyse the whole image, just a couple of pixels that differentiate it from the rest. Moreover, it is not necessary to analyse it all the time, not even 50 or 60 times per second, with much less, it is enough.<br><br>

After the GAMEOVER, there is always the CONTINUE, so the way to detect it is easier, since we already start from the GAMEOVER.<br>
<center><img src='https://github.com/rpsubc8/ESP32TinyMame/blob/main/preview/screencontinue.gif'></center>
If record 1 or 2 has been beaten, the melody of ranking 1 or 2 will be played, as well as the melody of Ranking Display 1 or 2 once saved.<br><br>

The relationship of screens and tunes is mainly as follows, and serves to analyse the differentiating pixel pattern:<br>
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
<h1>Cheats</h1>
The information is taken from:<br><br>
<center></center><a href='https://ryiron.wordpress.com/2019/10/14/legendary-wings-reversing-a-1980s-arcade-game/'>https://ryiron.wordpress.com/2019/10/14/legendary-wings-reversing-a-1980s-arcade-game</a></center><br><br>

| Memory  | Value | Action      |
|---------|-------|-------------|
| 0xC11B  | 1     | Invincible  |
| 0xC118  | 0x05  | Enhancer    |

Writing to this memory location from the higher layers of the emulator, even every second, would be enough.<br>

