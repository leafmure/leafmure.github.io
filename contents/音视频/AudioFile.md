---
title: AudioFile
date: 2024-03-13 19:40:30
categories:
- 音视频
tags:
- Audio
keywords:
- AudioFile
- 音频播放
description:
images:
---

### 概述
AudioFile 归属于 AudioToolBox 框架, 可以用于读取音频信息和分离音频帧, 与 AudioFileStream 功能类似, 但功能会相对强大. 它不仅可以读取并解析音频, 还可以写入音频. 但是它在解析音频时, 需要音频文件的完整性, 适合于本地音频文件, 不适合用于流播.
<!-- more -->
### AudioFile 初始化
#### AudioFileOpenURL
```Objective-C
typedef CF_ENUM(SInt8, AudioFilePermissions) {
    kAudioFileReadPermission      = 0x01,
    kAudioFileWritePermission     = 0x02,
    kAudioFileReadWritePermission = 0x03
};

extern OSStatus 
AudioFileOpenURL (  CFURLRef                            inFileRef,
                    AudioFilePermissions                inPermissions,
                    AudioFileTypeID                     inFileTypeHint,
                    AudioFileID __nullable * __nonnull  outAudioFile);
```
- inFileRef: 传入 **CFURLRef** url, 只支持本地文件.
- inPermissions: 枚举 **AudioFilePermissions** 授予读取权限.
- inFileTypeHint: **AudioFileTypeID** 音频建议类型, 不知道类型传 0.
- outAudioFile: **AudioFileID** AudioFile 实例 ID.

#### AudioFileOpenWithCallbacks
```Objective-C
AudioFileOpenWithCallbacks (
                void *                              inClientData,
                AudioFile_ReadProc                  inReadFunc,
                AudioFile_WriteProc __nullable      inWriteFunc,
                AudioFile_GetSizeProc               inGetSizeFunc,
                AudioFile_SetSizeProc __nullable    inSetSizeFunc,
                AudioFileTypeID                     inFileTypeHint,
                AudioFileID __nullable * __nonnull  outAudioFile);
```
- inClientData: 上下文对象, 回调函数中传入.
- inReadFunc: **AudioFile_ReadProc**, 读取音频数据时回调, 需给予数据返回.
- inWriteFunc: **AudioFile_WriteProc** 写入音频数据时回调, 需给予数据返回.
- inGetSizeFunc: **AudioFile_GetSizeProc** 获取音频文件长度时触发, 需给予返回文件长度.
- inSetSizeFunc: **AudioFile_SetSizeProc** 设置音频文件长度时触发, 需给予返回文件长度.
- inFileTypeHint: **AudioFileTypeID** 音频建议类型, 不知道类型传 0.
- outAudioFile: **AudioFileID** AudioFile 实例 ID.

**AudioFileOpenWithCallbacks** 创建方式需要先准备好音频文件数据, 在回调函数中返回 AudioFile 所需要的数据, 否则会创建失败. 创建的所需的数据是包含音频格式信息的那部分, 即 **AudioFileStream** 返回 **kAudioFileStreamProperty_ReadyToProducePackets** 时的长度. 所以如果 AudioFile 要读取网络音频流, 需 **AudioFileStream** 配合使用.

##### AudioFile_ReadProc
```Objective-C
typedef OSStatus (*AudioFile_ReadProc)(
                                void *      inClientData,
                                SInt64      inPosition, 
                                UInt32      requestCount,
                                void *      buffer, 
                                UInt32 *    actualCount);
```
- inClientData: 上下文对象.
- inPosition: AudioFile 所需数据在文件中的开始字节位置.
- requestCount: AudioFile 所需数据的字节长度.
- buffer: 通过 memcpy 函数, 给 AudioFile 返回所需的数据.
- actualCount: 返回给 AudioFile 数据的真实长度, 和 buffer 的长度应一致.

OSStatus 返回值一般都传 noError, 如果返回异常值, AudioFile 需要重新打开才能使用. 无异常情况, 没有数据可返回时, 也应返回 noError, actualCount 返回 0.

在读取网络音频流, 缓冲数据不够给 AudioFile, 只给了一部分的情况下, AudioFile 会表现异常, 无法继续正常工作, 此时需要重新创建打开.
<font color='red'>注意: AudioFile 不适用于网络音频流播放</font>

##### AudioFile_GetSizeProc
```Objective-C
typedef SInt64 (*AudioFile_GetSizeProc)(
                                void *      inClientData);
```
- inClientData: 上下文对象.
- 返回 SInt64 值, 音频文件字节长度.

