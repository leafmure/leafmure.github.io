---
title: H.264
date: 2023-11-24 18:54:47
categories:
- 音视频
tags:
- H.264
keywords:
- H.264
description:
images:
---

### H.264
H.264码流结构分为两层:
- 网络提取层 (Network Abstraction Layer, NAL)
- 视讯编码层 (Video Coding Layer, VCL)

VCL数据即被压缩编码后的视频数据序列, 需要通过封装到 NAL 单元中之后, 才可以用来传输或存储. H.264码流结构主要由一系列 GOP 构成, 一个 GOP 是一个视频编码序列, 每个 GOP 的第一帧是 IDR 帧, 此外还包含了 SPS 和 PPS 信息, 每个 GOP 都可以独立解码。 一个 GOP 由一系列 I、P、B帧构成, 一个视频帧又可以划分为 Slice（切片）, 一个 Slice 则由宏块构成.
<!-- more -->
![image](https://leafmure.github.io/pic/postImage/H.264/H.264码流结构.png)

### NALU
H.264 码流由 NALU 为基本单元组成, 方便传输和保存, 其主要格式有两种: AVCC、Annex-B, 内容组成为:
```
[NALU] = [NALU Header] + [NALU Payload]
```

#### AVCC
AVCC是由 NALU Size + NALU构成, 码流前四个字节表示 NALU Size, 主要适用于Mp4、FLV和MKV等封装容器. 
```
[NALU Size] + [NALU Header] + [NALU Payload] + [NALU Size] + [NALU Header] + [NALU Payload]
```

#### Annex-B
Annex-B是由Start Code + NALU构成, 主要适用于TS封装容器.
```
[Start Code] + [NALU Header] + [NALU Payload] + [Start Code] + [NALU Header] + [NALU Payload]
```
Start Code 为: 0x00000001或0x000001, 3字节的0x000001只有一种场合下使用, 就是一个完整的帧被划分为多个Slice的时候, 从第二个Slice开始, 包含这些 Slice 的NALU使用3字节起始码。即若NALU包含的Slice为一帧的开始就用0x00000001, 否则就用0x000001。

另外为了避免真实数据 0x00000001、0x000001 冲突, 使用 0x03 作为防竞争码, 即将其中 0x01 用 0x03 表示, 与 Start Code 区分开来.

#### NALU Header
H.264 的 NALU Header 固定占用1个字节, 主要标识了三部分内容:
```
| forbidden_zero_bit | nal_ref_idc | nal_unit_type |
|--------------------|-------------|---------------|
|        1 bit       |    2 bit    |    5 bit      |

```
- forbidden_zero_bit, 默认为0, 当传输过程中, 发现NALU数据有错误时, 可设置为1, 以便接收方纠错或丢掉该单元。
- nal_ref_idc, 标识NALU的重要性，值越大，重要性越高。当解码器处理不过来时，可以丢掉重要性为0的NALU。
- nal_unit_type 标识 NALU 的类型，决定了如何解析 NALU Payload.

nal_unit_type:
```
0：未定义
1：非IDR图像中不采用数据划分的片段
2：非IDR图像中A类数据划分片段
3：非IDR图像中B类数据划分片段
4：非IDR图像中C类数据划分片段
5：IDR图像的片段
6：补充增强信息（SEI）
7：序列参数集（SPS）
8：图像参数集（PPS）
9：分割符
10：序列结束符
11：流结束符
12：填充数据
13：序列参数集扩展
14：带前缀的NAL单元
15：子序列参数集
16 – 18：保留
19：不采用数据划分的辅助编码图像片段
20：编码片段扩展
21 – 23：保留
24 – 31：未规定
```
nal_ref_idc和nal_unit_type 具有一定的相关性:

 nal_unit_type | nal_ref_idc |
---|---|
 1 - 4 | 如果其中一个 NALU 为0，则图片的柄脚中 Type为1-4（包括1-4）的所有 NAL 单元均为0 |
 5 IDR图像的片段 | 非0 |
 7 序列参数集（SPS）| 非0 |
 8 图像参数集（PPS）| 非0 |
 13 序列参数集扩展| 非0 |
 15 子序列参数集 | 非0 |
 6, 9, 10, 11 or 12 |  0 |

### SPS
H.264 中的 SPS（Sequence Parameter Set）是一种参数集，用于描述视频序列的特征和配置信息。SPS 是在 H.264 视频流中的一个元数据单元，它包含了视频编码器的设置和视频序列的特性。
- Profile 和 Level：指定视频编码的配置和兼容性级别。
- 图像尺寸和宽高比：描述视频图像的尺寸和宽高比。
- 帧率和比特率：指定视频的帧率和比特率，影响视频的流畅度和压缩效率。
- 帧间预测和帧内预测设置：描述视频编码中的预测模式和帧类型。
- 量化参数：控制视频质量和压缩比例的参数。
- 熵编码模式：指定熵编码的方式，影响编码的复杂度和压缩效率。
- 参考帧设置：指定参考帧的配置和使用方式，用于帧间预测。

![image](https://leafmure.github.io/pic/postImage/H.264/H.264-SPS.png)
相关参数说明可查看文档: [H.264](https://www.itu.int/rec/T-REC-H.264)

##### profile_idc：
标识当前H.264码流的profile。我们知道，H.264中定义了三种常用的档次profile：
- 基准档次：baseline profile;
- 主要档次：main profile;
- 扩展档次：extended profile;

在H.264的SPS中，第一个字节表示profile_idc，根据profile_idc的值可以确定码流符合哪一种档次。判断规律为：
- profile_idc = 66 → baseline profile;
- profile_idc = 77 → main profile;
- profile_idc = 88 → extended profile;

在新版的标准中，还包括了High、High 10、High 4:2:2、High 4:4:4、High 10 Intra、High
4:2:2 Intra、High 4:4:4 Intra、CAVLC 4:4:4 Intra等，每一种都由不同的profile_idc表示。

另外，constraint_set0_flag ~ constraint_set5_flag是在编码的档次方面对码流增加的其他一些额外限制性条件。

在我们实验码流中，profile_idc = 0x42 = 66，因此码流的档次为baseline profile。

##### level_idc
标识当前码流的Level。编码的Level定义了某种条件下的最大视频分辨率、最大视频帧率等参数，码流所遵从的level由level_idc指定。
当前码流中，level_idc = 0x1e = 30，因此码流的级别为3。

##### seq_parameter_set_id
表示当前的序列参数集的id。通过该id值，图像参数集pps可以引用其代表的sps中的参数。

##### log2_max_frame_num_minus4
用于计算MaxFrameNum的值。计算公式为MaxFrameNum = 2^(log2_max_frame_num_minus4 +
4)。MaxFrameNum是frame_num的上限值，frame_num是图像序号的一种表示方法，在帧间编码中常用作一种参考帧标记的手段。

##### pic_order_cnt_type
表示解码picture order count(POC)的方法。POC是另一种计量图像序号的方式，与frame_num有着不同的计算方法。该语法元素的取值为0、1或2。

##### log2_max_pic_order_cnt_lsb_minus4
用于计算MaxPicOrderCntLsb的值，该值表示POC的上限。计算方法为MaxPicOrderCntLsb = 2^(log2_max_pic_order_cnt_lsb_minus4 + 4)。

##### max_num_ref_frames
用于表示参考帧的最大数目。

##### gaps_in_frame_num_value_allowed_flag
标识位，说明frame_num中是否允许不连续的值。

##### pic_width_in_mbs_minus1
用于计算图像的宽度。单位为宏块个数，因此图像的实际宽度为:

frame_width = 16 × (pic_width_in_mbs_minus1 + 1);

##### pic_height_in_map_units_minus1
使用PicHeightInMapUnits来度量视频中一帧图像的高度。PicHeightInMapUnits并非图像明确的以像素或宏块为单位的高度，而需要考虑该宏块是帧编码或场编码。PicHeightInMapUnits的计算方式为：

PicHeightInMapUnits = pic_height_in_map_units_minus1 + 1;

##### frame_mbs_only_flag
标识位，说明宏块的编码方式。当该标识位为0时，宏块可能为帧编码或场编码；该标识位为1时，所有宏块都采用帧编码。根据该标识位取值不同，PicHeightInMapUnits的含义也不同，为0时表示一场数据按宏块计算的高度，为1时表示一帧数据按宏块计算的高度。

按照宏块计算的图像实际高度FrameHeightInMbs的计算方法为：

FrameHeightInMbs = ( 2 − frame_mbs_only_flag ) * PicHeightInMapUnits

##### mb_adaptive_frame_field_flag
标识位，说明是否采用了宏块级的帧场自适应编码。当该标识位为0时，不存在帧编码和场编码之间的切换；当标识位为1时，宏块可能在帧编码和场编码模式之间进行选择。

##### direct_8x8_inference_flag
标识位，用于B_Skip、B_Direct模式运动矢量的推导计算。

##### frame_cropping_flag
标识位，说明是否需要对输出的图像帧进行裁剪。

##### vui_parameters_present_flag
标识位，说明SPS中是否存在VUI信息。

### vui_parameters
vui_parameters 是视频编码中的一种参数，用于描述视频序列的附加信息，即视频使用者信息 (Video Usability Information)。vui_parameters 包含了与视频序列的使用和显示相关的配置信息，以提供更好的用户体验和兼容性。
- 宽高比 (aspect_ratio_info)：视频的显示宽高比，用于正确显示视频的宽高比例。
- 颜色参数 (color_primaries、transfer_characteristics、matrix_coefficients)：视频的颜色空间信息，用于正确解释和显示视频的颜色。
- 时间相关信息 (timing_info)：视频的时间参数，包括帧率和时间码等，用于正确解码和播放视频。
- 视频信号范围 (video_signal_type_present_flag)：视频信号的范围，例如全范围或标准范围，用于正确显示视频的亮度和对比度。
- 音频同步信息 (nal_hrd_parameters、vcl_hrd_parameters)：视频和音频之间的同步信息，确保音视频同步播放。

![image](https://leafmure.github.io/pic/postImage/H.264/H.264-VUI.png)
相关参数说明可查看文档: [H.264](https://www.itu.int/rec/T-REC-H.264)

### PPS
H.264 PPS（Parameter Set）是一种视频编码标准中的参数集，用于描述视频编码器的配置信息。它包含了编码器的参数设置，如图像尺寸、帧率、码率控制方式等。

![image](https://leafmure.github.io/pic/postImage/H.264/H.264-PPS.png)
相关参数说明可查看文档: [H.264](https://www.itu.int/rec/T-REC-H.264)

##### pic_parameter_set_id
表示当前PPS的id。某个PPS在码流中会被相应的slice引用，slice引用PPS的方式就是在Slice header中保存PPS的id值。该值的取值范围为[0,255]。

##### seq_parameter_set_id
表示当前PPS所引用的激活的SPS的id。通过这种方式，PPS中也可以取到对应SPS中的参数。该值的取值范围为[0,31]。

##### entropy_coding_mode_flag
熵编码模式标识，该标识位表示码流中熵编码/解码选择的算法。对于部分语法元素，在不同的编码配置下，选择的熵编码方式不同。例如在一个宏块语法元素中，宏块类型mb_type的语法元素描述符为“ue(v)
| ae(v)”，在baseline profile等设置下采用指数哥伦布编码，在main profile等设置下采用CABAC编码。

标识位entropy_coding_mode_flag的作用就是控制这种算法选择。当该值为0时，选择左边的算法，通常为指数哥伦布编码或者CAVLC；当该值为1时，选择右边的算法，通常为CABAC。

##### bottom_field_pic_order_in_frame_present_flag
标识位，用于表示另外条带头中的两个语法元素 delta_pic_order_cnt_bottom 和 delta_pic_order_cn是否存在的标识。这两个语法元素表示了某一帧的底场的POC的计算方法。

##### num_slice_groups_minus1
表示某一帧中slice group的个数。当该值为0时，一帧中所有的slice都属于一个slice group。slice group是一帧中宏块的组合方式，定义在协议文档的3.141部分。

##### num_ref_idx_l0_default_active_minus1、num_ref_idx_l0_default_active_minus1
表示当Slice Header中的num_ref_idx_active_override_flag标识位为0时，P/SP/B
slice的语法元素num_ref_idx_l0_active_minus1和num_ref_idx_l1_active_minus1的默认值。

##### weighted_pred_flag
标识位，表示在P/SP slice中是否开启加权预测。

##### weighted_bipred_idc
表示在B Slice中加权预测的方法，取值范围为[0,2]。0表示默认加权预测，1表示显式加权预测，2表示隐式加权预测。

##### pic_init_qp_minus26和pic_init_qs_minus26
表示初始的量化参数。实际的量化参数由该参数、slice header中的slice_qp_delta/slice_qs_delta计算得到。

##### chroma_qp_index_offset
用于计算色度分量的量化参数，取值范围为[-12,12]。

##### deblocking_filter_control_present_flag
标识位，用于表示Slice header中是否存在用于去块滤波器控制的信息。当该标志位为1时，slice header中包含去块滤波相应的信息；当该标识位为0时，slice header中没有相应的信息。

##### constrained_intra_pred_flag
若该标识为1，表示I宏块在进行帧内预测时只能使用来自I和SI类型宏块的信息；若该标识位0，表示I宏块可以使用来自Inter类型宏块的信息。

##### redundant_pic_cnt_present_flag
标识位，用于表示Slice header中是否存在redundant_pic_cnt语法元素。当该标志位为1时，slice header中包含redundant_pic_cnt；当该标识位为0时，slice header中没有相应的信息。

### H.264 VideoToolbox 编解码
#### H.264 编码
##### 1、创建 VTCompressionSessionRef 编码器
```Objective-C
/// 视频帧序列压缩的会话
VTCompressionSessionRef _compressionSession;

/// 创建编码session
/// videoCompressonOutputCallback 编码结果回调函数
OSStatus status = VTCompressionSessionCreate(NULL, _configuration.videoSize.width, _configuration.videoSize.height, kCMVideoCodecType_H264, NULL, NULL, NULL, videoCompressonOutputCallback, (__bridge void *)self, &_compressionSession);
if (status != noErr) {
    NSLog(@"HardwareVideoEncoder create fail");
    return;
}
```

##### 2、设置编码器参数
```Objective-C
/// 设置最大关键帧间隔
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_MaxKeyFrameInterval, (__bridge CFTypeRef)@(_configuration.videoMaxKeyframeInterval));
    
/// 设置最大关键帧时间
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_MaxKeyFrameIntervalDuration, (__bridge CFTypeRef)@(_configuration.videoMaxKeyframeInterval/_configuration.videoFrameRate));
    
/// 设置视频的帧率
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_ExpectedFrameRate, (__bridge CFTypeRef)@(_configuration.videoFrameRate));
    
/// 设置视频的平均码率，单位是 bps
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_AverageBitRate, (__bridge CFTypeRef)@(_configuration.videoBitRate));
    
/// 设置码率范围限制
NSArray *limit = @[@(_configuration.videoBitRate * 1.5 / 8), @(1)];
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_DataRateLimits, (__bridge CFArrayRef)limit);
    
/// 是否启用实时压缩
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_RealTime, kCFBooleanTrue);
    
/// H264 编码
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_ProfileLevel, kVTProfileLevel_H264_Main_AutoLevel);

/// 是否允许帧重新排序
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_AllowFrameReordering, kCFBooleanFalse);

/// 指定 H.264 编码器的熵编码模式
VTSessionSetProperty(_compressionSession, kVTCompressionPropertyKey_H264EntropyMode, kVTH264EntropyMode_CABAC);
```

##### 3、准备开始编码
```Objective-C
VTCompressionSessionPrepareToEncodeFrames(_compressionSession);
```

##### 4、开始编码
```Objective-C
/// 编码数据
- (void)encodeVideoData:(nullable CVPixelBufferRef)pixelBuffer timeStamp:(uint64_t)timeStamp {
    if(_isBackGround) return;
    frameCount++;
    CMTime presentationTimeStamp = CMTimeMake(frameCount, (int32_t)_configuration.videoFrameRate);
    VTEncodeInfoFlags flags;
    CMTime duration = CMTimeMake(1, (int32_t)_configuration.videoFrameRate);
    
    NSDictionary *properties = nil;
    if (frameCount % (int32_t)_configuration.videoMaxKeyframeInterval == 0) {
        properties = @{(__bridge NSString *)kVTEncodeFrameOptionKey_ForceKeyFrame: @YES};
    }
    NSNumber *timeNumber = @(timeStamp);
    
    OSStatus status = VTCompressionSessionEncodeFrame(_compressionSession, pixelBuffer, presentationTimeStamp, duration, (__bridge CFDictionaryRef)properties, (__bridge_retained void *)timeNumber, &flags);
    if (status != noErr) {
        [self createCompressionSession];
    }
}
```

##### 5、编码输出
```Objective-C
static void videoCompressonOutputCallback(void *refCon, void *frameRef, OSStatus status, VTEncodeInfoFlags infoFlags, CMSampleBufferRef sampleBuffer) {
    
    if (status != noErr) return;
    
    if (!sampleBuffer) return;
    
    CFArrayRef array = CMSampleBufferGetSampleAttachmentsArray(sampleBuffer, true);
    if (!array) return;
    
    CFDictionaryRef dic = (CFDictionaryRef)CFArrayGetValueAtIndex(array, 0);
    if (!dic) return;
    
    // 判断当前帧是否为关键帧
    BOOL keyframe = !CFDictionaryContainsKey(dic, kCMSampleAttachmentKey_NotSync);
    
    HardwareVideoEncoder *videoEncoder = (__bridge HardwareVideoEncoder *)refCon;
    /// 读取sps、pps 数据
    readSPSAndPPSData(videoEncoder, sampleBuffer, keyframe);
    /// 获取编码后的数据
    readBlockBuffer(sampleBuffer, frameRef, keyframe, videoEncoder);
}

```

##### 6、获取 sps、pps 数据
```Objective-C
static void readSPSAndPPSData(HardwareVideoEncoder *videoEncoder, CMSampleBufferRef sampleBuffer, BOOL keyframe) {
    if (keyframe && !videoEncoder->sps) {
        /// 格式信息
        CMFormatDescriptionRef format = CMSampleBufferGetFormatDescription(sampleBuffer);
        /// 获取 sps 数据
        size_t sparameterSetSize, sparameterSetCount;
        const uint8_t *sparameterSet;
        OSStatus statusCode = CMVideoFormatDescriptionGetH264ParameterSetAtIndex(format, 0, &sparameterSet, &sparameterSetSize, &sparameterSetCount, 0);
        if (statusCode == noErr) {
            /// 获取 pps 数据
            size_t pparameterSetSize, pparameterSetCount;
            const uint8_t *pparameterSet;
            OSStatus statusCode = CMVideoFormatDescriptionGetH264ParameterSetAtIndex(format, 1, &pparameterSet, &pparameterSetSize, &pparameterSetCount, 0);
            if (statusCode == noErr) {
                videoEncoder->sps = [NSData dataWithBytes:sparameterSet length:sparameterSetSize];
                videoEncoder->pps = [NSData dataWithBytes:pparameterSet length:pparameterSetSize];
            }
        }
    }
}
```

##### 7、编码后的帧数据
```Objective-C
/// 获取编码后的数据
static void readBlockBuffer(CMSampleBufferRef sampleBuffer, void *frameRef, BOOL keyframe, HardwareVideoEncoder *videoEncoder) {
        
    // 当前帧时间
    uint64_t timeStamp = [((__bridge_transfer NSNumber *)frameRef) longLongValue];
    
    CMBlockBufferRef dataBuffer = CMSampleBufferGetDataBuffer(sampleBuffer);
    
    size_t length, totalLength;
    char *dataPointer;
    OSStatus statusCodeRet = CMBlockBufferGetDataPointer(dataBuffer, 0, &length, &totalLength, &dataPointer);
    
    if (statusCodeRet == noErr) {
        
        size_t bufferOffset = 0;
        static const int AVCCHeaderLength = 4;
        // 循环获取nalu数据 VAVC
        while (bufferOffset < totalLength - AVCCHeaderLength) {
            uint32_t NALUnitLength = 0;
            /// 读取前 4 个字节, 表示 nalu 数据的长度.
            memcpy(&NALUnitLength, dataPointer + bufferOffset, AVCCHeaderLength);
            NALUnitLength = CFSwapInt32BigToHost(NALUnitLength);
            
            VideoDataFrame *videoFrame = [VideoDataFrame new];
            videoFrame.timestamp = timeStamp;
            videoFrame.data = [[NSData alloc] initWithBytes:(dataPointer + bufferOffset + AVCCHeaderLength) length:NALUnitLength];
            videoFrame.isKeyFrame = keyframe;
            videoFrame.sps = videoEncoder->sps;
            videoFrame.pps = videoEncoder->pps;
            
            if (videoEncoder.delegate && [videoEncoder.delegate respondsToSelector:@selector(videoEncoder:videoFrame:)]) {
                [videoEncoder.delegate videoEncoder:videoEncoder videoFrame:videoFrame];
            }
            
            bufferOffset += AVCCHeaderLength + NALUnitLength;
        }
    }
}

```

#### H.264 解码
##### 1、根据 Start Code 解析出 NALU 块
```Objective-C
typedef NS_ENUM(NSUInteger, LH264AnnexBFormatParserStartCodeState) {
    LH264AnnexBFormatParserStartCodeStateInit = 0, /// 默认匹配状态
    LH264AnnexBFormatParserStartCodeStateFind1Zero = 1, /// 匹配到一个 0
    LH264AnnexBFormatParserStartCodeStateFind2Zero = 2, /// 匹配到两个 0
    LH264AnnexBFormatParserStartCodeStateFindMoreThan2Zero = 3, /// 匹配到超过两个 0
    LH264AnnexBFormatParserStartCodeStateFind2ZeroAnd1One = 4, /// 匹配到 0x00 00 01
    LH264AnnexBFormatParserStartCodeStateFindMoreThan2ZeroAnd1One = 5 /// 匹配到 0x00 00 00 01
};

- (LHH264NALU *)nextRead
{
    if (self.parsingBuffer.length == 0) return nil;
    
    BOOL findStartCode = NO;
    LH264AnnexBFormatParserStartCodeState startCodeState = LH264AnnexBFormatParserStartCodeStateInit;
    
    uint8_t *ptr = (uint8_t *)[self.parsingBuffer bytes];
    
    for (; _position < self.parsingBuffer.length; ++_position) {
        uint8_t byte = *(ptr + _position);
        
        if (startCodeState == LH264AnnexBFormatParserStartCodeStateInit) {
            if (byte == 0) {
                startCodeState = LH264AnnexBFormatParserStartCodeStateFind1Zero;
            }
        } else if (startCodeState == LH264AnnexBFormatParserStartCodeStateFind1Zero) {
            if (byte == 0) {
                startCodeState = LH264AnnexBFormatParserStartCodeStateFind2Zero;
            } else {
                startCodeState = LH264AnnexBFormatParserStartCodeStateInit;
            }
        } else if (startCodeState == LH264AnnexBFormatParserStartCodeStateFind2Zero) {
            if (byte == 0) {
                startCodeState = LH264AnnexBFormatParserStartCodeStateFindMoreThan2Zero;
            } else if (byte == 1) {
                startCodeState = LH264AnnexBFormatParserStartCodeStateFind2ZeroAnd1One;
            } else {
                startCodeState = LH264AnnexBFormatParserStartCodeStateInit;
            }
        } else if (startCodeState == LH264AnnexBFormatParserStartCodeStateFindMoreThan2Zero) {
            if (byte == 0) {
                // 太多0了处理不了啊
            } else if (byte == 1) {
                startCodeState = LH264AnnexBFormatParserStartCodeStateFindMoreThan2ZeroAnd1One;
            } else {
                startCodeState = LH264AnnexBFormatParserStartCodeStateInit;
            }
        } else if (startCodeState == LH264AnnexBFormatParserStartCodeStateFindMoreThan2ZeroAnd1One || startCodeState == LH264AnnexBFormatParserStartCodeStateFind2ZeroAnd1One) {
            if (_isFirstStartCode) {
                _isFirstStartCode = NO;
                startCodeState = LH264AnnexBFormatParserStartCodeStateInit;
                _nextFramePosition = _position;
                continue;
            }
            findStartCode = YES;
            break;
        } else {
            /// 天涯海角
        }
    }
    
    /// 分割出数据块
    if (findStartCode) {
        if (startCodeState == LH264AnnexBFormatParserStartCodeStateFindMoreThan2ZeroAnd1One && _position - _nextFramePosition > 4) {
            /// 00 00 00 01
            NSData *payloadData = [self.parsingBuffer subdataWithRange:NSMakeRange(_nextFramePosition, _position - _nextFramePosition)];
            NSData *outputData = [payloadData subdataWithRange:NSMakeRange(0, payloadData.length - 4)];
            NSLog(@"outputData: %@", outputData);
            
            LHH264NALU *nalu = [[LHH264NALU alloc] initWithData:outputData formatType:LHH264NALUFormatTypeAnnexB];
            findStartCode = NO;
            startCodeState = LH264AnnexBFormatParserStartCodeStateInit;
            _nextFramePosition = _position;
            [self dropOldData];
            return nalu;
        } else if (startCodeState == LH264AnnexBFormatParserStartCodeStateFind2ZeroAnd1One && _position - _nextFramePosition > 3) {
            // 00 00 01
            if (_position - _nextFramePosition > 3) {
                NSData *payloadData = [self.parsingBuffer subdataWithRange:NSMakeRange(_nextFramePosition, _position - _nextFramePosition)];
                NSData *outputData = [payloadData subdataWithRange:NSMakeRange(0, payloadData.length - 3)];
                NSLog(@"outputData: %@", outputData);
                
                LHH264NALU *nalu = [[LHH264NALU alloc] initWithData:outputData formatType:LHH264NALUFormatTypeAnnexB];
                findStartCode = NO;
                startCodeState = LH264AnnexBFormatParserStartCodeStateInit;
                _nextFramePosition = _position;
                [self dropOldData];
                return nalu;
            }
        } else {
            /// 那能怎么办呢
            NSLog(@"那能怎么办呢");
        }
    } else {
        /// 那能怎么办呢
        NSLog(@"那能怎么办呢");
    }
    return nil;
}
```

##### 2、根据解析出来的 sps 和 pps 数据创建 CMFormatDescriptionRef
```Objective-C
const uint8_t *parameterSets[] = {
    (const uint8_t *)[self.spsData bytes],
    (const uint8_t *)[self.ppsData bytes],
};
    
size_t parameterSetSizes[] = {
    self.spsData.length,
    self.ppsData.length
};
    
CMFormatDescriptionRef format;
OSStatus ret = CMVideoFormatDescriptionCreateFromH264ParameterSets(kCFAllocatorDefault,
                                                                       2,
                                                                       parameterSets,
                                                                       parameterSetSizes,
                                                                       4,
                                                                       &format);
if (ret == noErr) {
    if (_videoFormatDescription) {
        CFRelease(_videoFormatDescription);
        videoFormatDescription = NULL;
    }
    _videoFormatDescription = format;
}
```

##### 3、将编码数据封装成 CMSampleBuffer
```Objective-C
CMBlockBufferRef blockBuffer = nil;
OSStatus ret = CMBlockBufferCreateWithMemoryBlock(kCFAllocatorDefault,
                                                      (void *)data.bytes,
                                                      data.length,
                                                      kCFAllocatorNull,
                                                      NULL,
                                                      0,
                                                      data.length,
                                                      0,
                                                      &blockBuffer);
if (ret != noErr) {
    // skip this tag
    return nil;
}

CMSampleBufferRef sampleBuffer = nil;
size_t blockDataSizeArray[1] = {CMBlockBufferGetDataLength(blockBuffer)};
OSStatus ret = CMSampleBufferCreateReady(kCFAllocatorDefault,
                                    blockBuffer,
                                    description,
                                    1,
                                    0,
                                    nil,
                                    1,
                                    blockDataSizeArray,
                                    &sampleBuffer);
```

##### 4、创建解码器
```Objective-C
VTDecompressionOutputCallbackRecord callbackRecord;
callbackRecord.decompressionOutputRefCon = (__bridge void * _Nullable)(self);
callbackRecord.decompressionOutputCallback = decompressionOutputCallback;
    
/// 视频帧序列解压的会话
VTDecompressionSessionRef _decompressionSession;
CFDictionaryRef attributes = (__bridge CFDictionaryRef)@{
        (NSString *)kCVPixelBufferPixelFormatTypeKey: @(kCVPixelFormatType_32BGRA)};
OSStatus ret = VTDecompressionSessionCreate(kCFAllocatorDefault,
                                                _formatDescription,
                                                nil,
                                                attributes,
                                                &callbackRecord,
                                                &_decompressionSession);
```

##### 5、解码
```Objective-C
VTDecodeFrameFlags flags = 0;
VTDecodeInfoFlags flagOut = 0;
VTDecompressionSessionDecodeFrame(_decompressionSession, sampleBuffer, flags, nil, &flagOut);
```

##### 6、解码输出
```Objective-C
static void decompressionOutputCallback(void *decompressionOutputRefCon,
                                        void *sourceFrameRefCon,
                                        OSStatus status,
                                        VTDecodeInfoFlags infoFlags,
                                        CVImageBufferRef imageBuffer,
                                        CMTime presentationTimeStamp,
                                        CMTime presentationDuration)
{
    HardwareVideoDecodeWorker *decoder = (__bridge HardwareVideoDecodeWorker *)decompressionOutputRefCon;
    if (decoder == nil) return;
    [decoder decodeSessionDidOuputWithStatus:status infoFlags:infoFlags imageBuffer:imageBuffer presentationTimeStamp:presentationTimeStamp duration:presentationDuration];
}

- (void)decodeSessionDidOuputWithStatus:(OSStatus)status
                              infoFlags:(VTDecodeInfoFlags)infoFlags
                            imageBuffer:(nullable CVImageBufferRef)imageBuffer
                  presentationTimeStamp:(CMTime)presentationTimeStamp
                               duration:(CMTime)duration
{
    if (status != noErr || imageBuffer == nil) {
        return;
    }
    
    OSStatus ret;
    
    CMSampleTimingInfo timingInfo;
    timingInfo.duration = duration;
    timingInfo.presentationTimeStamp = presentationTimeStamp;
    timingInfo.decodeTimeStamp = kCMTimeInvalid;
    
    CMVideoFormatDescriptionRef videoFormatDescription = NULL;
    ret = CMVideoFormatDescriptionCreateForImageBuffer(kCFAllocatorDefault,
                                                       imageBuffer,
                                                       &videoFormatDescription);
    if (ret != noErr) {
        return;
    }
    
    CMSampleBufferRef sampleBuffer;
    ret = CMSampleBufferCreateForImageBuffer(kCFAllocatorDefault,
                                             imageBuffer,
                                             true,
                                             nil,
                                             nil,
                                             videoFormatDescription,
                                             &timingInfo,
                                             &sampleBuffer);
    CFRelease(videoFormatDescription);
    
    if (ret != noErr) {
        return;
    }
    
    if (sampleBuffer == nil) {
        return;
    }
    
    if (self.delegate && [self.delegate respondsToSelector:@selector(videoDecoder:decodeSampleBuffer:)]) {
        [self.delegate videoDecoder:self decodeSampleBuffer:sampleBuffer];
    }
}
```
