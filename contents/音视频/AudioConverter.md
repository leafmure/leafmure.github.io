---
title: AudioConverter
date: 2024-03-14 18:51:30
categories:
- 音视频
tags:
- Audio
keywords:
- AudioConverter
- 音频解码
description:
images:
---

### 概述
AudioConverter 可以将各种线性 PCM 音频格式之间进行转换, 除此之外, 它还能将 PCM 音频格式转为压缩音频格式, 也能将压缩音频格式转为 PCM 音频格式, 达到解码的效果, 如果要实现压缩音频格式之间的转换, 则可以先转 PCM 音频格式, 再转目标压缩音频格式, 支持的压缩音频格式有:
- MP3
- WAV
- AAC
- FLAC
- OGG
- WMA
- AIFF
- M4A
- ALAC
- AC3

<!-- more -->

### AudioStreamBasicDescription
```Objective-C
struct AudioStreamBasicDescription
{
    Float64             mSampleRate;  音频流的采样率（每秒采样次数）44100.0
    AudioFormatID       mFormatID; 音频数据的格式标识符，例如 kAudioFormatLinearPCM 表示线性 PCM 格式。
    AudioFormatFlags    mFormatFlags;  音频数据的格式标志位，用于指定特定的格式选项和特性。kAudioFormatFlagIsSignedInteger | kAudioFormatFlagIsPacked
    UInt32              mBytesPerPacket; 每个数据包中的字节数。  2
    UInt32              mFramesPerPacket; 每个数据包中的帧数。 1
    UInt32              mBytesPerFrame; 每帧的字节数。 2
    UInt32              mChannelsPerFrame; 每帧的声道数。 1
    UInt32              mBitsPerChannel; 每个声道的比特数。 16
    UInt32              mReserved;  保留字段，未使用。 0
};
```
#### kAudioFormatFlagIsFloat、kAudioFormatFlagIsSignedInteger
- kAudioFormatFlagIsFloat: 浮点型采样数据, 有: Float 16, Float 32.
- kAudioFormatFlagIsSignedInteger: 有符号整数采样数据. 有: Int16, Int32.

在 iOS 设备上，16 位浮点型 PCM 格式是不被直接支持的。iOS 的 Core Audio 
框架中并没有提供原生的支持来处理和播放 16 位浮点型 PCM 音频数据。
iOS 设备的音频引擎通常使用整数表示的 PCM 格式，如 16 位有符号整数（SInt16）或 32 位浮点型（Float32）。

#### kAudioFormatFlagIsNonInterleaved
**kAudioFormatFlagIsNonInterleaved** 是一个用于描述音频数据格式的标志，通常在Core Audio框架中使用。当设置了这个标志时，表示音频数据以非交错（non-interleaved）的方式存储。

在非交错音频数据中，每个声道的样本值被单独存储在各自的缓冲区中，而不是交错存储在同一个缓冲区中。这种表示方式在处理多通道音频数据时很有用，因为它可以更容易地对每个声道进行独立处理。

具体体现在 **AudioBufferList** 对 mBuffers 数量的分配上, **kAudioFormatFlagIsNonInterleaved** 时存在多个缓冲区, mBuffers 数量取决于 **AudioStreamBasicDescription** 的 mChannelsPerFrame. 而交错存储只有一个缓冲区, mBuffers 数量为 1.

#### 

### AudioConverter 初始化
#### 创建 AudioConverterRef 实例
```Objective-C
extern OSStatus
AudioConverterNew(      const AudioStreamBasicDescription * inSourceFormat,
                        const AudioStreamBasicDescription * inDestinationFormat,
                        AudioConverterRef __nullable * __nonnull outAudioConverter);
```
- inSourceFormat: 输入音频格式.
- inDestinationFormat: 需要转换的目标格式.
- outAudioConverter: 创建好的 AudioConverter 实例.

#### AudioConverter 属性设置和读取
```Objective-C
extern OSStatus
AudioConverterSetProperty(  AudioConverterRef           inAudioConverter,
                            AudioConverterPropertyID    inPropertyID,
                            UInt32                      inPropertyDataSize,
                            const void *                inPropertyData);
```
- inAudioConverter: AudioConverter 实例
- inPropertyID: 要设置的属性.
- inPropertyDataSize: 设置值的字节大小.
- inPropertyData: 设置值数据.

```Objective-C
extern OSStatus
AudioConverterGetProperty(  AudioConverterRef           inAudioConverter,
                            AudioConverterPropertyID    inPropertyID,
                            UInt32 *                    ioPropertyDataSize,
                            void *                      outPropertyData); 
```
- inAudioConverter: AudioConverter 实例.
- inPropertyID: 要读取的属性.
- ioPropertyDataSize: 属性值的字节大小.
- outPropertyData: 读取到的属性值.

