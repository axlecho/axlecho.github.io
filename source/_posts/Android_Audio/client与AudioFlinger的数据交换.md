---
title: client与AudioFlinger的数据交换
date: 2019-01-19 22:26:31
categories: Android_Audio
tags: 
    - framework
    - audio
---
AudioTrack与AudioFlinger在两个不同进程，他们之间要通过共享内存进行音频的数据交换。
交换的实现通过环形缓冲去来实现，貌似没有同步机制，从实验结果来看，AudioTrack写满缓冲区后AudioFlinger就会去读取。

数据交换的实现主要在AudioTrackShared.cpp中实现，包括AudioTrackClientProxy和AudioTrackServerProxy。

两边数据通过cblk的flag来进行数据的读写。

```cpp
//两边操作数据的接口
class Proxy : public RefBase {
    ...
public:
    struct Buffer {
        size_t  mFrameCount;            // number of frames available in this buffer
        void*   mRaw;                   // pointer to first frame
        size_t  mNonContig;             // number of additional non-contiguous frames available
    };

protected:
    // 共享内存的一些信息
    audio_track_cblk_t* const   mCblk;  // the control block
    void* const     mBuffers;           // starting address of buffers

    const size_t    mFrameCount;        // not necessarily a power of 2
    const size_t    mFrameSize;         // in bytes
    const size_t    mFrameCountP2;      // mFrameCount rounded to power of 2, streaming mode
    const bool      mIsOut;             // true for AudioTrack, false for AudioRecord
    const bool      mClientInServer;    // true for OutputTrack, false for AudioTrack & AudioRecord
    bool            mIsShutdown;        // latch set to true when shared memory corruption detected
    size_t          mUnreleased;        // unreleased frames remaining from most recent obtainBuffer
};
```

客户端的流程:获取Buffer -> 填充数据 -> 释放Buffer
```cpp
//获取Buffer在obtainBuffer中实现，里面还有一些关于获取失败的等待方式的一些东西  
//cblk的buffer通过rear和front，通过log可以看出rear和front都是增长的，rear - front就是填充了数据的缓冲区，怎么映射到buffer上还要再看。
status_t ClientProxy::obtainBuffer(Buffer* buffer, const struct timespec *requested,
    struct timespec *elapsed)
{
    //time的初始化
    // 几种Timeout方式
    enum {
        TIMEOUT_ZERO,       // requested == NULL || *requested == 0
        TIMEOUT_INFINITE,   // *requested == infinity
        TIMEOUT_FINITE,     // 0 < *requested < infinity
        TIMEOUT_CONTINUE,   // additional chances after TIMEOUT_FINITE
    } timeout;

    // 一个死循环来获取Buffer，通过break和goto end来实现不同的Timeout
    for (;;) {
        // ...(检查cblk的flag)
        // compute number of frames available to write (AudioTrack) or read (AudioRecord)
        int32_t front;
        int32_t rear;
        if (mIsOut) {
            // ...(这里有一大段注释，说android_atomic_acquire_load可能是无用的，但就是要加，就是要任性..)
            front = android_atomic_acquire_load(&cblk->u.mStreaming.mFront);
            rear = cblk->u.mStreaming.mRear;
        } else {
            rear = android_atomic_acquire_load(&cblk->u.mStreaming.mRear);
            front = cblk->u.mStreaming.mFront;
        }

        // 获取填充数据的buffer
        ssize_t filled = rear - front;
        // pipe should not be overfull
        if (!(0 <= filled && (size_t) filled <= mFrameCount)) {
            // ...(gg了)
        }

        // 获取可以利用的空间
        size_t avail = mIsOut ? mFrameCount - filled : filled;

        if (avail > 0) {
            // 'avail' may be non-contiguous, so return only the first contiguous chunk
            // 这里要处理的是这种情况，像如下的buffer(*是数据)
            // __________**********__________
            // 获取到avail是两边空白的和，这里只能要一边
            // ...
            // 获取到了buffer，走人
            status = NO_ERROR;
            break;
        }

        // ...(后面是avail等于0的情况,有可能是server那边没读完，也有可能其他情况,根据不同的Timeout方式选择等待或放弃)
    }
end:    // ...(错误处理等)
}

//填充数据没有特殊的api，一般用memcpy就可以了
memcpy(audioBuffer.i8, buffer, toWrite);
buffer = ((const char *) buffer) + toWrite;
userSize -= toWrite;
written += toWrite;

// 释放数据很简单
void ClientProxy::releaseBuffer(Buffer* buffer)
{
    // ...(参数检查，避免释放不合法的buffer
    mUnreleased -= stepCount;
    audio_track_cblk_t* cblk = mCblk;
    // 其实就只改了一个指针
    if (mIsOut) {
        int32_t rear = cblk->u.mStreaming.mRear;
        android_atomic_release_store(stepCount + rear, &cblk->u.mStreaming.mRear);
    } else {
        int32_t front = cblk->u.mStreaming.mFront;
        android_atomic_release_store(stepCount + front, &cblk->u.mStreaming.mFront);
    }
}
```