##### AudioFile_WriteProc
```Objective-C
typedef OSStatus (*AudioFile_WriteProc)(
                                void *      inClientData,
                                SInt64      inPosition, 
                                UInt32      requestCount, 
                                const void *buffer, 
                                UInt32    * actualCount);
```
- inClientData: 上下文对象.
- inPosition: AudioFile 写入数据在文件中的开始字节位置.
- requestCount: AudioFile 写入数据的字节长度.
- buffer: 通过 memcpy 函数, 给 AudioFile 返回写入数据.
- actualCount: 返回给 AudioFile 写入数据的真实长度, 和 buffer 的长度应一致.

##### AudioFile_GetSizeProc
```Objective-C
typedef OSStatus (*AudioFile_SetSizeProc)(
                                void *      inClientData,
                                SInt64      inSize);
```
- inClientData: 上下文对象.
- inSize: 写入数据字节长度

### AudioFile 读取音频信息
```Objective-C
extern OSStatus
AudioFileGetPropertyInfo(       AudioFileID             inAudioFile,
                                AudioFilePropertyID     inPropertyID,
                                UInt32 * __nullable     outDataSize,
                                UInt32 * __nullable     isWritable);
```
- inAudioFile: 传入 AudioFile ID.
- inPropertyID: 传入要读取的属性类型.
- outDataSize: 获取属性值的数据大小.
- isWritable: 获取属性是否支持写入.

```Objective-C
extern OSStatus
AudioFileGetProperty(   AudioFileID             inAudioFile,
                        AudioFilePropertyID     inPropertyID,
                        UInt32                  *ioDataSize,
                        void                    *outPropertyData);
```
- inAudioFile: 传入 AudioFile ID.
- nPropertyID: 传入要读取的属性类型.

#### AudioFilePropertyID
```Objective-C
CF_ENUM(AudioFilePropertyID)
{
    kAudioFilePropertyFileFormat            =   'ffmt',
    kAudioFilePropertyDataFormat            =   'dfmt',
    kAudioFilePropertyIsOptimized           =   'optm',
    kAudioFilePropertyMagicCookieData       =   'mgic',
    kAudioFilePropertyAudioDataByteCount    =   'bcnt',
    kAudioFilePropertyAudioDataPacketCount  =   'pcnt',
    kAudioFilePropertyMaximumPacketSize     =   'psze',
    kAudioFilePropertyDataOffset            =   'doff',
    kAudioFilePropertyChannelLayout         =   'cmap',
    kAudioFilePropertyDeferSizeUpdates      =   'dszu',
    kAudioFilePropertyDataFormatName        =   'fnme',
    kAudioFilePropertyMarkerList            =   'mkls',
    kAudioFilePropertyRegionList            =   'rgls',
    kAudioFilePropertyPacketToFrame         =   'pkfr',
    kAudioFilePropertyFrameToPacket         =   'frpk',
    kAudioFilePropertyRestrictsRandomAccess =   'rrap',
    kAudioFilePropertyPacketToRollDistance  =   'pkrl',
    kAudioFilePropertyPreviousIndependentPacket = 'pind',
    kAudioFilePropertyNextIndependentPacket =   'nind',
    kAudioFilePropertyPacketToDependencyInfo =  'pkdp',
    kAudioFilePropertyPacketToByte          =   'pkby',
    kAudioFilePropertyByteToPacket          =   'bypk',
    kAudioFilePropertyChunkIDs              =   'chid',
    kAudioFilePropertyInfoDictionary        =   'info',
    kAudioFilePropertyPacketTableInfo       =   'pnfo',
    kAudioFilePropertyFormatList            =   'flst',
    kAudioFilePropertyPacketSizeUpperBound  =   'pkub',
    kAudioFilePropertyPacketRangeByteCountUpperBound = 'prub',
    kAudioFilePropertyReserveDuration       =   'rsrv',
    kAudioFilePropertyEstimatedDuration     =   'edur',
    kAudioFilePropertyBitRate               =   'brat',
    kAudioFilePropertyID3Tag                =   'id3t',
    kAudioFilePropertyID3TagOffset          =   'id3o',
    kAudioFilePropertySourceBitDepth        =   'sbtd',
    kAudioFilePropertyAlbumArtwork          =   'aart',
    kAudioFilePropertyAudioTrackCount       =   'atct',
    kAudioFilePropertyUseAudioTrack         =   'uatk'
};
```
##### kAudioFilePropertyDataFormat
基础文件格式, 这个属性用于获取音频流的数据格式. 通过查询这个属性, 你可以获得有关音频流的基本信息, 如采样率、声道数、编码格式等.
```Objective-C
UInt32 size = sizeof(AudioStreamBasicDescription);
AudioStreamBasicDescription baseFormat;
AudioFileGetProperty(_fileID, kAudioFilePropertyDataFormat, &size, &baseFormat);
```