#### AudioConverterPropertyID
```Objective-C
CF_ENUM(AudioConverterPropertyID)
{
    kAudioConverterPropertyMinimumInputBufferSize       = 'mibs',
    kAudioConverterPropertyMinimumOutputBufferSize      = 'mobs',
    kAudioConverterPropertyMaximumInputPacketSize       = 'xips',
    // 用于指定音频转换器可以生成的最大输出数据包大小。这个属性通常用于配置音频转换器的行为，以确保输出数据包的大小在一定范围内，从而满足特定的需求或限制。
    kAudioConverterPropertyMaximumOutputPacketSize      = 'xops',
    kAudioConverterPropertyCalculateInputBufferSize     = 'cibs',
    kAudioConverterPropertyCalculateOutputBufferSize    = 'cobs',
    kAudioConverterPropertyInputCodecParameters         = 'icdp',
    kAudioConverterPropertyOutputCodecParameters        = 'ocdp',
    kAudioConverterSampleRateConverterComplexity        = 'srca',
    kAudioConverterSampleRateConverterQuality           = 'srcq',
    kAudioConverterSampleRateConverterInitialPhase      = 'srcp',
    kAudioConverterCodecQuality                         = 'cdqu',
    kAudioConverterPrimeMethod                          = 'prmm',
    kAudioConverterPrimeInfo                            = 'prim',
    kAudioConverterChannelMap                           = 'chmp',
    kAudioConverterDecompressionMagicCookie             = 'dmgc',
    kAudioConverterCompressionMagicCookie               = 'cmgc',
    kAudioConverterEncodeBitRate                        = 'brat',
    kAudioConverterEncodeAdjustableSampleRate           = 'ajsr',
    kAudioConverterInputChannelLayout                   = 'icl ',
    kAudioConverterOutputChannelLayout                  = 'ocl ',
    kAudioConverterApplicableEncodeBitRates             = 'aebr',
    kAudioConverterAvailableEncodeBitRates              = 'vebr',
    kAudioConverterApplicableEncodeSampleRates          = 'aesr',
    kAudioConverterAvailableEncodeSampleRates           = 'vesr',
    kAudioConverterAvailableEncodeChannelLayoutTags     = 'aecl',
    kAudioConverterCurrentOutputStreamDescription       = 'acod',
    kAudioConverterCurrentInputStreamDescription        = 'acid',
    kAudioConverterPropertySettings                     = 'acps',
    kAudioConverterPropertyBitDepthHint                 = 'acbd',
    kAudioConverterPropertyFormatList                   = 'flst'
};
```

