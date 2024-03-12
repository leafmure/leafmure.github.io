---
title: Mac 本地部署 Stable Diffusion
date: 2023-11-14 17:35:47
categories:
- 工具
tags:
- Stable Diffusion
keywords:
- Stable Diffusion,Mac
description:
images:
---
### 使用的 Mac 配置
MacBook Pro (2022)
芯片: Apple M2
内存: 16GB
存储: 512GB
系统: Ventura 13.0
<!-- more -->

### 环境要求
#### Homebrew
##### 官方安装 (需要梯子)
安装命令:
```bash
/bin/zsh -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

卸载命令:
```bash
/bin/zsh -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall.sh)"
```

##### 国内维护镜像
安装命令:
```bash
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```

卸载命令:
```bash
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/HomebrewUninstall.sh)"
```

##### Homebrew 基础指令
```bash
# 查看版本
brew -v

# 安装
brew install xxx
```

#### Python
查看 Python 是否安装
```bash
python3 -V

输出: Python 3.10.10
```
如果没有输出版本号, 前往 [Python 官网](https://www.python.org/downloads/macos/) 下载, 教程使用的 Python 3.10.10, 具体以 [stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) 要求.

安装后, 需要配置环境变量.
```bash
# 1.编辑 ~/.zprofile
vim ~/.zprofile

# 2.在 zprofile 加入以下内容:
PATH="/Library/Frameworks/Python.framework/Versions/3.10/bin:${PATH}"
export PATH
alias python="/Library/Frameworks/Python.framework/Versions/3.10/bin/python3"

# 3.使配置生效
source ~/.zprofile
```

### 下载 Stable Diffusion
使用 git clone Stable Diffusino WebUI 仓库
```bash
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui
```

### 下载模型
#### 模型类型
模型主要分为三大类: 大模型(checkpoint)、Lora、VAE, 不同类型的模型放置路径不一样.
##### 大模型(checkpoint)
模型文件后缀为: ckpt 或者 safetensors, 需放置在 /stable-diffusion-webui/models/Stable-diffusion 目录下.

##### Lora 模型
需放置在 /stable-diffusion-webui/models/Lora 目录下, Lora 文件夹需要执行 ./webui.sh 生成.

##### VAE 模型
需放置在 /stable-diffusion-webui/models/VAE 目录下

#### 模型获取
需在启动 WebUI 前下载好一个大模型放置目录下, 否则会自动下载默认模型, 比较耗费时间.

##### [C站](https://civitai.com/)
![image](https://meanmouse.github.io/pic/postImage/Mac-本地部署-StableDiffusion/C站模型获取.png)
##### [Hugging Face](https://huggingface.co/models?pipeline_tag=text-to-image&sort=downloads)
![image](https://meanmouse.github.io/pic/postImage/Mac-本地部署-StableDiffusion/HuggingFace1.png)
![image](https://meanmouse.github.io/pic/postImage/Mac-本地部署-StableDiffusion/HuggingFace1.png)
##### [LiblibAI](https://www.liblib.ai/modelinfo/8b4b7eb6aa2c480bbe65ca3d4625632d)
![image](https://meanmouse.github.io/pic/postImage/Mac-本地部署-StableDiffusion/LiblibAI.png)

### 启动 WebUI
#### 准备依赖工具
```bash
brew install cmake protobuf rust wget
```

#### 执行 webui.sh
进入从 git 拉取下来的 stable-diffusion-webui 目录, 执行下面命令.
```bash
 cd stable-diffusion-webui

./webui.sh
```
![image](https://meanmouse.github.io/pic/postImage/Mac-本地部署-StableDiffusion/应用界面.png)

### 汉化
可使用 [stable-diffusion-webui-chinese](https://github.com/VinsonLaro/stable-diffusion-webui-chinese) 扩展进行汉化, 具体使用方法请阅读[安装说明](https://github.com/VinsonLaro/stable-diffusion-webui-chinese#%E6%96%B9%E6%B3%951%E9%80%9A%E8%BF%87webui%E6%8B%93%E5%B1%95%E8%BF%9B%E8%A1%8C%E5%AE%89%E8%A3%85)

#### 遇到的问题
##### 1. [Failed building wheel for basicsr](https://github.com/AUTOMATIC1111/stable-diffusion-webui/issues/13113)
在 stable-diffusion-webui/modules/launch_utils.py 文件第138行中的 python 命令中插入 --use-pep517
```
return run(f'"{python}" -m pip {command} --use-pep517 --prefer-binary{index_url_line}', desc=f"Installing {desc}", errdesc=f"Couldn't install {desc}", live=live)
```
[github issues](https://github.com/AUTOMATIC1111/stable-diffusion-webui/issues/13113)

##### 2. [stable-diffusion-webui-extensions 加载 SSL 错误]
在 launch_utils.py 中的 from module.paths_internal import script_path, extensions_dir 后面添加:
```
import ssl ssl._create_default_https_context = ssl._create_unverified_context
```
[github issues](https://github.com/AUTOMATIC1111/stable-diffusion-webui/issues/9285)