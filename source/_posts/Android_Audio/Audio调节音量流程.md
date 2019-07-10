---
title: Audio调节音量流程
date: 2019-01-19 22:17:55
categories: Android_Audio
tags: 
    - framework
    - audio
---

Audio音量调节是一级一级调节，而且分不同的流类型，如响铃，通话，多媒体等。不同的设备(蓝牙设备)的设置方法有所区别。

sdk的api，设置相应流的音量。不同的流index的范围不一样
```java
//--->frameworks/base/media/java/android/media/AudioManager.java
public void setStreamVolume(int streamType, int index, int flags) {
    IAudioService service = getService();
    try {
        service.setStreamVolume(streamType, index, flags, 
            getContext().getOpPackageName());
    } catch (RemoteException e) {
        Log.e(TAG, "Dead object in setStreamVolume", e);
    }
}
```

java层Service实现,volume的调节的实现是用state模式来实现，可能需要原子性或不同的模式下调节音量的操作不同。
```java
//--->frameworks/base/services/core/java/com/android/server/audio/AudioService.java
private void setStreamVolume(int streamType, int index, int flags, 
    String callingPackage,String caller, int uid) {

    // ...(检查参数)
    // ...(转换参数)

    // 获取设备
    final int device = getDeviceForStream(streamType);
    // ...(特殊处理a2dp)
    // ...(检查uid，实体按键调节音量需要判断当前用户？)

    synchronized (mSafeMediaVolumeState) {
        mPendingVolumeCommand = null;
        oldIndex = streamState.getIndex(device);
        index = rescaleIndex(index * 10, streamType, streamTypeAlias);

        // ...(特殊处理a2dp)
        // ...(特殊处理HDMI)
        // ...(设置一些标志位，如标记一些不可调节音量的设备)

        //检查当前是否可设置音量
        if (!checkSafeMediaVolume(streamTypeAlias, index, device)) {
            // 不可以则生成PendingCommand，等待合适的时机
            mVolumeController.postDisplaySafeVolumeWarning(flags);
            mPendingVolumeCommand = new StreamVolumeCommand(
                streamType, index, flags, device);
        } else {
            // 设置音量
            onSetStreamVolume(streamType, index, flags, device, 
                caller);
            index = mStreamStates[streamType].getIndex(device);
        }
    }

    // 发送更新音量信息
    sendVolumeUpdate(streamType, oldIndex, index, flags);
}

private void onSetStreamVolume(int streamType, int index, int flags, 
    int device,String caller) {
    final int stream = mStreamVolumeAlias[streamType];
    // 设置音量
    setStreamVolumeInt(stream, index, device, false, caller);

    // ...(判断音量是否为0，调节模式(静音或响铃))
    mStreamStates[stream].mute(index == 0);
}

private void setStreamVolumeInt(int streamType,int index,int device,
    boolean force,String caller) {
    VolumeStreamState streamState = mStreamStates[streamType];
    if (streamState.setIndex(index, device, caller) || force) {
        // Post message to set system volume (it in turn will post a message
        // to persist).
        sendMsg(mAudioHandler,MSG_SET_DEVICE_VOLUME,SENDMSG_QUEUE,device,
            0,streamState,0);
    }
}

  @Override
public void handleMessage(Message msg) {
    // ...
    switch (msg.what) {
        case MSG_SET_DEVICE_VOLUME:
            setDeviceVolume((VolumeStreamState) msg.obj, msg.arg1);
            break;
        // ...
    }
    // ...
}

private void setDeviceVolume(VolumeStreamState streamState, int device) {

    synchronized (VolumeStreamState.class) {
        // 设置音量
        streamState.applyDeviceVolume_syncVSS(device);

        // ...(Apply change to all streams using this one as alias)
    }

    // Post a persist volume msg
    sendMsg(mAudioHandler,MSG_PERSIST_VOLUME,SENDMSG_QUEUE,device,0,
        streamState,PERSIST_DELAY);
}

public void applyDeviceVolume_syncVSS(int device) {
    int index;
    if (mIsMuted) {
        index = 0;
    } else if (((device & AudioSystem.DEVICE_OUT_ALL_A2DP) != 0 
        && mAvrcpAbsVolSupported) || ((device & mFullVolumeDevices) != 0)) {
        index = (mIndexMax + 5)/10;
    } else {
        index = (getIndex(device) + 5)/10;
    }
    AudioSystem.setStreamVolumeIndex(mStreamType, index, device);
}
```

AudioSystem的setStreamVolumeIndex是个native函数，从这里到跳到c++代码
```java
//--->frameworks/base/media/java/android/media/AudioSystem.java
public static native int setStreamVolumeIndex(int stream, int index,int device);
```

