---
title: AudioFileStream
date: 2024-03-12 13:42:30
categories:
- 音视频
tags:
- Audio
keywords:
- AudioFileStream
- Audio File Stream Services
- 音频流播放
description:
images:
---

## 概述

>Audio File Stream Services provides the interface for parsing streamed audio files—in which only a limited window of data is available at a time.
音频文件流服务提供了用于解析流式音频文件的接口, 其中一次只能使用有限的数据窗口. 
<!-- more -->

>Audio file streams, by nature, are not random access. When you request data from a stream, earlier data might no longer be accessible and later data might not yet be available. In addition, the data you obtain (and then provide to a parser) might include partial packets. To parse streamed audio data, then, a parser must remember data from partially satisfied requests, and must be able to wait for the remainder of that data. In other words, a parser must be able to suspend parsing as needed and then resume where it left off.
音频文件流本质上不是随机访问的. 当您从流请求数据时, 较早的数据可能无法再访问, 而较晚的数据可能尚不可用. 此外, 您获取的数据（然后提供给解析器）可能包括部分数据包. 为了解析流式音频数据, 解析器必须记住部分满足的请求中的数据, 并且必须能够等待该数据的其余部分. 换句话说, 解析器必须能够根据需要暂停解析, 然后从中断处恢复. 

>To use a parser, you pass data from a streamed audio file, as you acquire it, to the parser. When the parser has a complete packet of audio data or a complete property, it invokes a callback function. Your callbacks then process the parsed data—such as by playing it or writing it to disk.
要使用解析器, 您可以在获取流式音频文件时将数据传递到解析器. 当解析器具有完整的音频数据包或完整的属性时, 它会调用回调函数. 然后, 您的回调会处理解析后的数据, 例如播放数据或将其写入磁盘. 

