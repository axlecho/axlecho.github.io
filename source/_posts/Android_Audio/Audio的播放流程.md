---
title: Audio的播放流程
date: 2019-01-12 20:53:32
tags: 
- android_framework
cover: /images/ape_fwk_audio.png
---


这是基于Android5.1分析的，前几版本好像有些不同，6.0没改太多，不过大体思想是一致的

播放就像个排水机，AuidoPolicyService是阀门，AudioFlinger是排水池，PlaybackThread是发动机，Track是源，AudioOutput是排水孔。AudioTrack是水桶

排水首先要凿个孔（openOutput），然后添加发动机（建立PlaybackThread），然后将源接到水桶上（建立Track），选择排水孔（selectOutput），开启相应的发动机（PlaybackThread从睡眠中唤醒），然后就各自排水了。。。。

AudioTrack服务端的启动及准备
---
服务端的指的是AudioFlinger跟AudioPolicyService等音频相关的服务，这些服务会在系统开机的时候启动
在系统启动完成后，客户端（一般都是app）就能利用这些服务来使用系统提供的功能

AUDIO相关的服务启动
---
开机时系统启动各种服务，AudioFlinger跟AudioPolicyService和一些音频相关的服务会在此启动。
各服务的Instantiate()函数在BinderService.h中定义实现，主要是用于抽象出注册服务的操作。
BinderService是个模板类，服务继承该类后可以直接注册到systemserver。
```java
//--->frameworks/av/media/mediaserver/main_mediaserver.cpp
int main(int argc __unused, char** argv)
{
    ...
    AudioFlinger::instantiate();
    MediaPlayerService::instantiate();
    AudioPolicyService::instantiate();
    ...
}

//--->frameworks/native/include/binder/BinderService.h
template<typename SERVICE>
class BinderService
{
public:
    static status_t publish(bool allowIsolated = false) {
        sp<IServiceManager> sm(defaultServiceManager());
        // 这里用模板生成了具体服务的对象
        // new SERVICE()将会调用服务(AudioFlinger，AudioPolicyService等)的构造函数
        return sm->addService(
                String16(SERVICE::getServiceName()),
                new SERVICE(), allowIsolated);
    }
    ...
    static void instantiate() { publish(); }
    ...

};
```

AUDIOFLINGER的创建
---
AudioFlinger承担混音工作。（总之很重要啦）
AudioFlinger的构造函数主要是对成员变量和调试工具的初始化。
onFirstRef一般做进一步的初始化工作，AudioFlinger暂时没有在该函数中做重要的工作。
```c
//--->frameworks/av/services/audioflinger.cpp
AudioFlinger::AudioFlinger()
    : BnAudioFlinger(),
      mPrimaryHardwareDev(NULL),
      mAudioHwDevs(NULL),
      mHardwareStatus(AUDIO_HW_IDLE),
      mMasterVolume(1.0f),
      mMasterMute(false),
      mNextUniqueId(1),
      mMode(AUDIO_MODE_INVALID),
      mBtNrecIsOff(false),
      mIsLowRamDevice(true),
      mIsDeviceTypeKnown(false),
      mGlobalEffectEnableTime(0),
      mPrimaryOutputSampleRate(0)
{
...
#ifdef TEE_SINK
    ....
#endif
...
}

void AudioFlinger::onFirstRef()
{
    Mutex::Autolock _l(mLock);
    ...
    mPatchPanel = new PatchPanel(this);
    mMode = AUDIO_MODE_NORMAL;
}
```

AUDIOPOLICYSERVICE的创建
---
AudioPolicyService用于控制音频播放策略（比如插耳机的时候来电用什么设备去播放音乐）、管理音频设备等