jni的代码没做处理，直接转发给c++层的AudioSystem
```cpp
//--->frameworks/base/core/jni/android_media_AudioSystem.cpp
static JNINativeMethod gMethods[] = { ...
 {"setStreamVolumeIndex","(III)I",   (void *)android_media_AudioSystem_setStreamVolumeIndex},
...};

static jint android_media_AudioSystem_setStreamVolumeIndex(
    JNIEnv *env,jobject thiz,jint stream,jint index,jint device)
{
    return (jint) check_AudioSystem_Command(AudioSystem::setStreamVolumeIndex(
        static_cast <audio_stream_type_t>(stream),index,(audio_devices_t)device));
}
```

AudioSystem又踢给AudioPolicyService(这里是binder通信，从这里跳到服务端处理)
```cpp
//--->frameworks/av/media/libmedia/AudioSystem.cpp
status_t AudioSystem::setStreamVolumeIndex(audio_stream_type_t stream,
    int index,audio_devices_t device)
{
    const sp<IAudioPolicyService>& aps = AudioSystem::get_audio_policy_service();
    if (aps == 0) return PERMISSION_DENIED;
    return aps->setStreamVolumeIndex(stream, index, device);
}
```

AudioPolicyService做了些权限和参数检查，转发给AudioPolicyManager
```cpp
//---frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp
status_t AudioPolicyService::setStreamVolumeIndex(audio_stream_type_t stream,
    int index,audio_devices_t device)
{
    if (mAudioPolicyManager == NULL) return NO_INIT;
    if (!settingsAllowed()) return PERMISSION_DENIED;
    if (uint32_t(stream) >= AUDIO_STREAM_PUBLIC_CNT) return BAD_VALUE;
    Mutex::Autolock _l(mLock);
    return mAudioPolicyManager->setStreamVolumeIndex(stream,index,device);
}
```

AudioPolicyManager的处理比较复杂，主要是包括了音频策略的判断
```cpp
//--->frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
status_t AudioPolicyManager::setStreamVolumeIndex(audio_stream_type_t stream,
    int index,audio_devices_t device)
{

    // ...(检查音量及设备是否为audio设备)
    // ...(策略判断)
    if ((device != AUDIO_DEVICE_OUT_DEFAULT) && 
        (device & (strategyDevice | accessibilityDevice)) == 0) {
        return NO_ERROR;
    }

    // ...(设置每个输出设备的音量)
    status_t volStatus = checkAndSetVolume(stream, index, desc, curDevice);
    // ...
}

status_t AudioPolicyManager::checkAndSetVolume(audio_stream_type_t stream,int index,
    const sp<AudioOutputDescriptor>& outputDesc,audio_devices_t device,int delayMs,bool force)
{
    // ...(do not change actual stream volume if the stream is muted)
    // ...(do not change in call volume if bluetooth is connected and vice versa)

    // 声音等级与真正参数的转换
    float volumeDb = computeVolume(stream, index, device);  
    // 设置输出设备的声音
    outputDesc->setVolume(volumeDb, stream, device, delayMs, force);

    // 设置通话的音量？？
    if (stream == AUDIO_STREAM_VOICE_CALL || stream == AUDIO_STREAM_BLUETOOTH_SCO) {
        // ...
        mpClientInterface->setVoiceVolume(voiceVolume, delayMs);
        // ...
    }
    return NO_ERROR;
}
```

AudioOutputDescriptor是音频设备描述符，outputDesc是SwAudioOutputDescriptor类型。
```cpp
//--->/frameworks/av/services/audiopolicy/common/managerdefinitions/src/AudioOutputDescriptor.cpp
 bool AudioOutputDescriptor::setVolume(float volume,audio_stream_type_t stream,
    audio_devices_t device __unused,uint32_t delayMs,bool force)
{
    // We actually change the volume if:
    // - the float value returned by computeVolume() changed
    // - the force flag is set
    if (volume != mCurVolume[stream] || force) {
        ALOGV("setVolume() for stream %d, volume %f, delay %d", stream, volume, delayMs);
        mCurVolume[stream] = volume;
        return true;
    }
    return false;
}

bool SwAudioOutputDescriptor::setVolume(float volume,audio_stream_type_t stream,
    audio_devices_t device,uint32_t delayMs,bool force)
{
    bool changed = AudioOutputDescriptor::setVolume(volume, stream, device, 
        delayMs, force);

    if (changed) {
        // Force VOICE_CALL to track BLUETOOTH_SCO stream volume when bluetooth audio is
        // enabled
        float volume = Volume::DbToAmpl(mCurVolume[stream]);
        if (stream == AUDIO_STREAM_BLUETOOTH_SCO) {
            mClientInterface->setStreamVolume(AUDIO_STREAM_VOICE_CALL, volume,
                mIoHandle, delayMs);
        }
        mClientInterface->setStreamVolume(stream, volume, mIoHandle, delayMs);
    }
    return changed;
}
```

