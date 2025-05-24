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