AudioPolicyService的构造函数更简单，只是初始化主要成员。
AudioPolicyService会在onFristRef中做比较多的工作，比如创建command线程，初始化重要成员mAudioPolicyManager。
```
//--->frameworks/av/services/audiopolicy/AudioPolicyService.cpp
AudioPolicyService::AudioPolicyService()
    : BnAudioPolicyService(),
    mpAudioPolicyDev(NULL),
    mpAudioPolicy(NULL),
    mAudioPolicyManager(NULL),
    mAudioPolicyClient(NULL),
    mPhoneState(AUDIO_MODE_INVALID)
{}

void AudioPolicyService::onFirstRef()
{
    ...
    {
        Mutex::Autolock _l(mLock);
        mTonePlaybackThread = new AudioCommandThread(String8("ApmTone"), this);
        mAudioCommandThread = new AudioCommandThread(String8("ApmAudio"), this);
        mOutputCommandThread = new AudioCommandThread(String8("ApmOutput"), this);

#ifdef USE_LEGACY_AUDIO_POLICY
    ...(暂时这宏意义不明)
#else
        mAudioPolicyClient = new AudioPolicyClient(this);
        mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
#endif
    }
    ...(效果相关)
}
```

AUDIOPOLICYMANAGER的创建
---

AudioPolicyManager作为音频调度策略的实现，在AudioPolicyService关于音频调度的基本都是直接转发给AudioPolicyManager。
（貌似可以重载AudioPolicyManager来改动音频策略的实现,6.0开始可以直接动态选择不同的AudiPolicyManger实现）

在构造函数中，打开了所有能用的音频设备和录音设备，并调用AudioPolicyService创建了相应设备的混音线程。
```c
//--->frameworks/av/services/audiopolicy/AudioPolicyManager.cpp
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
    :mPrimaryOutput((audio_io_handle_t)0),
    ...

{
    mpClientInterface = clientInterface;
    ...
    // 加载音频模块
    defaultAudioPolicyConfig();
    ...
    for (size_t i = 0; i < mHwModules.size(); i++) {
        ...
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++) {
            ...
            status_t status = mpClientInterface->openOutput(outProfile->mModule->mHandle,  // 打开音频设备
                &output,
                &config,
                &outputDesc->mDevice,
                String8(""),
                &outputDesc->mLatency,
                outputDesc->mFlags);
        }
    }

    ...(打开录音设备)
}
```

mpClientInterface就是AudioPolicyService，AudioPolicyService最终会调用AudioFlinger的openOutput函数。
```c
//--->frameworks/av/services/audiopolicy/AudioPolicyClientImpl.cpp
status_t AudioPolicyService::AudioPolicyClient::openOutput(audio_module_handle_t module,
    audio_io_handle_t *output,
    audio_config_t *config,
    audio_devices_t *devices,
    const String8& address,
    uint32_t *latencyMs,
    audio_output_flags_t flags)
{
    sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
    if (af == 0) {
        ALOGW("%s: could not get AudioFlinger", __func__);
        return PERMISSION_DENIED;
    }
    return af->openOutput(module, output, config, devices, address, latencyMs, flags);
}
```

AudioFlinger的openOutput中会针对输出设备的类型创建了一个PlaybackThread。
PlaybackThread在AudioFlinger相当重要，相当于音频系统的发动机

PlaybackThread有几种，比较常见有MixerThread，蓝牙耳机设备需要外放（比如ring类型的流需要同时从耳机与喇叭出来）的时候使用DuplicatingThread。
```c
//--->frameworks/av/services/audioflinger/AudioFlinger.cpp
status_t AudioFlinger::openOutput(audio_module_handle_t module,
    audio_io_handle_t *output,
    audio_config_t *config,
    audio_devices_t *devices,
    const String8& address,
    uint32_t *latencyMs,
    audio_output_flags_t flags){
    ...
    sp<PlaybackThread> thread = openOutput_l(module, output, config, *devices, address, flags);
    ...
}

sp<AudioFlinger::PlaybackThread> AudioFlinger::openOutput_l(audio_module_handle_t module,
    audio_io_handle_t *output,
    audio_config_t *config,
    audio_devices_t devices,
    const String8& address,
    audio_output_flags_t flags)
{
    ...
    status_t status = hwDevHal->open_output_stream(
        hwDevHal,
        *output,
        devices,
        flags,
        config,
        &outStream,
        address.string());
    ...
    // 根据flags 创建相应的thread
    AudioStreamOut *outputStream = new AudioStreamOut(outHwDev, outStream, flags);
    PlaybackThread *thread;
    if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
        thread = new OffloadThread(this, outputStream, *output, devices);  
    } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
            || !isValidPcmSinkFormat(config->format)
            || !isValidPcmSinkChannelMask(config->channel_mask)) {
        thread = new DirectOutputThread(this, outputStream, *output, devices);
    } else {
        thread = new MixerThread(this, outputStream, *output, devices);
    }
    ...
}
```