引用自 Apple Document: [Audio File Stream Services](https://developer.apple.com/documentation/audiotoolbox/audio_file_stream_services?language=objc)

总的来说, AudioFileStream 可以读取音频流中的音频信息以及分离音频帧, 它支持的音频类型有:
- AIFF
- AIFC
- WAVE
- CAF
- NeXT
- ADTS
- MPEG Audio Layer 3
- AAC
- MPEG-4

## AudioFileStream 初始化
```Objective-C
extern OSStatus AudioFileStreamOpen (void * inClientData,
                                     AudioFileStream_PropertyListenerProc inPropertyListenerProc,
                                     AudioFileStream_PacketsProc inPacketsProc,
                                     AudioFileTypeID inFileTypeHint,
                                     AudioFileStreamID * outAudioFileStream)
```
- inClientData: 上下文对象.

- inPropertyListenerProc: **AudioFileStream_PropertyListenerProc**音频信息回调函数, 每有解析到音频的信息就会回调一次.

- inPacketsProc: **AudioFileStream_PacketsProc** 分离音频帧回调函数, 在解析完每次传入的Data后, 触发回调.

- inFileTypeHint: **AudioFileTypeID** 音频类型建议, AudioFileStream 会根据开发者给出的建议类型解析, 如果不确定类型就传 0.

- outAudioFileStream: **AudioFileStreamID** AudioFileStream 实例标识.

### AudioFileTypeID
```Objective-C
CF_ENUM(AudioFileTypeID) {
        kAudioFileAIFFType              = 'AIFF',
        kAudioFileAIFCType              = 'AIFC',
        kAudioFileWAVEType              = 'WAVE',
        kAudioFileRF64Type              = 'RF64',
        kAudioFileBW64Type              = 'BW64',
        kAudioFileWave64Type            = 'W64f',
        kAudioFileSoundDesigner2Type    = 'Sd2f',
        kAudioFileNextType              = 'NeXT',
        kAudioFileMP3Type               = 'MPG3',   // mpeg layer 3
        kAudioFileMP2Type               = 'MPG2',   // mpeg layer 2
        kAudioFileMP1Type               = 'MPG1',   // mpeg layer 1
        kAudioFileAC3Type               = 'ac-3',
        kAudioFileAAC_ADTSType          = 'adts',
        kAudioFileMPEG4Type             = 'mp4f',
        kAudioFileM4AType               = 'm4af',
        kAudioFileM4BType               = 'm4bf',
        kAudioFileCAFType               = 'caff',
        kAudioFile3GPType               = '3gpp',
        kAudioFile3GP2Type              = '3gp2',       
        kAudioFileAMRType               = 'amrf',
        kAudioFileFLACType              = 'flac',
        kAudioFileLATMInLOASType        = 'loas'
};
```

## 解析音频流数据
```Objective-C
extern OSStatus AudioFileStreamParseBytes(AudioFileStreamID inAudioFileStream,
                                          UInt32 inDataByteSize,
                                          const void* inData,
                                          UInt32 inFlags);
```
- inAudioFileStream: 传入初始化得到的 AudioFileStream 实例标识.

- inDataByteSize: 本次解析的数据长度.

- inData: 本次解析的数据.

- inFlags: 本次的解析和上一次解析是否是连续的关系, 如果是连续的传入0, 否则传入kAudioFileStreamParseFlag_Discontinuity. 解析以帧为单位, 解码前我们无法得知每个帧具体所处文件流字节位置, 因此在按帧单位解析完后, 会有数据剩余, 在接收的下一次连续数据后, 我们需要将上次剩余数据拼接继续解析, 这样才能保证完整解析. 但如 Seek 操作, Seek 后接收的数据和上一次剩余数据是不连续的, 所以我们传入 kAudioFileStreamParseFlag_Discontinuity 丢弃掉上一次剩余数据.

**AudioFileStreamParseBytes** 方法会返回 OSStatus, 解析成功返回 noErr, 以下是相关操作的错误码:
```Objective-C
CF_ENUM(OSStatus)
{
    /// 不支持该文件类型
    kAudioFileStreamError_UnsupportedFileType       = 'typ?',
    /// 此文件类型不支持数据格式
    kAudioFileStreamError_UnsupportedDataFormat     = 'fmt?',
    /// 不支持该属性
    kAudioFileStreamError_UnsupportedProperty       = 'pty?',
    /// 属性数据的大小不正确. 
    kAudioFileStreamError_BadPropertySize           = '!siz',
    /// 由于文件的数据包表或其他定义, 信息要么不存在, 要么在音频数据之后. 总的来说这个文件需要全部下载完才能播放, 无法流播. 
    kAudioFileStreamError_NotOptimized              = 'optm',
    /// 数据包偏移量小于零或超过文件末尾, 或者在构建数据包表时读取损坏的数据包大小. 
    kAudioFileStreamError_InvalidPacketOffset       = 'pck?',
    /// 该文件格式不正确, 或者不是其类型的音频文件的有效实例, 或者未被识别为音频文件. 
    kAudioFileStreamError_InvalidFile               = 'dta?',
    /// 在音频数据之前, 此文件中不存在属性值. 
    kAudioFileStreamError_ValueUnknown              = 'unk?',
    /// 提供给分析器的数据量不足以产生任何结果. 
    kAudioFileStreamError_DataUnavailable           = 'more',
    /// 试图进行非法操作. 
    kAudioFileStreamError_IllegalOperation          = 'nope',
    /// 发生了未指定的错误. 
    kAudioFileStreamError_UnspecifiedError          = 'wht?',
    /// 不连续性无法恢复
    kAudioFileStreamError_DiscontinuityCantRecover  = 'dsc!'
};
```

## 解析音频信息
### AudioFileStream_PropertyListenerProc
在调用 **AudioFileStreamParseBytes** 后, 首先会去解析音频流中的音频属性信息. 解析到后触发 **AudioFileStream_PropertyListenerProc** 回调.

```Objective-C
typedef void (*AudioFileStream_PropertyListenerProc)(void * inClientData,
                                                     AudioFileStreamID inAudioFileStream,
                                                     AudioFileStreamPropertyID inPropertyID,
                                                     UInt32 * ioFlags);
```
- inClientData: 上下文对象.
- inAudioFileStream: AudioFileStream 实例标识.
- inPropertyID: **AudioFileStreamPropertyID** 属性信息类型.
- ioFlags: 返回参数, 表示这个property是否需要被缓存, 需要缓存则赋值 kAudioFileStreamPropertyFlag_PropertyIsCached, 一般不用赋值处理.

**AudioFileStream_PropertyListenerProc** 会多次触发, 可以根据回调的 **AudioFileStreamPropertyID** 去读取对应信息.

### AudioFileStreamPropertyID
```Objective-C
CF_ENUM(AudioFileStreamPropertyID)
{
    kAudioFileStreamProperty_ReadyToProducePackets          =   'redy',
    kAudioFileStreamProperty_FileFormat                     =   'ffmt',
    kAudioFileStreamProperty_DataFormat                     =   'dfmt',
    kAudioFileStreamProperty_FormatList                     =   'flst',
    kAudioFileStreamProperty_MagicCookieData                =   'mgic',
    kAudioFileStreamProperty_AudioDataByteCount             =   'bcnt',
    kAudioFileStreamProperty_AudioDataPacketCount           =   'pcnt',
    kAudioFileStreamProperty_MaximumPacketSize              =   'psze',
    kAudioFileStreamProperty_DataOffset                     =   'doff',
    kAudioFileStreamProperty_ChannelLayout                  =   'cmap',
    kAudioFileStreamProperty_PacketToFrame                  =   'pkfr',
    kAudioFileStreamProperty_FrameToPacket                  =   'frpk',
    kAudioFileStreamProperty_RestrictsRandomAccess          =   'rrap',
    kAudioFileStreamProperty_PacketToRollDistance           =   'pkrl',
    kAudioFileStreamProperty_PreviousIndependentPacket      =   'pind',
    kAudioFileStreamProperty_NextIndependentPacket          =   'nind',
    kAudioFileStreamProperty_PacketToDependencyInfo         =   'pkdp',
    kAudioFileStreamProperty_PacketToByte                   =   'pkby',
    kAudioFileStreamProperty_ByteToPacket                   =   'bypk',
    kAudioFileStreamProperty_PacketTableInfo                =   'pnfo',
    kAudioFileStreamProperty_PacketSizeUpperBound           =   'pkub',
    kAudioFileStreamProperty_AverageBytesPerPacket          =   'abpp',
    kAudioFileStreamProperty_BitRate                        =   'brat',
    kAudioFileStreamProperty_InfoDictionary                 =   'info'
};
```
#### kAudioFileStreamProperty_ReadyToProducePackets
表示音频信息解析完成, 开始分离音频帧. 

#### kAudioFileStreamProperty_DataOffset
获取音频内容在音频流中开始的位置
```Objective-C
SInt64 dataOffset
UInt32 offsetSize = sizeof(dataOffset);
AudioFileStreamGetProperty(_audioFileStreamID, kAudioFileStreamProperty_DataOffset, &offsetSize, &dataOffset);
```

#### kAudioFileStreamProperty_AudioDataByteCount
获取音频流中音频内容数据长度
```Objective-C
SInt64 audioDataByteCount = 0;
UInt32 sizeOfUInt32 = sizeof(audioDataByteCount);
AudioFileStreamGetProperty(_audioFileStreamID, kAudioFileStreamProperty_AudioDataByteCount, &sizeOfUInt32, &audioDataByteCount);
```
如获取不到, 可以通过 fileSize(音频文件总长度) 和 **kAudioFileStreamProperty_DataOffset** 计算获得.
```Objective-C
SInt64 audioDataByteCount = fileSize - dataOffset;
```


#### kAudioFileStreamProperty_BitRate
获取音频数据的码率, 获取这个malProperty 是为了计算音频的总时长 Duration. 
```Objective-C
/// 这是 CBR 固定码率音频的计算方式, VBR 需解析 Xing 头获取总帧数, 总帧数 * 帧时长.
estimatedDuration = (audioDataByteCount * 8.0) / bitRate * 1000
```

BitRate 可能获取不到或者为 0, 那么就需要根据解析的音频帧进行计算平均码率.
```Objective-C
averagePacketSize = packetsSizeTotal / packetsCount;
packetDuration = fileFormat.mFramesPerPacket / fileFormat.mSampleRate;
averageBitRate =  8.0 * averagePacketSize / packetDuration;
```

#### kAudioFileStreamProperty_DataFormat
基础文件格式, 这个属性用于获取音频流的数据格式. 通过查询这个属性, 你可以获得有关音频流的基本信息, 如采样率、声道数、编码格式等.
```Objective-C
AudioStreamBasicDescription baseFormat;
UInt32 asbdSize = sizeof(baseFormat);
AudioFileStreamGetProperty(_audioFileStreamID, kAudioFileStreamProperty_DataFormat, &asbdSize, &baseFormat);
```

#### kAudioFileStreamProperty_FormatList
这个属性用于获取音频流支持的所有格式列表. 当一个音频流支持多种格式时, 你可以使用这个属性来获取所有支持的格式列表. 
```Objective-C
Boolean outWriteable;
UInt32 formatListSize;
OSStatus status = AudioFileStreamGetPropertyInfo(_audioFileStreamID, kAudioFileStreamProperty_FormatList, &formatListSize, &outWriteable);
if (status == noErr)
{
    AudioFormatListItem *formatList = malloc(formatListSize);
    OSStatus status = AudioFileStreamGetProperty(_audioFileStreamID, kAudioFileStreamProperty_FormatList, &formatListSize, formatList);
    if (status == noErr)
    {
        UInt32 supportedFormatsSize;
        status = AudioFormatGetPropertyInfo(kAudioFormatProperty_DecodeFormatIDs, 0, NULL, &supportedFormatsSize);
        if (status != noErr)
        {
            free(formatList);
            return;
        }
        
        UInt32 supportedFormatCount = supportedFormatsSize / sizeof(OSType);
        OSType *supportedFormats = (OSType *)malloc(supportedFormatsSize);
        status = AudioFormatGetProperty(kAudioFormatProperty_DecodeFormatIDs, 0, NULL, &supportedFormatsSize, supportedFormats);
        if (status != noErr)
        {
            free(formatList);
            free(supportedFormats);
            return;
        }
        
        for (int i = 0; i * sizeof(AudioFormatListItem) < formatListSize; i += sizeof(AudioFormatListItem))
        {
            AudioStreamBasicDescription format = formatList[i].mASBD;
            for (UInt32 j = 0; j < supportedFormatCount; ++j)
            {
                if (format.mFormatID == supportedFormats[j])
                {
                    /// 支持的 format
                    break;
                }
            }
        }
        free(supportedFormats);
    }
    free(formatList);
}
```

#### kAudioFileStreamProperty_AudioDataPacketCount
获取音频帧(packet)总数, 此信息存在误差或不准确.
```Objective-C
SInt64 packetCountSize;
UInt32 sizeOfUInt32 = sizeof(packetCountSize);
AudioFileStreamGetProperty(_audioFileStreamID, kAudioFileStreamProperty_AudioDataPacketCount, &sizeOfUInt32, &packetCountSize);
```

#### kAudioFileStreamProperty_AverageBytesPerPacket
用于获取音频流中每个数据包（packet）的平均字节数. 在处理音频文件或流时, 这个属性可以提供有关音频数据包大小的信息, 有助于解析和处理音频数据流.
```Objective-C
SInt64 packetCountSize;
UInt32 sizeOfUInt32 = sizeof(packetCountSize);
AudioFileStreamGetProperty(_audioFileStreamID, kAudioFileStreamProperty_AverageBytesPerPacket, &sizeOfUInt32, &packetCountSize);
```

#### kAudioFileStreamProperty_MaximumPacketSize
获取最大音频帧最大值, 比较贴近实际大小.
```Objective-C
UInt32 realityMaxPacketSize;
UInt32 sizeOfUInt32 = sizeof(realityMaxPacketSize);
AudioFileStreamGetProperty(_audioFileStreamID, kAudioFileStreamProperty_MaximumPacketSize, &sizeOfUInt32, &realityMaxPacketSize);
```

#### kAudioFileStreamProperty_PacketSizeUpperBound
获取音频帧最大边界大小, 理论最大.
```Objective-C
UInt32 maxPacketSize;
UInt32 sizeOfUInt32 = sizeof(maxPacketSize);
AudioFileStreamGetProperty(_audioFileStreamID, kAudioFileStreamProperty_PacketSizeUpperBound, &sizeOfUInt32, &maxPacketSize);
```

## 分离音频帧
在接收到 **kAudioFileStreamProperty_ReadyToProducePackets** 后, **AudioFileStreamParseBytes** 开始分离音频帧, 分离完成会触发 **AudioFileStream_PacketsProc** 回调.
```Objective-C
typedef void (*AudioFileStream_PacketsProc)(void * inClientData,
                                            UInt32 numberOfBytes,
                                            UInt32 numberOfPackets,
                                            const void * inInputData,
                                            AudioStreamPacketDescription * inPacketDescriptions);
```
- inClientData: 上下文对象.
- numberOfBytes: 本次解析的数据大小.
- numberOfPackets: 本次分离的音频帧数.
- inInputData: 本次处理的所有数据.
- inPacketDescriptions: 音频帧信息, 记录音频帧在本次数据中的字节范围和字节大小.

```Objective-C
struct  AudioStreamPacketDescription
{
    SInt64  mStartOffset; /// 开始字节位置
    UInt32  mVariableFramesInPacket; /// 多少个数据帧
    UInt32  mDataByteSize; /// 音频帧大小
};
```

处理分离的音频帧:
```Objective-C
/// packets: const void * inInputData
- (void)handleAudioFileStreamPackets:(const void *)packets
                       numberOfBytes:(UInt32)numberOfBytes
                     numberOfPackets:(UInt32)numberOfPackets
                  packetDescriptions:(AudioStreamPacketDescription *)packetDescriptioins
{
    
    if (numberOfBytes == 0 || numberOfPackets == 0)
    {
        return;
    }
    
    BOOL deletePackDesc = NO;
    if (packetDescriptioins == NULL)
    {
        // 如果packetDescriptioins不存在, 就按照CBR处理, 平均每一帧的数据后生成packetDescriptioins
        deletePackDesc = YES;
        UInt32 packetSize = numberOfBytes / numberOfPackets;
        AudioStreamPacketDescription *descriptions = (AudioStreamPacketDescription *)malloc(sizeof(AudioStreamPacketDescription) * numberOfPackets);
        
        for (int i = 0; i < numberOfPackets; i++)
        {
            UInt32 packetOffset = packetSize * i;
            /// 开始位置
            descriptions[i].mStartOffset = packetOffset;
            /// 数据帧数量
            descriptions[i].mVariableFramesInPacket = 0;
            /// 计算数据大小
            if (i == numberOfPackets - 1)
            {
                descriptions[i].mDataByteSize = numberOfBytes - packetOffset;
            }
            else
            {
                descriptions[i].mDataByteSize = packetSize;
            }
        }
        packetDescriptioins = descriptions;
    }

    for (int i = 0; i < numberOfPackets; ++i)
    {
        SInt64 packetOffset = packetDescriptioins[i].mStartOffset;
        NSData *data = [NSData dataWithBytes:packets + packetOffset length:packetDescription.mDataByteSize];
        ...
    }

    if (deletePackDesc)
    {
        free(packetDescriptioins);
    }
}
```

## AudioFileStreamSeek
**AudioFileStreamSeek** 可以用来寻找某一个帧（Packet）对应的字节偏移（byte offset）
```Objective-C
extern OSStatus
AudioFileStreamSeek(    
                AudioFileStreamID inAudioFileStream,
                SInt64  inPacketOffset,
                SInt64  *outDataByteOffset,
                AudioFileStreamSeekFlags *ioFlags);
```
- inPacketOffset: 音频文件中第几个音频帧.
- outDataByteOffset: 输出音频帧在文件中的字节位置.
- ioFlags:  如果是 **kAudioFileStreamSeekFlag_OffsetIsEstimated** 值表明获取的 outDataByteOffset 值是预估的并不准确, 反则, 获得是准确的值.

## 关闭AudioFileStream
AudioFileStream 使用完毕后需要调用 **AudioFileStreamClose** 进行关闭.
```Objective-C
extern OSStatus AudioFileStreamClose(AudioFileStreamID inAudioFileStream); 
```

