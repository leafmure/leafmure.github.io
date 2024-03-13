---
title: CBR、VBR音频 SeekTime
date: 2024-03-12 19:50:30
categories:
- 音视频
tags:
- Audio
keywords:
- CBR
- VBR
- SeekTime
description:
images:
---

音频 Seek Time 功能需要我们在音频文件中找到对应播放时间点的字节位置, 这样才能实现播放时间点的跳跃. 

音频存在 CBR、VBR、CVBR 等码率编码方式, 影响每一帧的数据长度大小. 这意味着对于不同码率编码的音频, 实现 Seek Time 的方式也不一样, 这里主要针对 CBR 和 VBR.
<!-- more -->
### CBR
CBR代表恒定比特率（Constant Bit Rate）, 其中音频数据以恒定的比特率进行传输或存储. 在CBR编码中, 每秒传输的比特数保持不变, 这意味着无论音频内容复杂度如何, 文件大小也会保持一致. CBR 的音频每一帧的大小和时长都保持一致, 对于 CBR 的 Seek Time 实现也就比较容易.

由于 CBR 音频每帧大小一致, 我们可以近似的算出时间点对应的字节位置:
```Objective-C
/// dataOffset  kAudioFileStreamProperty_DataOffset
/// audioDataByteCount  kAudioFileStreamProperty_AudioDataByteCount
NSTimeInterval estimatedDuration = (audioDataByteCount * 8.0) / bitRate * 1000
NSUInteger byteSeekOffset = dataOffset + seekTime / (estimatedDuration / 1000.0) * audioDataByteCount;
```
通过 seekTime / packetDuration 可以获取是第几个音频帧, 然后通过 **AudioFileStreamSeek** 方法可以获取音频帧所在的字节位置.
```Objective-C
NSTimeInterval packetDuration = fileFormat.mFramesPerPacket / fileFormat.mSampleRate;
SInt64 seekToPacket = floor(seekTime / packetDuration);
UInt32 ioFlags = 0;
SInt64 outDataByteOffset;
OSStatus status = AudioFileStreamSeek(_audioFileStreamID, seekToPacket, &outDataByteOffset, &ioFlags);
```
如果 ioFlags 包含 **kAudioFileStreamSeekFlag_OffsetIsEstimated**, 说明获得的字节位置是不准确的, 因此我们直接使用 **byteSeekOffset**, 反之, 我们可以使用 **AudioFileStreamSeek** 获得的 **outDataByteOffset**, 同时, 我们需要修正一下 SeekTime.
```Objective-C
if (status == noErr && !(ioFlags & kAudioFileStreamSeekFlag_OffsetIsEstimated))
{
    position = outDataByteOffset + dataOffset;
    seekTime = packetDuration * seekToPacket;
}
else
{
    position = byteSeekOffset;
}
```

### VBR
VBR代表可变比特率（Variable Bit Rate）, 其中音频数据以根据内容复杂度而变化的比特率进行传输或存储. 在VBR编码中, 编码器会动态调整比特率, 以便更有效地表示音频内容, 从而在保持高质量的同时减少文件大小. VBR 每一帧的时长是一样的, 但是数据大小是不一致的, 因此无法采用 CBR 方式去计算时间点对应的字节位置, 以及计算音频文件总时长.

为了解决上述的两个问题, VBR 编码增加了一些数据字段. 字段规范有两种: Xing 规范、VBRI 规范, 由于使用 VBRI 规范的 VBR 编码不常见, 大多数 VBR 编码都是采用 Xing 规范, 因此只针对 Xing 规范实现. 

Xing 规范的主要内容是 Xing 头, VBR 音频的开头第一个音频帧用来存储 Xing 头信息,  不存储音频数据. Xing 头以 “Xing” 或 “Info” 字符作为字段开头的标记, Xing 头距离帧头(一般是 4 字节)之间是一段为0的数据,

#### Xing结构
Xing 头的结构信息如下:

| Xing开始字节 | 字节长度 | 含义 | 示例 |
|-------|-------|-------|-------|
| 0 | 4 | VBR头标记, 4个ASCII字符的VBR头 ID, 要么是Xing, 要么是Info, 无NULL结尾（普通字符串都以NULL,即\0结尾） | 'Xing' |
| 4 | 4 | 存放一个标志, 用于表示接下来存在哪些域/字段,各字段逻辑或的结果：<br>0x0001 - 总帧数存储区域设置为存在, 不包括第一帧.<br>0x0002 - 文件长度存储区域设置为存在, 不包括标签；<br>0x0004 - TOC 索引存储区域设置为存在；<br>0x0008 - 质量指示存储区域设置为存在 | 0x0007 就表示下面存在：<br>0x0001 - 总帧数<br>0x0002 - 文件长度<br>0x0004 - TOC 索引|
| 8 | 4 | 总帧数 Frames | 1912 |
| 8 or 12 | 4 | 文件长度 | 123456 |
| 8 or 12 or 16 | 100 | TOC 表 | |
| 8 or 12 or 16 or 108 or 112 or 116 | 4 | 音频质量指示, 最差0, 最好100 | 0 |