PlaybackThread创建完之后在onFirstRef()中会自己启动起来,并会在threadLoop等待一个AudioTrack的连接。
```c
//--->frameworks/av/services/audioflinger/Threads.cpp
AudioFlinger::MixerThread::MixerThread(const sp<AudioFlinger>& audioFlinger, AudioStreamOut* output,
        audio_io_handle_t id, audio_devices_t device, type_t type)
    :   PlaybackThread(audioFlinger, output, id, device, type),
        mFastMixerFutex(0)
{...(主要初始化了Mixer)}

AudioFlinger::PlaybackThread::PlaybackThread(const sp<AudioFlinger>& audioFlinger,
    AudioStreamOut* output,
    audio_io_handle_t id,
    audio_devices_t device,
    type_t type)
    :   ThreadBase(audioFlinger, id, device, AUDIO_DEVICE_NONE, type),
    ...
{...(初始化了音量相关的参数和获取输出设备的参数)}

// 在此PlaybackThread会跑起来，运行threadLoop()
void AudioFlinger::PlaybackThread::onFirstRef()
{
    run(mName, ANDROID_PRIORITY_URGENT_AUDIO);
}

// threadLoop是整个AudioFlinger的核心，混音的工作在此进行
bool AudioFlinger::PlaybackThread::threadLoop()
{
    ...
    cacheParameters_l();
    ...
    checkSilentMode_l();
    while (!exitPending())
    {
        ...
        processConfigEvents_l();
        ...
        size_t size = mActiveTracks.size();
        for (size_t i = 0; i < size; i++) {
            sp<Track> t = mActiveTracks[i].promote();
            if (t != 0) {
                mLatchD.mFramesReleased.add(t.get(),
                        t->mAudioTrackServerProxy->framesReleased());
            }
        }
        ...
        saveOutputTracks();
        ...
        threadLoop_standby();
        ...
        clearOutputTracks();
        ...
        checkSilentMode_l();
        ...
        mMixerStatus = prepareTracks_l(&tracksToRemove);
        ...
        // 混音，主要设置AudioMixer的参数
        threadLoop_mix();
        ...
        ssize_t ret = threadLoop_write();
        ...
        threadLoop_drain();
        ...
        threadLoop_removeTracks(tracksToRemove);
        tracksToRemove.clear();
    }
    threadLoop_exit();
    ...
}
```
到此各种线程服务准备完毕，可以播放AudioTrack了

客户端播放AudioTrack
---
Android SDK向外提供了MediaPlayer和比较底层的AudioTrack接口，MediaPlayer做一些解码的工作，最终还是会使用到AudioTrack。

AUDIOTRACK的构建
--
调用从app开始，首先是AudioTrack(java)的构造函数,会调用jni的native_setup
```java
//--->frameworks/base/media/java/android/media/AudioTrack.java
 public AudioTrack(AudioAttributes attributes, AudioFormat format, int bufferSizeInBytes,
            int mode, int sessionId)throws IllegalArgumentException {
    ...(参数检查)
    mStreamType = AudioSystem.STREAM_DEFAULT;
    int[] session = new int[1];
    session[0] = sessionId;
    // native initialization
    int initResult = native_setup(new WeakReference<AudioTrack>(this), mAttributes,
            mSampleRate, mChannels, mAudioFormat,
            mNativeBufferSizeInBytes, mDataLoadMode, session);
    ...
}
```

