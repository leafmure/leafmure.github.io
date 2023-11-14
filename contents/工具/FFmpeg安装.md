---
title: FFmpeg 安装
date: 2023-09-19 17:35:47
categories:
- 工具
tags:
- FFmpeg
- rtmp
- SRS
- H.256推流
keywords:
- FFmpeg,rtmp,SRS,H.256
description:
images:
---
# FFmpeg 安装

## Homebrew 方式安装
1、执行安装命令
```
brew install ffmpeg
brew install ffmpeg --HEAD # 安装最新
```

2、选择第三方仓库: homebrew-ffmpeg 和 varenc-homebrew-ffmpeg 
```
brew tap homebrew-ffmpeg/ffmpeg # homebrew-ffmpeg 仓库
```
<!-- more -->
3、根据第三方仓库安装
```
brew install homebrew-ffmpeg/ffmpeg/ffmpeg
```

4、关联 options
```
# 获取支持的options
brew options homebrew-ffmpeg/ffmpeg/ffmpeg

# 关联options
brew install homebrew-ffmpeg/ffmpeg/ffmpeg --with-aribb24 --with-chromaprint --with-decklink --with-fdk-aac --with-game-music-emu --with-jack --with-jpeg-xl --with-libaribb24 --with-libbluray --with-libbs2b --with-libcaca --with-libflite --with-libgsm --with-libmodplug --with-libopenmpt --with-librist --with-librsvg --with-libsoxr --with-libssh --with-libvidstab --with-libvmaf --with-libxml2 --with-libzvbi --with-opencore-amr --with-openh264 --with-openjpeg --with-openssl --with-openssl@1.1 --with-rav1e --with-rtmpdump --with-rubberband --with-speex --with-srt --with-svt-av1 --with-tesseract --with-two-lame --with-webp --with-xvid --with-zeromq --with-zimg
```

## FFmpeg 支持 RTMP H265
### 安装 libx264
```
git clone https://code.videolan.org/videolan/x264.git
cd x264
mkdir build
./configure --prefix=$(pwd)/build --disable-asm --disable-cli --disable-shared --enable-static
make -j10
make install
```
### 安装 libx265
```
git clone https://bitbucket.org/multicoreware/x265_git.git
cd x265_git/build/linux
mkdir build
cmake -DCMAKE_INSTALL_PREFIX=$(pwd)/build -DENABLE_SHARED=OFF ../../source
make -j10
make install
```
### 设置 libx264、libx265 环境变量
```
export PKG_CONFIG_PATH=~/x264/build/lib/pkgconfig:~/x265_git/build/linux/build/lib/pkgconfig
```
### ffmpeg 中 rtmp/flv 模块支持 h.265
```
git clone -b 5.1 https://github.com/runner365/ffmpeg_rtmp_h265.git
cd ffmpeg_rtmp_h265
cp flv.h ~/FFmpeg/libavformat/
cp flv*.c ~/FFmpeg/libavformat/
```
### ffmpeg 重新编译安装
```
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
git checkout n6.0
mkdir build
./configure \
  --prefix=$(pwd)/build \
  --enable-gpl --enable-nonfree --enable-pthreads --extra-libs=-lpthread \
  --disable-asm --disable-x86asm --disable-inline-asm \
  --enable-decoder=aac --enable-decoder=aac_fixed --enable-decoder=aac_latm --enable-encoder=aac \
  --enable-libx264 --enable-libx265 \
  --pkg-config-flags='--static'
make -j10
make install
```

## rtmp/flv h.265 服务器支持
nginx-rtmp 只支持 h.264, SRS 已支持 rtmp/flv h.265, 因此建议使用 SRS 服务.
```
git clone https://github.com/ossrs/srs.git
cd trunk
./configure --osx --h265=on
make
```
SRS 相关操作指令
```
# 启动
./objs/srs -c conf/rtmp.conf

# 查看日志
tail -n 30 -f ./objs/srs.log

# 查看服务器状态
./etc/init.d/srs status
```
具体详细见文档[SRS](https://ossrs.net/lts/zh-cn/docs/v5/doc/getting-started-build)