---
title: AUGraph 音频流播放
date: 2024-03-14 19:41:30
categories:
- 音视频
tags:
- Audio
keywords:
- AUGraph
- AudioUnit
- 音频流播放
description:
images:
---

AUGraph 属于 Core Audio 框架, 用于描述音频单元（Audio Unit）之间的连接关系和信号流. AUGraph 抽象了音频流的处理过程, 它不直接控制 Audio Unit, 而是由基础单元节点 AUNode 管理. 一个 AUNode 对应一个 Audio Unit.

<!-- more -->

### AUGraph 构建
#### 1.创建 AUGraph 实例
```Objective-C
AUGraph audioGraph;
NewAUGraph(&audioGraph);
```

#### 2.创建并添加 AUNode 节点
```Objective-C
/// IO 播放单元
- (AudioComponentDescription)audioUnitDescription
{
    AudioComponentDescription outputUnitDescription;
    bzero(&outputUnitDescription, sizeof(AudioComponentDescription));
    outputUnitDescription.componentType = kAudioUnitType_Output;
    outputUnitDescription.componentSubType = kAudioUnitSubType_RemoteIO;
    outputUnitDescription.componentManufacturer = kAudioUnitManufacturer_Apple;
    outputUnitDescription.componentFlags = 0;
    outputUnitDescription.componentFlagsMask = 0;
    return outputUnitDescription;
}

/// 混合 播放单元
- (AudioComponentDescription)mixUnitDescription
{
    AudioComponentDescription mixerUnitDescription;
    bzero(&mixerUnitDescription, sizeof(AudioComponentDescription));
    mixerUnitDescription.componentType = kAudioUnitType_Mixer;
    mixerUnitDescription.componentSubType = kAudioUnitSubType_MultiChannelMixer;
    mixerUnitDescription.componentManufacturer = kAudioUnitManufacturer_Apple;
    mixerUnitDescription.componentFlags = 0;
    mixerUnitDescription.componentFlagsMask = 0;
    return mixerUnitDescription;
}

AUNode audioNode;
AudioComponentDescription audioUnitDescription = [self audioUnitDescription];
AUGraphAddNode(audioGraph, &audioUnitDescription, &audioNode);

AUNode mixNode;
AudioComponentDescription mixUnitDescription = [self mixUnitDescription];
AUGraphAddNode(audioGraph, &mixUnitDescription, &mixNode);
```

#### 3.打开 AUGraph
```Objective-C
AUGraphOpen(audioGraph)
```
#### 4.初始化控制单元
```Objective-C
AudioUnit audioUnit;
AudioUnit mixUnit;
AUGraphNodeInfo(audioGraph, audioNode, audioUnitDescription, audioUnit);
AUGraphNodeInfo(audioGraph, mixNode, mixUnitDescription, mixUnit);
```

#### 5.AUNode 节点连接
```
AUGraphConnectNodeInput(audioGraph, mixNode, 0, audioNode, 0);
```

#### 6.设定输入源
```Objective-C
OSStatus status = noErr;
/// 设置音频输出格式描述
AudioStreamBasicDescription destFormat = basicDescription;
status = AudioUnitSetProperty(mixUnit, kAudioUnitProperty_StreamFormat, kAudioUnitScope_Output, 0, &destFormat, sizeof(destFormat));
status = AudioUnitSetProperty(mixUnit, kAudioUnitProperty_StreamFormat, kAudioUnitScope_Input, 0, &destFormat, sizeof(destFormat));

/// 单元输出设置
UInt32 maxFPS = 4096;
status = AudioUnitSetProperty(mixUnit, kAudioUnitProperty_MaximumFramesPerSlice, kAudioUnitScope_Global, 0,&maxFPS, sizeof(maxFPS));
status = AudioUnitAddPropertyListener(mixUnit, kAudioOutputUnitProperty_IsRunning, AudioUnitPropertyListenerProc, (__bridge void *)(self));

/// 渲染音频
AURenderCallbackStruct callbackStruct;
callbackStruct.inputProcRefCon = (__bridge void *)(self);
callbackStruct.inputProc = AURenderCallback;
status = AUGraphSetNodeInputCallback(audioGraph, mixNode, 0, &callbackStruct);
```

#### 7.初始化 AUGraph
```Objective-C
AUGraphInitialize(audioGraph);
```

### AUGraph 播放
#### 1.AudioUnitPropertyListenerProc
```Objective-C
void AudioUnitPropertyListenerProc(void *inRefCon, AudioUnit inUnit, AudioUnitPropertyID inID, AudioUnitScope inScope, AudioUnitElement inElement)
{
    ...
}
```
- inRefCon: 上下文对象.
- inUnit: 监听的 AudioUnit.
- inID: 监听的属性
- inScope: 属性所属的范围，例如全局范围、输入范围或输出范围等.
- inElement: 属性所属的元素，用于指定属性作用的具体元素，如输入通道、输出通道等.

**AudioUnitPropertyListenerProc**是一个回调函数，用于在 Audio Unit 属性发生变化时接收通知。当注册了该回调函数后，当指定的音频单元属性发生更改时，系统会调用这个回调函数来通知应用程序。

#### 2.AURenderCallback
```Objective-C
static OSStatus AURenderCallback(void *userData, AudioUnitRenderActionFlags *ioActionFlags, const AudioTimeStamp *inTimeStamp, UInt32 inBusNumber, UInt32 inNumberFrames, AudioBufferList *ioData)
{
    ...
}
```
- userData: 上下文对象.
- ioActionFlags: 一个指向标志位的指针，用于指示音频单元的操作状态。
- inTimeStamp: 包含有关当前音频帧时间信息的结构体。
- inBusNumber: 指定了音频单元的总线号码，用于确定哪个输入或输出总线正在进行处理。
- inNumberFrames: 指定要处理的音频帧数。
- ioData: 一个指向音频缓冲区列表的指针，包含了输入和输出音频数据。

**AURenderCallback** 是音频渲染回调函数, 在此我们需要提供音频数据给 ioData 才能进行播放.

#### 3.提供音频数据给 ioData
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
AudioConverterFillComplexBuffer(_audioConverter, input_data_proc, (__bridge void *)(self), &inNumberFrames, ioData, NULL);

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
```Objective-C
AUGraphStart(audioGraph);
AudioOutputUnitStart(audioUnit);
```

#### 5.停止播放
```Objective-C
OSStatus status = AUGraphIsRunning(audioGraph, &isRunning);
if (status != noErr) {
    return;
}
    
if (isRunning) {
    status = AUGraphStop(audioGraph);
    if (status != noErr) {
        return;
    }
}
```