jni对应的函数是android_media_AudioTrack_setup，生成一个AudioTrack(c++)并设置其参数
```c
//--->android/frameworks/base/core/jni/android_media_AudioTrack.cpp
static jint android_media_AudioTrack_setup(JNIEnv *env, jobject thiz, jobject weak_this,
    jobject jaa,
    jint sampleRateInHertz, jint javaChannelMask,
    jint audioFormat, jint buffSizeInBytes, jint memoryMode, jintArray jSession) {

    ...(检查参数)
    sp<AudioTrack> lpTrack = new AudioTrack();
    ...
    // 看样子很重要
    AudioTrackJniStorage* lpJniStorage = new AudioTrackJniStorage();

    // 分不同的模式设置track
    switch (memoryMode) {
    case MODE_STREAM:
        status = lpTrack->set(...);
        break;
    case MODE_STATIC:
        // AudioTrack is using shared memory
        status = lpTrack->set(...);
        break;
    }
    ...(错误处理)
}
```

AudioTrack的无参构造函数很简单，主要工作还是放在set里面
```java
//--->frameworks/av/media/libmedia/AudioTrack.cpp
AudioTrack::AudioTrack()
    : mStatus(NO_INIT),
      mIsTimed(false),
      mPreviousPriority(ANDROID_PRIORITY_NORMAL),
      mPreviousSchedulingGroup(SP_DEFAULT),
      mPausedPosition(0)
{
    mAttributes.content_type = AUDIO_CONTENT_TYPE_UNKNOWN;
    mAttributes.usage = AUDIO_USAGE_UNKNOWN;
    mAttributes.flags = 0x0;
    strcpy(mAttributes.tags, "");
}
```

AudioTrack的set的工作很多，服务端的track其实是在此时建立的。
```c
//--->frameworks/av/media/libmedia/AudioTrack.cpp
status_t AudioTrack::set(
        audio_stream_type_t streamType,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t frameCount,
        audio_output_flags_t flags,
        callback_t cbf,
        void* user,
        uint32_t notificationFrames,
        const sp<IMemory>& sharedBuffer,
        bool threadCanCallJava,
        int sessionId,
        transfer_type transferType,
        const audio_offload_info_t *offloadInfo,
        int uid,
        pid_t pid,
        const audio_attributes_t* pAttributes)
{
    ...(设置参数等)
    if (cbf != NULL) {
        mAudioTrackThread = new AudioTrackThread(*this, threadCanCallJava);
        mAudioTrackThread->run("AudioTrack", ANDROID_PRIORITY_AUDIO, 0 /*stack*/);
    }


    status_t status = createTrack_l();
    ...(错误处理)
}
```