##### kAudioConverterDecompressionMagicCookie (解码必要配置)
>In the realm of Core Audio, a magic cookie is an opaque set of metadata attached to a compressed sound file or stream. The metadata gives a decoder the details it needs to properly decompress the file or stream. You treat a magic cookie as a black box, relying on Core Audio functions to copy, read, and use the contained metadata.
>在 Core Audio 领域，magic cookie 是附加到压缩声音文件或流的一组不透明元数据。元数据为解码器提供正确解压缩文件或流所需的详细信息。您将 magic cookie 视为黑匣子，依靠 Core Audio 功能来复制、读取和使用所包含的元数据。
>引用 [Apple Document](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/CoreAudioOverview/CoreAudioEssentials/CoreAudioEssentials.html#//apple_ref/doc/uid/TP40003577-CH10-SW17)

部分音频文件存在 magic cookies 信息, 这个信息在解码时需要用到, 所以如果音频存在 magic cookies, 就要设置 **kAudioConverterDecompressionMagicCookie** 属性. 没有得话, 就不用处理.
```Objective-C
NSData *cookieData = ...;
if (cookieData.length > 0) {
    OSStatus status = AudioConverterSetProperty(_audioConverter, kAudioConverterDecompressionMagicCookie, (UInt32)cookieData.length, [cookieData bytes]);
    if (status != noErr) {
        /// 销毁 AudioConverterRef
    }
}
```

##### kAudioConverterCurrentInputStreamDescription
AudioConverterRef 的输入源音频格式, 虽说在创建时传入了音频格式, 但不能保证设置成功, 所以需要再获取一下确认.
```Objective-C
AudioStreamBasicDescription inputFormat;
UInt32 size = sizeof(inputFormat);
AudioConverterGetProperty(_audioConverter, kAudioConverterCurrentInputStreamDescription, &size, &inputFormat);
```

##### kAudioConverterCurrentOutputStreamDescription
AudioConverterRef 的输出源音频格式, 虽说在创建时传入了音频格式, 但不能保证设置成功, 所以需要再获取一下确认.
```Objective-C
AudioStreamBasicDescription outputFormat;
UInt32 size = sizeof(outputFormat);
AudioConverterGetProperty(_audioConverter, kAudioConverterCurrentOutputStreamDescription, &size, &outputFormat);
```

### 转换格式
#### AudioConverterConvertBuffer
```Objective-C
extern OSStatus
AudioConverterConvertBuffer(    AudioConverterRef               inAudioConverter,
                                UInt32                          inInputDataSize,
                                const void *                    inInputData,
                                UInt32 *                        ioOutputDataSize,
                                void *                          outOutputData);
```
- inAudioConverter: AudioConverter 实例.
- inInputDataSize: 输入源音频字节大小.
- inInputData: 输入源音频数据.
- ioOutputDataSize: 输出源音频字节大小.
- outOutputData: 输出源音频数据.

**AudioConverterConvertBuffer** 用于缓冲区对缓冲区的音频转换, 这个转换只支持同样采样率的 PCM -> PCM, 如需 PCM 和 压缩格式的转换请使用 **AudioConverterFillComplexBuffer**.

#### AudioConverterConvertComplexBuffer
```Objective-C
extern OSStatus
AudioConverterConvertComplexBuffer( AudioConverterRef               inAudioConverter,
                                    UInt32                          inNumberPCMFrames,
                                    const AudioBufferList *         inInputData,
                                    AudioBufferList *               outOutputData)
```
- inAudioConverter: AudioConverter 实例.
- inNumberPCMFrames: 输入 PCM 数据帧数量.
- inInputData: 输入源音频数据(**AudioBufferList**).
- outOutputData: 输出源音频数据(**AudioBufferList**)

**AudioConverterConvertBuffer** 用于 AudioBufferList 对 AudioBufferList 的音频转换, 这个转换只支持同样采样率的 PCM -> PCM, 如需 PCM 和 压缩格式的转换请使用 **AudioConverterFillComplexBuffer**.

与 **AudioConverterConvertBuffer** 相比, 它可以处理更复杂的音频转换, 它允许处理多个输入数据包和输出数据包，每个数据包可以具有不同的格式或大小。

#### AudioConverterFillComplexBuffer
```Objective-C
extern OSStatus
AudioConverterFillComplexBuffer(    AudioConverterRef                   inAudioConverter,
                                    AudioConverterComplexInputDataProc  inInputDataProc,
                                    void * __nullable                   inInputDataProcUserData,
                                    UInt32 *                            ioOutputDataPacketSize,
                                    AudioBufferList *                   outOutputData,
                                    AudioStreamPacketDescription * __nullable outPacketDescription);
```
- inAudioConverter: AudioConverter 实例.
- inInputDataProc: **AudioConverterComplexInputDataProc** 提供输入源音频数据的回调函数.
- inInputDataProcUserData: 上下文对象, 将在回调函数中传入.
- ioOutputDataPacketSize: 转换到目标格式时, 输出的音频数据字节大小.
- outOutputData: 转换到目标格式的数据.
- outPacketDescription: 输入时提供一个分配好的 **AudioStreamPacketDescription** 内存块, **AudioConverterComplexInputDataProc** 回调中需要返回输入源音频帧的信息.

#### AudioConverterComplexInputDataProc
```Objective-C
typedef OSStatus
(*AudioConverterComplexInputDataProc)(  AudioConverterRef               inAudioConverter,
                                        UInt32 *                        ioNumberDataPackets,
                                        AudioBufferList *               ioData,
                                        AudioStreamPacketDescription * __nullable * __nullable outDataPacketDescription,
                                        void * __nullable               inUserData);
```
- inAudioConverter: AudioConverter 实例.
- ioNumberDataPackets: 可转换的音频帧数量.
- ioData: 提供输入源音频数据.
- outDataPacketDescription: 不为 NULL 时, 需要提供输入源音频帧信息.
- inUserData: 上下文对象.

示例:
```Objective-C
static OSStatus AudioConverter_input_data_proc(AudioConverterRef inAudioConverter, UInt32 *ioNumberDataPackets, AudioBufferList *ioData, AudioStreamPacketDescription **outDataPacketDescription, void *inUserData)
{
    /// 填充数据到输出 ioData
    /// srcBuffer 音频帧数据
    ioData->mBuffers[0].mData = (void* __nullable)[srcBuffer bytes];
    ioData->mBuffers[0].mDataByteSize = (UInt32)srcBuffer.length;
    ioData->mBuffers[0].mNumberChannels = 1;
    
    if (outDataPacketDescription != NULL) {
        /// 音频帧信息
        *outDataPacketDescription = outputPktDescs;
    }
}
/// 音频帧分离使用: AudioFileStream
```

### 销毁 AudioConverter
AudioConverter 使用完后需及时销毁.
```Objective-C
extern OSStatus
AudioConverterDispose(  AudioConverterRef   inAudioConverter);
```
