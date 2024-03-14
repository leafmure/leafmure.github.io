---
title: AVAudioEngine音频流播放
date: 2024-03-14 20:51:30
categories:
- 音视频
tags:
- Audio
keywords:
- AVAudioEngine
- Audio Engine
- 音频流播放
description:
images:
---

### 概述
Audio Engine 提供了强大、功能丰富的 API 来简化音频生成、处理和输入/输出任务。该引擎包含一组节点，这些节点连接起来形成音频信号处理链。这些节点在渲染到输出目的地之前对信号执行各种任务。Audio Engine 可以提供以下功能实现:
- 使用文件和缓冲区播放音频
- 在处理链中的任意点捕获音频
- 添加内置效果，如混响、延迟、失真和您的自定义效果
- 执行立体声和 3D 混合
- 提供 MIDI 播放和对采样器乐器的控制

引用来自: [Apple Document](https://developer.apple.com/documentation/avfaudio/audio_engine?language=objc)

<!-- more -->

### AVAudioEngine 构建
#### 1.初始化 AVAudioEngine
```Objective-C
_audioEngine = [[AVAudioEngine alloc] init];
```

#### 2.创建并添加 AVAudioPlayerNode
```Objective-C
_playerNode = [[AVAudioPlayerNode alloc] init];
[_audioEngine attachNode:_playerNode];
```

#### 3.连接 AVAudioPlayerNode 到输出节点
```Objective-C
[_audioEngine connect:_playerNode to:_audioEngine.mainMixerNode format:nil];
```

#### 4.AVAudioEngine 准备
```Objective-C
[_audioEngine prepare];
```

### AVAudioEngine 播放
#### 1.创建 AVAudioPCMBuffer
```Objective-C
AudioStreamBasicDescription *fileFormat = ...;
AVAudioFormat *format = [[AVAudioFormat alloc] initWithStreamDescription:&fileFormat];
AVAudioPCMBuffer *pcmBuffer = [[AVAudioPCMBuffer alloc] initWithPCMFormat:format frameCapacity:512];
pcmBuffer.frameLength = pcmBuffer.frameCapacity;
```

#### 2.填充音频数据到 pcmBuffer.mutableAudioBufferList
首先我们需要使用 **AudioFileStream** 对音频数据流进行分离音频帧. 
```Objective-C
AudioFileStreamParseBytes(_audioFileStreamID,(UInt32)[data length],[data bytes], 0);
/// 输出分离的音频帧数据
static void audio_file_stream_packets_proc(void *inClientData,
                                           UInt32 inNumberBytes,
                                           UInt32 inNumberPackets,
                                           const void *inInputData,
                                           AudioStreamPacketDescription    *inPacketDescriptions)
{
    /// inInputData 音频帧数据
    // inNumberPackets 音频帧数量
    // inPacketDescriptions 音频帧信息,包含数据字节起始位置和长度.
    ...
}
```

在得到音频帧后, 使用 **AudioConverter** 将音频帧从压缩格式转为 PCM 格式数据.
```Objective-C
AudioConverterFillComplexBuffer(_audioConverter, input_data_proc, (__bridge void *)(self), &inNumberFrames, pcmBuffer.mutableAudioBufferList, NULL);

static OSStatus input_data_proc(AudioConverterRef inAudioConverter, UInt32 *ioNumberDataPackets, AudioBufferList *ioData, AudioStreamPacketDescription **outDataPacketDescription, void *inUserData)
{
    /// 伪代码
    /// inInputData <- audio_file_stream_packets_proc
    /// inPacketDescriptions <- audio_file_stream_packets_proc
    ioData->mBuffers[0].mData = [inInputData bytes];
    ioData->mBuffers[0].mDataByteSize = [inInputData length];
    ioData->mBuffers[0].mNumberChannels = 1;
    if (outDataPacketDescription != NULL) {
        *outDataPacketDescription = inPacketDescriptions;
    }
    return noErr;
}
```

#### 4.开始播放
创建定时器控制播放速率
```Objective-C
NSTimeInterval packetDuration = (fileFormat.mFramesPerPacket / fileFormat.mSampleRate) * 1000.0;
/// 正常速率
_timmer = [NSTimer timerWithTimeInterval:packetDuration * 0.5 repeats:YES block:^(NSTimer * _Nonnull timer) {
    /// 播放 pcmBuffer
    [_playerNode scheduleBuffer:pcmBuffer completionHandler:^{}];
}];
[[NSRunLoop currentRunLoop] addTimer:_timmer forMode:NSRunLoopCommonModes];
```

启动 AVAudioEngine 和 AVAudioPlayerNode
```Objective-C
if (_playerNode.isPlaying) {
    return;
}
    
if (!_audioEngine.isRunning) {
    NSError *error;
    [_audioEngine startAndReturnError:&error];
    if (error) {
        return;
    }
}
    
[_playerNode play];
[_timmer setFireDate:[NSDate date]];
```

#### 5.停止播放
```Objective-C
[_timmer invalidate];
_timmer = nil;
[_playerNode stop];
[_audioEngine stop];
[_audioEngine disconnectNodeInput:_playerNode];
```