这里插入一个track选择output的一个过程,在工作中经常碰到这个鬼东西。在创建track的时候其实音频的路由已经定好了，之前还一直以为在start后选，在createTrack之前，会调用getOutputForAttr来获取当前的流对应的output（后面有空会补一下output跟device及stream的乱七八糟的关系）
```c
//--->frameworks/av/media/libmedia/AudioTrack.cpp
status_t AudioTrack::createTrack_l()
{
    const sp<IAudioFlinger>& audioFlinger = AudioSystem::get_audio_flinger();
    ...(计算FrameCount，Latency等

    // 这里的output即到底层的通路，可以用adb shell dumpsys media.audio_policy 看到所有的output
    status_t status = AudioSystem::getOutputForAttr(attr, &output,
                                   (audio_session_t)mSessionId, &streamType, mClientUid,
                                   mSampleRate, mFormat, mChannelMask,
                                   mFlags, mSelectedDeviceId, mOffloadInfo);

    sp<IAudioTrack> track = audioFlinger->createTrack(streamType,  //创建服务端的track
        mSampleRate,
        format,
        mChannelMask,
        &temp,
        &trackFlags,
        mSharedBuffer,
        output,
        tid,
        &mSessionId,
        mClientUid,
        &status);

    ...(debug代码等)
    // AudioTrackClientProxy主要实现管理cblk和服务端通信
    mProxy = new AudioTrackClientProxy(cblk, buffers, frameCount, mFrameSizeAF);
    ...
}

// AudioSystem转发给AudiopolicyService
//--->frameworks/av/media/libmedia/AudioSystem.cpp
status_t AudioSystem::getOutputForAttr(const audio_attributes_t *attr,
                                        audio_io_handle_t *output,
                                        audio_session_t session,
                                        audio_stream_type_t *stream,
                                        uid_t uid,
                                        uint32_t samplingRate,
                                        audio_format_t format,
                                        audio_channel_mask_t channelMask,
                                        audio_output_flags_t flags,
                                        audio_port_handle_t selectedDeviceId,
                                        const audio_offload_info_t *offloadInfo)
{
    const sp<IAudioPolicyService>& aps = AudioSystem::get_audio_policy_service();
    if (aps == 0) return NO_INIT;
    return aps->getOutputForAttr(attr, output, session, stream, uid,
                                 samplingRate, format, channelMask,
                                 flags, selectedDeviceId, offloadInfo);
}

// AudioPolicyService 转发给AudioPolicyManager
//--->/frameworks/av/services/audiopolicy/service/AudioPolicyInterfaceImpl.cpp
status_t AudioPolicyService::getOutputForAttr(const audio_attributes_t *attr,
                                              audio_io_handle_t *output,
                                              audio_session_t session,
                                              audio_stream_type_t *stream,
                                              uid_t uid,
                                              uint32_t samplingRate,
                                              audio_format_t format,
                                              audio_channel_mask_t channelMask,
                                              audio_output_flags_t flags,
                                              audio_port_handle_t selectedDeviceId,
                                              const audio_offload_info_t *offloadInfo)
{
    ...
    if (IPCThreadState::self()->getCallingPid() != getpid_cached || uid == (uid_t)-1) {
        uid_t newclientUid = IPCThreadState::self()->getCallingUid();
        if (uid != (uid_t)-1 && uid != newclientUid) {
            ALOGW("%s uid %d tried to pass itself off as %d", __FUNCTION__, newclientUid, uid);
        }
        uid = newclientUid;
    }
    return mAudioPolicyManager->getOutputForAttr(attr, output, session, stream, uid, samplingRate,
                                    format, channelMask, flags, selectedDeviceId, offloadInfo);
}

// 这里涉及到了流和策略及output的选择，还是比较重要的吧
//--->frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
status_t AudioPolicyManager::getOutputForAttr(const audio_attributes_t *attr,
                                              audio_io_handle_t *output,
                                              audio_session_t session,
                                              audio_stream_type_t *stream,
                                              uid_t uid,
                                              uint32_t samplingRate,
                                              audio_format_t format,
                                              audio_channel_mask_t channelMask,
                                              audio_output_flags_t flags,
                                              audio_port_handle_t selectedDeviceId,
                                              const audio_offload_info_t *offloadInfo)
{
    ...(流类型判断)
    // 获取路由策略
    routing_strategy strategy = (routing_strategy) getStrategyForAttr(&attributes);
    // 获取策略对应的设备
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);
    ...(判断flag)
    // 获取对应设备的output
    *output = getOutputForDevice(device, session, *stream,
                                 samplingRate, format, channelMask,
                                 flags, offloadInfo);
    ...
    return NO_ERROR;
}
```

接下来的服务端AudioFlinger会调用PlaybackThread的createTrack_l创建Track。
```c
//--->frameworks/av/services/audioflinger/AudioFlinger.cpp
sp<IAudioTrack> AudioFlinger::createTrack(
    audio_stream_type_t streamType,
    uint32_t sampleRate,
    audio_format_t format,
    audio_channel_mask_t channelMask,
    size_t *frameCount,
    IAudioFlinger::track_flags_t *flags,
    const sp<IMemory>& sharedBuffer,
    audio_io_handle_t output,
    pid_t tid,
    int *sessionId,
    int clientUid,
    status_t *status)
{
    ...(参数检查)
    // 根据output选择对应的thread
    PlaybackThread *thread = checkPlaybackThread_l(output);
    ...

    track = thread->createTrack_l(client, streamType, sampleRate, format,
            channelMask, frameCount, sharedBuffer, lSessionId, flags, tid, clientUid, &lStatus);
    ...
    // return handle to client
    trackHandle = new TrackHandle(track);

Exit:
    *status = lStatus;
    return trackHandle;
}
```