AudioOutputDescriptor的mClientInterface是AudioPolicyService，所以会转到AudioPolicyService的setStreamVolume
AudioPolicyService异步执行这个操作,最后会转到AudioSystem的setStreamVolume。
```cpp
//--->frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp
int AudioPolicyService::setStreamVolume(audio_stream_type_t stream,float volume,
    audio_io_handle_t output,int delayMs)
{

    return (int)mAudioCommandThread->volumeCommand(stream, volume,output, delayMs);
}

status_t AudioPolicyService::AudioCommandThread::volumeCommand(audio_stream_type_t stream,
    float volume,audio_io_handle_t output,int delayMs)
{
    // ...(封装了一下data跟command)
    return sendCommand(command, delayMs);
}

status_t AudioPolicyService::AudioCommandThread::sendCommand(sp<AudioCommand>& command, int delayMs) {
    // ...(一些命令队列的操作)
}

// 处理函数
bool AudioPolicyService::AudioCommandThread::threadLoop()
{
    // ...
    while (!exitPending())
    {
        // ...
        switch (command->mCommand) {
        // ...
        case SET_VOLUME: 
            // ...(Lock)
            VolumeData *data = (VolumeData *)command->mParam.get();
            command->mStatus = AudioSystem::setStreamVolume(data->mStream,
                data->mVolume,data->mIO);
        break;
        // ...
    }
}
```

AudioSystem又转到AudioFlinger
```cpp
//--->frameworks/av/media/libmedia/AudioSystem.cpp 
status_t AudioSystem::setStreamVolume(audio_stream_type_t stream, float value,
    audio_io_handle_t output)
{
    // ...(权限参数检查)
    af->setStreamVolume(stream, value, output);
    return NO_ERROR;
}
AudioFlinger会去获取output对应的PlaybackThread并设置PlaybackThread的音量，如果output == AUDIO_IO_HANDLE_NONE，则设置所有PlaybackThread的音量。

//--->frameworks/av/services/audioflinger/AudioFlinger.cpp
status_t AudioFlinger::setStreamVolume(audio_stream_type_t stream, float value,
        audio_io_handle_t output)
{
    // ...(权限检查)
    // ...(流类型检查)

    AutoMutex lock(mLock);
    // ...(获取对应设备的PlaybackTread)

    // ???
    mStreamTypes[stream].volume = value;

    if (thread == NULL) {   // output == AUDIO_IO_HANDLE_NONE
        for (size_t i = 0; i < mPlaybackThreads.size(); i++) {
            mPlaybackThreads.valueAt(i)->setStreamVolume(stream, value);
        }
    } else {
        thread->setStreamVolume(stream, value);
    }

    return NO_ERROR;
}
```

PlaybackThread设置mStreamTypes的volume。并唤醒PlaybackThread线程
```cpp
void AudioFlinger::PlaybackThread::setStreamVolume(audio_stream_type_t stream, float value)
{
    Mutex::Autolock _l(mLock);
    mStreamTypes[stream].volume = value;
    broadcast_l();
}
```

不同类型的Thread貌似有不同使用方法 MixerThread是在prepareTracks_l里使用，最后会设置AudioMixer的参数
```cpp
//--->frameworks/av/services/audioflinger/Threads.cpp
AudioFlinger::PlaybackThread::mixer_state AudioFlinger::MixerThread::prepareTracks_l(
    Vector< sp<Track> > *tracksToRemove) {
    // ...
    // FastTrack
    track->mCachedVolume = masterVolume * mStreamTypes[track->streamType()].volume;

    // ...
    // NormalTrack
    // 这里涉及到了左右声道的音量的计算
    // compute volume for this track
    uint32_t vl, vr;       // in U8.24 integer format
    float vlf, vrf, vaf;   // in [0.0, 1.0] float format
    float typeVolume = mStreamTypes[track->streamType()].volume;
    float v = masterVolume * typeVolume;
    // ...
    //计算完设置混音器的参数
    mAudioMixer->setParameter(name, param, AudioMixer::VOLUME0, &vlf);
    mAudioMixer->setParameter(name, param, AudioMixer::VOLUME1, &vrf);
    mAudioMixer->setParameter(name, param, AudioMixer::AUXLEVEL, &vaf);
    // ...
}

// 最后会调用到mAudioMixer的setVolumeRampVariables
static inline bool setVolumeRampVariables(float newVolume, int32_t ramp,
    int16_t *pIntSetVolume, int32_t *pIntPrevVolume, int32_t *pIntVolumeInc,
    float *pSetVolume, float *pPrevVolume, float *pVolumeInc){...}
```

DirectOutputThread在processVolume_l里使用(processVolume_l在prepareTracks_l中被调用)
processVolume_l直接设置了输出设备的volume
```cpp
//--->frameworks/av/services/audioflinger/Threads.cpp   
void AudioFlinger::DirectOutputThread::processVolume_l(Track *track, bool lastTrack) {
    ...
    float typeVolume = mStreamTypes[track->streamType()].volume;
    float v = mMasterVolume * typeVolume;
    // ...(一系列的设置)
    if (mOutput->stream->set_volume) {
        mOutput->stream->set_volume(mOutput->stream, left, right);
    }
}
```

还有其他的几种Thread都是上面两种Thread的子类，处理方式是一致的
