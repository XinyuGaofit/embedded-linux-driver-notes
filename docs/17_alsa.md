# 17. ALSA / ASoC

ALSA 是 Linux 音频框架。

嵌入式 SoC 上常见 ASoC：

```text
Userspace
  ↓
ALSA PCM / mixer
  ↓
ASoC
  ├─ CPU DAI
  ├─ Codec DAI
  ├─ PCM / DMA
  └─ Machine / Card
  ↓
I2S/SAI + DMA + Codec
```

## 用户空间

常见：

```text
/dev/snd/pcmC0D0p
/dev/snd/pcmC0D0c
/dev/snd/controlC0
```

工具：

```bash
aplay
arecord
amixer
```

## PCM

音频采样流：

```text
playback
capture
```

常见参数：

```text
sample rate
channels
sample format
period size
buffer size
```

## DAI

Digital Audio Interface。

例如：

```text
I2S
Left Justified
DSP_A/B
```

## Codec

负责：

```text
ADC
DAC
Mixer
Mic Bias
Headphone output
```

## ASoC 分层

```text
Machine Driver
    ↓
连接 CPU DAI 和 Codec DAI

CPU DAI Driver
    ↓
SoC I2S/SAI controller

Codec Driver
    ↓
外部 audio codec

PCM/DMA
    ↓
数据搬运
```

## Linux 4.9 常见对象

4.9 与新内核 ASoC API 差异较大，优先参考当前 BSP 的 `sound/soc/`。

常见对象：

```text
snd_soc_codec_driver
snd_soc_dai_driver
snd_soc_card
snd_soc_dai_link
snd_pcm_substream
snd_pcm_runtime
```

## PCM 回调概念

常见流程：

```text
open
hw_params
prepare
trigger
pointer
close
```

`trigger` 控制：

```text
START
STOP
PAUSE
RESUME
```

底层通常进一步控制 DMA 和 I2S/SAI。

## 和用户态 ALSA 的对应

用户态：

```text
snd_pcm_open
snd_pcm_hw_params
snd_pcm_readi
```

内核：

```text
ALSA PCM Core
   ↓
ASoC PCM
   ↓
CPU DAI / DMA / Codec
```

## 学习顺序

```text
PCM 数据从哪里来
      ↓
DMA 如何搬
      ↓
CPU DAI 如何产生 I2S
      ↓
Codec 如何 ADC/DAC
      ↓
Machine driver 如何连接
```

之后再看 DAPM、mixer control、clock routing。