PlaybackThread会创建Track的实体对象
```c
//--->android/frameworks/av/services/audioflinger/Threads.cpp
sp<AudioFlinger::PlaybackThread::Track> AudioFlinger::PlaybackThread::createTrack_l(
    const sp<AudioFlinger::Client>& client,
    audio_stream_type_t streamType,
    uint32_t sampleRate,
    audio_format_t format,
    audio_channel_mask_t channelMask,
    size_t *pFrameCount,
    const sp<IMemory>& sharedBuffer,
    int sessionId,
    IAudioFlinger::track_flags_t *flags,
    pid_t tid,
    int uid,
    status_t *status)
{
    ...(设置参数)
    if (!isTimed) {
        track = new Track(this, client, streamType, sampleRate, format,
                          channelMask, frameCount, NULL, sharedBuffer,
                          sessionId, uid, *flags, TrackBase::TYPE_DEFAULT);
    } else {
        track = TimedTrack::create(this, client, streamType, sampleRate, format,
                channelMask, frameCount, sharedBuffer, sessionId, uid);
    }
    ...
}
```
(**TODO**:这样应有Track构建的分析)

到这一步，客户端跟服务端的track都创建好了，就等着播放了。

AUDIOTRACK的播放
---
AudioTrack(java)会调用play进行音频播放的准备。
```java
//--->frameworks/base/media/java/android/media/AudioTrack.java
public void play()
throws IllegalStateException {
    if (mState != STATE_INITIALIZED) {
        throw new IllegalStateException("play() called on uninitialized AudioTrack.");
    }
    if (isRestricted()) {
        setVolume(0);
    }
    synchronized(mPlayStateLock) {
        native_start();
        mPlayState = PLAYSTATE_PLAYING;
    }
}
```

play()会调用jni的native_start(),对应的函数是android_media_AudioTrack_start(), android_media_AudioTrack_start()只是做了转发，最后会调用 AudioTrack(c++)的start(), AudioTrack的start又会调用TrackHandle(服务端的Track代理) 的start，最后会调用到服务端Track的start。
```c
//--->frameworks/av/media/libmedia/AudioTrack.cpp
status_t AudioTrack::start()
{
    ...(标记位和参数的检测)
    status = mAudioTrack->start();
    ...
}
```

TrackHandle仅仅做过转发,最后会触发PlaybackThread的addTrack_l()
```c
//--->frameworks/av/services/audioflinger/Tracks.cpp
status_t AudioFlinger::TrackHandle::start() {
    return mTrack->start();
}

status_t AudioFlinger::PlaybackThread::Track::start(AudioSystem::sync_event_t event __unused,
                                                    int triggerSession __unused)
{
    ...
    status = playbackThread->addTrack_l(this);
    ...
}
```

PlaybackTread的addTrack_l主要工作是添加track到mActiveTracks,并激活沉睡的PlaybackTread。
```
//--->frameworks/av/services/audioflinger/Threads.cpp
status_t AudioFlinger::PlaybackThread::addTrack_l(const sp<Track>& track)
{
    ...
    mActiveTracks.add(track);
    ...
    onAddNewTrack_l();
}


void AudioFlinger::PlaybackThread::onAddNewTrack_l()
{
    ALOGV("signal playback thread");
    broadcast_l();
}

void AudioFlinger::PlaybackThread::broadcast_l()
{
    // Thread could be blocked waiting for async
    // so signal it to handle state changes immediately
    // If threadLoop is currently unlocked a signal of mWaitWorkCV will
    // be lost so we also flag to prevent it blocking on mWaitWorkCV
    mSignalPending = true;
    mWaitWorkCV.broadcast();
}
```

接下来就是写入音频数据 AudioTrack.java会调用write写入音频数据(播放声音)
```
//--->frameworks/base/media/java/android/media/AudioTrack.java
public int write(byte[] audioData, int offsetInBytes, int sizeInBytes) {
    int ret = native_write_byte(audioData, offsetInBytes, sizeInBytes, mAudioFormat,
            true /*isBlocking*/);
}
```