服务端也是差不多的流程:获取Buffer -> 使用数据 -> 释放Buffer
```cpp
//获取Buffer在obtainBuffer中实现，方式跟客户端的差不多
status_t ServerProxy::obtainBuffer(Buffer* buffer, bool ackFlush)
{
    // ...(参数检查，避免buffer为空等)
    if (mIsOut) {
        int32_t flush = cblk->u.mStreaming.mFlush;
        rear = android_atomic_acquire_load(&cblk->u.mStreaming.mRear);
        front = cblk->u.mStreaming.mFront;
        if (flush != mFlush) {
            // effectively obtain then release whatever is in the buffer
            // Note:这里有一大段修正Front的 不知在干吗  
        }   
    } else {
        front = android_atomic_acquire_load(&cblk->u.mStreaming.mFront);
        rear = cblk->u.mStreaming.mRear;
    }

    // 计算客户端填了多少数据
    size_t availToServer;
    if (mIsOut) {
        availToServer = filled;
        mAvailToClient = mFrameCount - filled;
    } else {
        availToServer = mFrameCount - filled;
        mAvailToClient = filled;
    }

    // 'availToServer' may be non-contiguous, so return only the first contiguous chunk 
    // ...(这里跟客户端一样也有是去左右其中一段)
no_init:
    // ...(错误处理)
}

// 服务端使用数据场景比较复杂，主要是混音跟重采样比较麻烦
// obtainBuffer被分装在Track(AudioFlinger的一个内部类)的getNextBuffer中
// DirectOutputThread中使用跟客户端差不多，也是直接写
memcpy(curBuf, buffer.raw, buffer.frameCount * mFrameSize);
frameCount -= buffer.frameCount;
curBuf += buffer.frameCount * mFrameSize;

// 其他的基本都是在AudioMixer中被使用，
// 比如重采样，hook是一个函数指针，根据配置会选择不同的采样函数
t.bufferProvider->getNextBuffer(&t.buffer, outputPTS);
t.hook(&t, outTemp + outFrames * t.mMixerChannelCount, t.buffer.frameCount,
    state->resampleTemp, aux);
// 比如混音
t.bufferProvider->getNextBuffer(&b, outputPTS);
const int16_t *in = b.i16;
do {
    uint32_t rl = *reinterpret_cast<const uint32_t *>(in);
    in += 2;
    int32_t l = mulRL(1, rl, vrl) >> 12;
    int32_t r = mulRL(0, rl, vrl) >> 12;
    // clamping...
    l = clamp16(l);
    r = clamp16(r);
    *out++ = (r<<16) | (l & 0xFFFF);
} while (--outFrames);

// 释放Buffer也比较简单
void ServerProxy::releaseBuffer(Buffer* buffer)
{
    // ...(参数检查)
    // 基本跟客户端一样
    if (mIsOut) {
        int32_t front = cblk->u.mStreaming.mFront;
        android_atomic_release_store(stepCount + front, &cblk->u.mStreaming.mFront);
    } else {
        int32_t rear = cblk->u.mStreaming.mRear;
        android_atomic_release_store(stepCount + rear, &cblk->u.mStreaming.mRear);
    }

    // 唤醒客户端
    // ...(各种参数的计算)
    if (!(old & CBLK_FUTEX_WAKE)) {
        (void) syscall(__NR_futex, &cblk->mFutex,
                mClientInServer ? FUTEX_WAKE_PRIVATE : FUTEX_WAKE, 1);
    }

    // ...(清空buffer)
}
```

这里仅仅是数据的交换流程，具体控制在Track里，Track的各种状态都会影响改流程的。