##### kAudioFilePropertyFormatList
这个属性用于获取音频流支持的所有格式列表. 当一个音频流支持多种格式时, 你可以使用这个属性来获取所有支持的格式列表.
```Objective-C
UInt32 size;
OSStatus status;

status = AudioFileGetPropertyInfo(_fileID, kAudioFilePropertyFormatList, &size, NULL);
if (status != noErr)
{
  return NO;
}

UInt32 numFormats = size / sizeof(AudioFormatListItem);
AudioFormatListItem *formatList = (AudioFormatListItem *)malloc(size);
status = AudioFileGetProperty(_fileID, kAudioFilePropertyFormatList, &size, formatList);
if (status != noErr)
{
    free(formatList);
    return NO;
}

if (numFormats == 1)
{
    fileFormat = formatList[0].mASBD;
}
else
{
    status = AudioFormatGetPropertyInfo(kAudioFormatProperty_DecodeFormatIDs, 0, NULL, &size);
    if (status != noErr) {
      free(formatList);
      return NO;
    }

    UInt32 numDecoders = size / sizeof(OSType);
    OSType *decoderIDS = (OSType *)malloc(size);

    status = AudioFormatGetProperty(kAudioFormatProperty_DecodeFormatIDs, 0, NULL, &size, decoderIDS);
    if (status != noErr) {
      free(formatList);
      free(decoderIDS);
      return NO;
    }

    UInt32 i;
    for (i = 0; i < numFormats; ++i) {
      OSType decoderID = formatList[i].mASBD.mFormatID;

      BOOL found = NO;
      for (UInt32 j = 0; j < numDecoders; ++j) {
        if (decoderID == decoderIDS[j]) {
          found = YES;
          break;
        }
      }

      if (found) {
        break;
      }
    }

    free(decoderIDS);

    if (i >= numFormats)
    {
      free(formatList);
      return NO;
    }

    fileFormat = formatList[i].mASBD;
}

free(formatList);
```

##### kAudioFilePropertyBitRate
获取音频数据的码率, 以每秒传输的比特数（bps）为单位.
```Objective-C
UInt32 bitRate = 0;
UInt32 size = sizeof(bitRate);
AudioFileGetProperty(_fileID, kAudioFilePropertyBitRate, &size, &bitRate);
```

##### kAudioFilePropertyDataOffset
获取音频内容在音频流中开始的位置
```Objective-C
SInt64 dataOffset = 0;
UInt32 size = sizeof(dataOffset);
AudioFileGetProperty(_fileID, kAudioFilePropertyDataOffset, &size, &dataOffset);
```

##### kAudioFilePropertyAudioDataByteCount
获取音频内容数据长度
```Objective-C
SInt64 audioDataByteCount = 0;
UInt32 size = sizeof(audioDataByteCount);
AudioFileGetProperty(_fileID, kAudioFilePropertyAudioDataByteCount, &size, &audioDataByteCount);
```

##### kAudioFilePropertyEstimatedDuration
获取音频预估时长, 不一定准确.
```Objective-C
Float64 estimatedDuration = 0.0;
UInt32 size = sizeof(estimatedDuration);
AudioFileGetProperty(_fileID, kAudioFilePropertyEstimatedDuration, &size, &estimatedDuration);
```

##### kAudioFilePropertyMaximumPacketSize
获取音频文件的最大包大小
```Objective-C
UInt32 realityMaxPacketSize = 0.0;
UInt32 size = sizeof(realityMaxPacketSize);
AudioFileGetProperty(_fileID, kAudioFilePropertyMaximumPacketSize, &size, &realityMaxPacketSize);
```

##### kAudioFilePropertyPacketSizeUpperBound
获取音频文件的最大包大小上限
```Objective-C
UInt32 maxPacketSize = 0.0;
UInt32 size = sizeof(maxPacketSize);
AudioFileGetProperty(_fileID, kAudioFilePropertyPacketSizeUpperBound, &size, &maxPacketSize);
```

##### kAudioFilePropertyAudioDataPacketCount
获取音频帧(packet)总数, 此信息存在误差或不准确.
```Objective-C
SInt64 packetCount = 0.0;
UInt32 size = sizeof(packetCount);
AudioFileGetProperty(_fileID, kAudioFilePropertyAudioDataPacketCount, &size, &packetCount);
```

