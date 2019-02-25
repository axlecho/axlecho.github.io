---
title: audio 混音
date: 2019-01-19 22:09:41
tags: android_framework
---

android的混音是通过AudioMixer来实现的，最近遇到了一个混音的问题，该是好好看看音频的基本知识了。

音频的基本知识
---

很早之前就知道音频存储是通过采样来实现的，就是所谓的A/D（Analog-to-Digital Converter与D/A（Digital Analog Converter）
音轨有很多属性如

*   采样率(sampleRate)
*   编码格式(format)
*   通道(channelCount)
*   总帧数(frameCount) ?
*   音量(volume) ?
*   播放速率(playbackRate) ?

前三个比较重要，具体去了解一下

#### 采样率
看到知乎上有一个回答，感觉挺形象的--[什么是音频的采样率？采样率和音质有没有关系？](http://www.zhihu.com/question/20351692?utm_campaign=rss&utm_medium=rss&utm_source=rss&utm_content=title)

一般的采样率固定在`44100HZ`(- -就是一秒记44100次)，理由是因为人耳听觉范围在20HZ~20KHZ，这样记录能还原最高22.05KHZ的声音。
还有一个采样率`48000HZ`也比较常见

#### 编码格式
指每次采样所用的bit数，比如8bit，16bit  
从命名来看，android好像用来8bit，16bit跟32bit

*   AUDIO_FORMAT_PCM_16_BIT
*   AUDIO_FORMAT_PCM_8_BIT
*   AUDIO_FORMAT_PCM_32_BIT
*   AUDIO_FORMAT_PCM_8_24_BIT
*   AUDIO_FORMAT_PCM_FLOAT
*   AUDIO_FORMAT_PCM_16_BIT_OFFLOAD ?
*   AUDIO_FORMAT_PCM_16_BIT_OFFLOAD ?

#### 通道
就是两个耳机有不同的声音？？
一般有单通道(mono) 双通道(stereo)

AudioMixer混音过程
---
参考博客园上的一篇博客[[Android] 混音器AudioMixer](http://www.tuicool.com/articles/2mqUjav)

#### AUDIOMIXER的创建
mNormalFrameCount为输入buffer大小？？mSampleRate为输出采样率
```cpp
AudioFlinger::MixerThread::MixerThread(...)
    : ...{
    ...
    mAudioMixer = new AudioMixer(mNormalFrameCount, mSampleRate);
    ...
}
```

#### 配置参数
在MixerThread的prepareTracks_l会对AudioMixer的参数进行配置，如音量，输入源，混音参数等。
```cpp
AudioFlinger::PlaybackThread::mixer_state AudioFlinger::MixerThread::prepareTracks_l(...){
    ...
    // 设置输入源  name是引索集合，track是mActiveTracks？？
    mAudioMixer->setBufferProvider(name, track);

    // 启用该启用的音轨并更新状态
    mAudioMixer->enable(name);

    // 设置左右音量
    mAudioMixer->setParameter(name, param, AudioMixer::VOLUME0, &vlf);
    mAudioMixer->setParameter(name, param, AudioMixer::VOLUME1, &vrf);
    mAudioMixer->setParameter(name, param, AudioMixer::AUXLEVEL, &vaf);

    // 设置混音格式
    mAudioMixer->setParameter(name,AudioMixer::TRACK,AudioMixer::FORMAT, (void *)track->format());

    // 设置通道数
    mAudioMixer->setParameter(name,AudioMixer::TRACK,AudioMixer::CHANNEL_MASK, (void *)(uintptr_t)track->channelMask());

    ...
    // 设置采样率
    mAudioMixer->setParameter(name,AudioMixer::RESAMPLE,AudioMixer::SAMPLE_RATE,(void *)(uintptr_t)reqSampleRate);

    // 设置播放速率  
    mAudioMixer->setParameter(name,AudioMixer::TIMESTRETCH,AudioMixer::PLAYBACK_RATE,&playbackRate);

    ... // mark一下
    if (mMixerBufferEnabled && (track->mainBuffer() == mSinkBuffer || track->mainBuffer() == mMixerBuffer)) {
         mAudioMixer->setParameter(name,AudioMixer::TRACK,AudioMixer::MIXER_FORMAT, (void *)mMixerBufferFormat);
    }

    // 设置输出
    mAudioMixer->setParameter(name,AudioMixer::TRACK,AudioMixer::MAIN_BUFFER, (void *)mMixerBuffer);
    ...
}
```

#### 混音函数选择
先计算track的flags，然后通过flags来判断要混音的函数
```cpp
void AudioMixer::process__validate(state_t* state, int64_t pts)
{
    ...(计算需要invalidate的track)

    // 主要通过下列几个参数去判断
    bool all16BitsStereoNoResample = true;
    bool resampling = false;
    bool volumeRamp = false;
    uint32_t en = state->enabledTracks;

    while (en) {    //对所有需要进行混音的track
        const int i = 31 - __builtin_clz(en); //取出最高位为1的bit
        en &= ~(1<<i);  //把这一位置为0

        countActiveTracks++;
        track_t& t = state->tracks[i];  //取出来track

        // 计算flags
        uint32_t n = 0;
        n |= NEEDS_CHANNEL_1 + t.channelCount - 1;    //至少有一个channel需要混音
        n |= NEEDS_FORMAT_16;          //必须为16bit PCM
        n |= t.doesResample() ? NEEDS_RESAMPLE_ENABLED : NEEDS_RESAMPLE_DISABLED; //是否需要重采样
        if (t.auxLevel != 0 && t.auxBuffer != NULL) {
            n |= NEEDS_AUX_ENABLED;
        }

        if (t.volumeInc[0]|t.volumeInc[1]) {
            volumeRamp = true;
        } else if (!t.doesResample() && t.volumeRL == 0) {
            n |= NEEDS_MUTE_ENABLED;
        }
        t.needs = n;    //更新track flag

        ...(通过flags选择混音函数)

        //这里调用一次进行混音，后续会在MixerThread的threadLoop_mix内调用
        state->hook(state, pts);  

        ...
    }
}
```
#### 混音
就是像搓面把几根面搓成一团

在android里使用process_xxx函数来实现，几个方法大同小异(不是我说的),举个process__genericResampling当例子。
```cpp
// generic code with resampling
void AudioMixer::process__genericResampling(state_t* state, int64_t pts)
{
    int32_t* const outTemp = state->outputTemp;
    size_t numFrames = state->frameCount;

    uint32_t e0 = state->enabledTracks;
    while (e0) {
        ...(选出有相同mainBuffer的track集合e1)
        int32_t *out = t1.mainBuffer;
        memset(outTemp, 0, size);

        // 搓面
        while (e1) {
            const int i = 31 - __builtin_clz(e1);
            e1 &= ~(1<<i);
            track_t& t = state->tracks[i];
            int32_t *aux = NULL;
            if (CC_UNLIKELY(t.needs & NEEDS_AUX)) {
                aux = t.auxBuffer;
            }

            // this is a little goofy, on the resampling case we don't
            // acquire/release the buffers because it's done by
            // the resampler.
            if (t.needs & NEEDS_RESAMPLE) {
                t.resampler->setPTS(pts);
                t.hook(&t, outTemp, numFrames, state->resampleTemp, aux);
            } else {...}
        }
    }
}
```

真正的采样在这(最终的采样在resample)
```cpp
void AudioMixer::track__genericResample(track_t* t, int32_t* out, size_t outFrameCount,
        int32_t* temp, int32_t* aux)
{
    ALOGVV("track__genericResample\n");
    t->resampler->setSampleRate(t->sampleRate);

    // ramp gain - resample to temp buffer and scale/mix in 2nd step
    if (aux != NULL) {
        ...(设置ramp音量)
    } else {
        if (CC_UNLIKELY(t->volumeInc[0]|t->volumeInc[1])) {
            t->resampler->setVolume(UNITY_GAIN_FLOAT, UNITY_GAIN_FLOAT);
            memset(temp, 0, outFrameCount * MAX_NUM_CHANNELS * sizeof(int32_t));
            t->resampler->resample(temp, outFrameCount, t->bufferProvider);
            volumeRampStereo(t, out, outFrameCount, temp, aux);
        }

        // constant gain
        else {
            t->resampler->setVolume(t->mVolume[0], t->mVolume[1]);
            t->resampler->resample(out, outFrameCount, t->bufferProvider);
        }
    }
}
```