native_write_byte()会调用jni的android_media_AudioTrack_write_byte，会调用jni的android_media_AudioTrack_write_byte，
主要是获取java传下来的数据， 并调用writeToTrack来向共享内存写入数据，writeToTrack又分track是否为stream或static来做不同的处理。
```
//--->android/frameworks/base/core/jni/android_media_AudioTrack.cpp
static jint android_media_AudioTrack_write_byte(JNIEnv *env,  jobject thiz,
    jbyteArray javaAudioData,
    jint offsetInBytes, jint sizeInBytes,
    jint javaAudioFormat,
    jboolean isWriteBlocking)
{
    ...
    cAudioData = (jbyte *)env->GetByteArrayElements(javaAudioData, NULL);
    ...
    jint written = writeToTrack(lpTrack, javaAudioFormat, cAudioData, offsetInBytes, sizeInBytes,
            isWriteBlocking == JNI_TRUE /* blocking */);
    ...
}

jint writeToTrack(const sp<AudioTrack>& track, jint audioFormat, const jbyte* data,
                  jint offsetInBytes, jint sizeInBytes, bool blocking = true) {
    if (track->sharedBuffer() == 0) {
        written = track->write(data + offsetInBytes, sizeInBytes, blocking);
    } else {
        ...
        switch (format) {
        default:
        case AUDIO_FORMAT_PCM_FLOAT:
        case AUDIO_FORMAT_PCM_16_BIT: {
            ...
            memcpy(track->sharedBuffer()->pointer(), data + offsetInBytes, sizeInBytes);
            ...
            } break;
        case AUDIO_FORMAT_PCM_8_BIT: {
            ...
            memcpy_to_i16_from_u8(dst, src, count);
            ...
            } break;

        }
    }
    return written;

}
```

其中stream类型的track会调用AudioTrack(c++)的write,AudioTrack会使用obtainBuffer获取一块共享内存，
并写入数据，写完后用releaseBuffer释放共享内存。（就可以给AudioFlingr使用了）
```c
//--->frameworks/av/media/libmedia/AudioTrack.cpp
ssize_t AudioTrack::write(const void* buffer, size_t userSize, bool blocking)
{
    ...(参数检查)
    while (userSize >= mFrameSize) {
          audioBuffer.frameCount = userSize / mFrameSize;
          status_t err = obtainBuffer(&audioBuffer,
                  blocking ? &ClientProxy::kForever : &ClientProxy::kNonBlocking);
          ...(错误处理)
          ...(memcpy buffer -> audioBuffer);
          ...(计算剩余数据)
          releaseBuffer(&audioBuffer);
      }

}

status_t AudioTrack::obtainBuffer(Buffer* audioBuffer, int32_t waitCount)
{
    ...（参数转换跟计算)
    return obtainBuffer(audioBuffer, requested);
}

status_t AudioTrack::obtainBuffer(Buffer* audioBuffer, const struct timespec *requested,
        struct timespec *elapsed, size_t *nonContig)
{
    ...（参数转换）
    status = proxy->obtainBuffer(&buffer, requested, elapsed);
    ...(结果的填充)
}

void AudioTrack::releaseBuffer(Buffer* audioBuffer)
{
    ...
    mProxy->releaseBuffer(&buffer);
    ...
}
```

obtainBuffer跟releaseBuffer的具体实现交给了AudioTrackClientProxy来实现，主要是管理cblk对象与共享内存。应该深入研究一下。

服务端读取共享内存的音频数据是在PlaybackThread的threadLoop()中进行的，MixerThread也使用该函数，
不过重写了threadLoop_mix()等关键函数(典型的多态)。
```c
//--->frameworks/av/services/audioflinger/Threads.cpp
bool AudioFlinger::PlaybackThread::threadLoop()
{
    ...
    cacheParameters_l();
    ...
    acquireWakeLock();
    ...
    checkSilentMode_l();
    while (!exitPending()){
        ...(lock)
        processConfigEvents_l();
        ...
        saveOutputTracks();
        ...(wakelock wait,not understand,**mark**)
        ...
        threadLoop_standby(); // 准备音频设备??
        ...(参数检测)
        prepareTracks_l(&tracksToRemove);
        ...
        if (mBytesRemaining == 0) {
            mCurrentWriteLength = 0;
            if (mMixerStatus == MIXER_TRACKS_READY) {
                // threadLoop_mix() sets mCurrentWriteLength
                threadLoop_mix(); // 混音
            } ...(其他情况处理)
            ...(音效处理)
        }

        if (mBytesRemaining) {
              ssize_t ret = threadLoop_write(); // 写到音频设备
              if (ret < 0) {
                  mBytesRemaining = 0;
              } else {
                  mBytesWritten += ret;
                  mBytesRemaining -= ret;
              }
          }...(其他情况处理)
          // 播放完成，删除已经播放的tracks
          threadLoop_removeTracks(tracksToRemove);
          tracksToRemove.clear();
          clearOutputTracks();
          effectChains.clear();
    }

    threadLoop_exit();
    ...
    releaseWakeLock();
    mWakeLockUids.clear();
    mActiveTracksGeneration++;
  }
}
```

