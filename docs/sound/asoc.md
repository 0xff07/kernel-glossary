---
topics: sound
tags:
    - "sound"
---

# ALSA SoC (ASoC)

Try `aplay -L` and do `trace-cmd` when doing `speaker-test` on that card. For example:

```
$ aplay -L
...
plughw:CARD=sofhdadsp,DEV=0
    sof-hda-dsp, 
    Direct hardware device without any conversions
...
```

Then:

```
$ sudo trace-cmd -p function_graph -F \
    speaker-test -t wav -c 2 -Dplughw:sofhdadsp
```

And finally:

```
$ trace-cmd report | less
```

## References

### Videos

- [Generic ALSA SoC Sound Card History (Simple/Graph) - Kuninori Morimoto, Renesas Electronics](https://youtu.be/usRwGpJJkpk)
- [Insight of an Audio Driver Based on ALSA - Chandrasekar Ramakrishnan, Samsung](https://youtu.be/6R8Wjytv-eQ)
- [ASoC: Supporting Audio on an Embedded Board - Alexandre Belloni, Bootlin](https://youtu.be/572T8RDHK_A)
- [FOSDEM 2024 - From phone hardware to mobile Linux (11:20)](https://youtu.be/VMOxarhhYuM?t=680)

### Links

- [ALSA SoC Layer](https://www.kernel.org/doc/html/latest/sound/soc/index.html)
- [ALSA overview](https://wiki.st.com/stm32mpu/wiki/ALSA_overview)
- [Soundcard configuration](https://wiki.st.com/stm32mpu/wiki/Soundcard_configuration)
- [Audio and Embedded Linux](https://www.marcusfolkesson.se/blog/audio-and-embedded-linux/)