#### 解析 Xing
首先, 我们可以通过 **kAudioFileStreamProperty_DataOffset** 拿到音频文件中内容起始位置, Xing 内容会在 DataOffset 位置前第一帧中, DataOffset 后紧接着就是下一帧, 所以通过 DataOffset + 2 可以获得音频帧头标识.
```Objective-C
NSInteger packetHeadTagLength = 2;
NSData *data = [fileData subdataWithRange:NSMakeRange(0, dataOffset + packetHeadTagLength)];
NSData *firstData = [data subdataWithRange:NSMakeRange(0, dataOffset)]
NSData *packetHeadTag = [data subdataWithRange:NSMakeRange(dataOffset, packetHeadTagLength)]
```
然后, 通过音频帧头标识, 从 DataOffset 位置往前匹配帧头标识, 找到第一帧完整数据.
```Objective-C
[self foundFirstPacket:firstData packetHeadTag:packetHeadTag];

+ (NSData *)foundFirstPacket:(NSData *)data packetHeadTag:(NSData *)packetHeadTag
{
    if (data.length <= packetHeadTag.length) return nil;
    
    NSMutableArray *matchTag = [NSMutableArray array];
    uint8_t *packetHeadTagPtr = (uint8_t *)[packetHeadTag bytes];
    for (NSInteger position = 0; position < packetHeadTag.length; position++) {
        NSInteger v = *(packetHeadTagPtr + position);
        [matchTag insertObject:@(v) atIndex:0];
    }
    
    NSData *packetData = data;
    uint8_t *ptr = (uint8_t *)[packetData bytes];
    NSInteger matchIndex = 0;
    NSInteger matchPostion = 0;
    for (NSInteger position = packetData.length - 1; position >= 0; position--) {
        NSInteger v = *(ptr + position);
        if (matchIndex < matchTag.count) {
            if (v == [matchTag[matchIndex] integerValue])
            {
                matchIndex += 1;
            } else {
                matchIndex = 0;
            }
        } else {
            matchPostion = position;
            break;
        }
    }
    if (matchPostion == 0) {
        return nil;
    } else {
        return [packetData subdataWithRange:NSMakeRange(matchPostion + 1, packetData.length - matchPostion - 1)];
    }
}
```
找到第一帧后, 我们可以开始去寻找 Xing 信息数据.
```Objective-C
NSData *firstPacketData = [self foundFirstPacket:firstData packetHeadTag:packetHeadTag];
uint8_t *ptr = (uint8_t *)[firstPacketData bytes];
NSInteger foundPosition = 4;
for (NSInteger position = 4; position < firstPacketData.length; position++) 
{
    NSInteger v = *(ptr + position);
    if (v == 0) {
        foundPosition += 1;
    } else {
        break;
    }
}
NSData *xingOrInfoData = [firstPacketData subdataWithRange:NSMakeRange(foundPosition, firstPacketData.length - foundPosition)];
```
找到 Xing 信息数据, 就可以开始按结构解析.
```Objective-C
NSData *tagData = [data subdataWithRange:NSMakeRange(0, 4)];
NSString *vbrTag = [[NSString alloc] initWithData:tagData encoding:NSUTF8StringEncoding];

/// xing 解析
if ([vbrTag.lowercaseString isEqualToString:@"xing"] || [vbrTag.lowercaseString isEqualToString:@"info"]) {
    NSData *xingTagData = [data subdataWithRange:NSMakeRange(tagData.length, data.length - tagData.length)];

    NSData *validTagData = [xingTagData subdataWithRange:NSMakeRange(0, 4)];
    NSInteger validTag = [self intValue:validTagData];
    BOOL hasTotalFrameCountTag = NO, hasContentLengthTag = NO, hasTocTag = NO, hasQualityTag = NO;
    if (validTag & 1) {
        hasTotalFrameCountTag = YES;
    }
    
    if (validTag & 2) {
        hasContentLengthTag = YES;
    }
    
    if (validTag & 4) {
        hasTocTag = YES;
    }
    
    if (validTag & 8) {
        hasQualityTag = YES;
    }
    
    NSInteger readOffset = 4;
    if (hasTotalFrameCountTag) {
        NSInteger totalFrameCount = [self intValue:[xingTagData subdataWithRange:NSMakeRange(readOffset, 4)]];
        NSTimeInterval estimatedDuration = (Float64)totalFrameCount * fileFormat.mFramesPerPacket / fileFormat.mSampleRate * 1000;
        readOffset += 4;
    }
    
    if (hasContentLengthTag) {
        NSInteger contentLength = [self intValue:[xingTagData subdataWithRange:NSMakeRange(readOffset, 4)]];
        readOffset += 4;
    }
    
    if (hasTocTag) {
        // 获取 TOC 数据
        NSData *tocData = [xingTagData subdataWithRange:NSMakeRange(readOffset, 100)];
        // 解析 TOC 数据
        NSMutableArray *tocArray = [NSMutableArray array];
        const unsigned char *tocBytes = tocData.bytes;
        for (NSUInteger i = 0; i < tocData.length; i++) {
            unsigned char tocValue = tocBytes[i];
            double percentage = (double)tocValue / 256.0;
            [tocArray addObject:@(percentage)];
        }
    }
}
```
解析完后, 我们主要获得音频总帧数 totalFrameCount 和 TOC数据表, 用于计算音频总时长和实现 SeekTime 功能.
```Objective-C
NSTimeInterval seekTime;
NSTimeInterval packetDuration = fileFormat.mFramesPerPacket / fileFormat.mSampleRate;
NSTimeInterval estimatedDuration = (Float64)totalFrameCount * packetDuration * 1000;
float percent = (time / (estimatedDuration / 1000.0)) * 100;
int index = (int)percent;
position = (NSUInteger)([vbrToc[index] floatValue] * fileSize) + dataOffset;
seekTime = index * packetDuration;
```
得到字节位置 position 后, 就可以进行 Seek 操作了.