##### kAudioFilePropertyMagicCookieData
Magic Cookie 一种用于描述特定音频格式或编解码器配置信息的数据块, 解码器需要该信息.
```Objective-C
UInt32 cookieSize;
OSStatus status = AudioFileGetPropertyInfo(_fileID, kAudioFilePropertyMagicCookieData, &cookieSize, NULL);
if (status != noErr)
{
    return nil;
}

void *cookieData = malloc(cookieSize);
status = AudioFileGetProperty(_fileID, kAudioFilePropertyMagicCookieData, &cookieSize, cookieData);
if (status != noErr)
{
    return nil;
}

NSData *cookie = [NSData dataWithBytes:cookieData length:cookieSize];
free(cookieData);
```

### 读取音频数据
#### AudioFileReadBytes
```Objective-C
extern OSStatus 
AudioFileReadBytes (    AudioFileID     inAudioFile,
                        Boolean         inUseCache,
                        SInt64          inStartingByte, 
                        UInt32          *ioNumBytes, 
                        void            *outBuffer) 
```
- inAudioFile: AudioFile 实例 ID.
- inUseCache: 是否需要缓存, 一般传 false.
- inStartingByte: 从第几个字节开始读取.
- ioNumBytes: 需要读取多少字节.
- outBuffer: 读取到的数据内容.

**AudioFileReadBytes** 仅负责读取数据, 不分离音频帧和解码.

#### AudioFileReadPacketData
```Objective-C
extern OSStatus 
AudioFileReadPacketData (   AudioFileID                     inAudioFile, 
                            Boolean                         inUseCache,
                            UInt32 *                        ioNumBytes,
                            AudioStreamPacketDescription * __nullable outPacketDescriptions,
                            SInt64                          inStartingPacket, 
                            UInt32 *                        ioNumPackets,
                            void * __nullable               outBuffer); 
```
- inAudioFile: AudioFile 实例 ID.
- inUseCache: 是否需要缓存, 一般传 false.
- ioNumBytes: 输入时表示需要读取数据的大小, 输出后, 表示实际读取大小, 与 outBuffer 长度一致.
- outPacketDescriptions: **AudioStreamPacketDescription** 音频帧信息.
- inStartingPacket: 读第几个音频帧.
- ioNumPackets: 读取到的音频帧数量.
- outBuffer: 读取到的音频数据, 需要分配帧预估大小 * 帧数量内存空间, 分配的空间大小影响读取音频帧的数量.

#### AudioFileReadPackets
```Objective-C
/// !废弃 API_DEPRECATED("no longer supported", macos(10.2, 10.10), ios(2.0, 8.0))
extern OSStatus 
AudioFileReadPackets (  AudioFileID                     inAudioFile, 
                        Boolean                         inUseCache,
                        UInt32 *                        outNumBytes,
                        AudioStreamPacketDescription * __nullable outPacketDescriptions,
                        SInt64                          inStartingPacket, 
                        UInt32 *                        ioNumPackets,
                        void * __nullable               outBuffer);
```
- inAudioFile: AudioFile 实例 ID.
- inUseCache: 是否需要缓存, 一般传 false.
- outNumBytes: 实际读取的数据长度.
- outPacketDescriptions: **AudioStreamPacketDescription** 音频帧信息.
- ioNumPackets: 读取到的音频帧数量.
- outBuffer: 读取到的音频数据, 需要分配帧预估大小 * 最大帧大小/上界, 分配的空间大小影响读取音频帧的数量.

### AudioFile Seek
AudioFile 可以根据下标去读取音频帧数据, 所以可以先计算需要 Seek 的音频帧下标, 就能简单实现 Seek 操作.

AudioFile 下标读取音频帧是按顺序解析读取的, 因此在得到完整音频文件数据时, Seek 会比较快, 但如果是网络音频的话, 则需要缓冲加载到 Seek 位置音频帧数据时, 才能 Seek 成功, 因此 AudioFile 不适合与网络音频流播.

如果非要用于网络音频流播, 就需配合使用 **AudioFileStream**. **AudioFileReadBytes** 读取音频数据, **AudioFileStream** 分离音频帧. 这需要计算得到音频帧对应的字节位置, CBR 编码和 VBR 编码计算方式不同.

### 关闭AudioFile
关闭AudioFile 使用完毕后需要调用 AudioFileClose 进行关闭.
```Objective-C
extern OSStatus AudioFileClose  (AudioFileID        inAudioFile)
```