这里重点看一下 MixerThread的threadLoop_mix 和 threadLoop_write
threadLoop_mix调用了AudioMixer的process,threadLoop_write最终调用mOutput->stream->write写到驱动里去了
```c
//--->frameworks/av/services/audioflinger/Threads.cpp
void AudioFlinger::MixerThread::threadLoop_mix()
{
    ...
    // mix buffers...
    mAudioMixer->process(pts);
    ...
}

ssize_t AudioFlinger::MixerThread::threadLoop_write()
{
    ...(处理一些fastmix的情况)
    return PlaybackThread::threadLoop_write();
}

ssize_t AudioFlinger::PlaybackThread::threadLoop_write()
{
    ...(一些Sink操作)
    bytesWritten = mOutput->stream->write(mOutput->stream,
                                           (char *)mSinkBuffer + offset, mBytesRemaining);
    ...
}
```

AudioMixer的process是一个hook函数，根据不同的情况会调用不同的函数。具体的调用会调用到AudioMixer中以process开头的一组函数。如

*   void AudioMixer::process__validate(state_t* state, int64_t pts)
*   void AudioMixer::process__nop(state_t* state, int64_t pts)
*   void AudioMixer::process__genericNoResampling(state_t* state, int64_t pts)
*   void AudioMixer::process__genericResampling(state_t* state, int64_t pts)
*   void AudioMixer::process__OneTrack16BitsStereoNoResampling(state_t* state,int64_t pts)
*   void AudioMixer::process__OneTrack24BitsStereoNoResampling(state_t* state,int64_t pts)
*   void AudioMixer::process_NoResampleOneTrack(state_t* state, int64_t pts)

这里以process__OneTrack16BitsStereoNoResampling为例子,在获取buffer的时候使用了Track的getNextBuffer和releaseBuffer。
```c
//--->frameworks/av/services/audioflinger/AudioMixer.cpp
void AudioMixer::process(int64_t pts)
{
    mState.hook(&mState, pts);
}

void AudioMixer::process__OneTrack16BitsStereoNoResampling(state_t* state,
                                                           int64_t pts)
{
    const int i = 31 - __builtin_clz(state->enabledTracks);
    const track_t& t = state->tracks[i];
    AudioBufferProvider::Buffer& b(t.buffer);
    ...(音量设置)
    while (numFrames) {
        ...(Frame的计算)
        t.bufferProvider->getNextBuffer(&b, outputPTS);     // 获取客户端写入的buffer
        ...(错误处理)

        // 混音
        switch (t.mMixerFormat) {
            case AUDIO_FORMAT_PCM_FLOAT:  ... break;
            case AUDIO_FORMAT_PCM_16_BIT: ... break;
            default:LOG_ALWAYS_FATAL("bad mixer format: %d", t.mMixerFormat);
        }
        ...
        t.bufferProvider->releaseBuffer(&b);
}
```

其他问题
---
这里只是简单的过了一下AudioTrack播放音频的流程，其他的地方还有很多东西要看。

*   共享内存的同步(AudioTrackClientProxy的obtainBuffer和releaseBuffer*和bufferProvider的getNextBuffer和releaseBuffer);
*   mOutput->stream->write最终的去向。
*   AudioPolicyService的一些音频策略
*   上层的Mediaplayer的一些工作