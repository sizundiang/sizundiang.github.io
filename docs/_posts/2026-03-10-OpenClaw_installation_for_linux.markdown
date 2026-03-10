---
layout: post
title: OpenClaw信创电脑安装记录（含x86和arm64两种指令集类型）
author: sizundiang
tags: [技术, AI, OpenClaw]
categories: tech_notes
---

统信UOS桌面操作系统目前已成为不可忽视的操作系统当中的一强，大家更熟悉的可能是信创电脑这个名字，在很多行业已经得到了推广与使用，统信系统兼容多种指令集，目前市面上有x86架构（以联想开天为代表）和arm64架构（以华为擎云为代表），至少我能接触到的有这两类。

以上是一些背景信息，最近OpenClaw爆火也让我想装一个试试，但网上找了一圈，好像没有专门针对统信系统的安装教程，当然，统信系统本质还是Debian，或者说Linux，其实装起OpenClaw反而更加方便，但这也只是理论上，实际装起来还是遇到不少坑的。因此本文也是对我在上述两种指令集下的统信电脑安装流程的一个总结，也是方便后续能够翻出来复用吧。

X86架构下的统信系统没啥好说的，至少在安装OpenClaw上，没遇到太大的问题，先按照Node官网说明安装好nvm（Node版本管理工具）和Node.js（目前最新的LTS版本是24.14.0），并检验版本：

![Node installation](/assets/images/2026-03-10-OpenClaw_installation_for_linux/node.png)

更新镜像源（可选）：

```
npm config set registry https://registry.npmmirror.com
```

补充其他必要的工具，如Git，照着官网来就好：

![Git installation](/assets/images/2026-03-10-OpenClaw_installation_for_linux/git.png)

Git需事先配置账号，这一块就不展开了。

在正式安装前最好更新一下软件包列表，以及安装一下等会编译可能用到的工具链和依赖：


```
# 1. 更新软件包列表
sudo apt update

# 2. 安装编译所需的工具链和依赖
sudo apt install -y build-essential cmake python3
```

接下来可以尝试执行安装openclaw的指令了：

```
npm install -g openclaw@latest
```

Cmake是必需预先装的，在后续进行llama.cpp的编译过程中会用到，当在安装日志中看到形如：

```
npm error ^[[2K^[[1A^[[2K^[[1A^[[2K^[[G[node-llama-cpp] ✔ Cloned ggml-org/llama.cpp (local bundle)
npm error ^[[?25h[node-llama-cpp] ◷ Downloading cmake
npm error [node-llama-cpp] ✖ Failed to download cmake
npm error [node-llama-cpp] To install "cmake", you can run "sudo apt update && sudo apt install -y cmake"
npm error [node-llama-cpp] ◷ Downloading cmake
npm error [node-llama-cpp] ✖ Failed to download cmake
npm error [node-llama-cpp] To install "cmake", you can run "sudo apt update && sudo apt install -y cmake"
npm error [node-llama-cpp] The prebuilt binary for platform "linux" "x64" with Vulkan support is not compatible with the current system, falling back to using no GPU
npm error [node-llama-cpp] The prebuilt binary for platform "linux" "x64" is not compatible with the current system, falling back to building from source
的日志时，就知道该装cmake了（GPU支持这一块不设置也是可以的）
同样是这个llama.cpp，在arm64指令集的统信系统中依然是个坑，
npm error command failed
npm error command sh -c node ./dist/cli/cli.js postinstall
npm error [node-llama-cpp] Cloning llama.cpp
npm error ^[[?25l[node-llama-cpp] Cloning ggml-org/llama.cpp (local bundle)                    0%
npm error ^[[2K^[[1A^[[2K^[[G[node-llama-cpp] Cloning ggml-org/llama.cpp (local bundle)                    0%
npm error ^[[2K^[[1A^[[2K^[[G[node-llama-cpp] Cloning ggml-org/llama.cpp (local bundle)                    1%
...
✔ Cloned ggml-org/llama.cpp (local bundle)
npm error ^[[?25hNot searching for unused variables given on the command line.
npm error -- The C compiler identification is GNU 8.3.0
npm error -- The CXX compiler identification is GNU 8.3.0
...
npm error [ 11%] Building CXX object llama.cpp/ggml/src/CMakeFiles/ggml-cpu.dir/ggml-cpu/arch/arm/repack.cpp.o
npm error [ 12%] Linking CXX static library libcpp-httplib.a
npm error [ 12%] Built target cpp-httplib
npm error [node-llama-cpp] A prebuilt binary was not found, falling back to using no GPU
npm error [node-llama-cpp] The prebuilt binary for platform "linux" "arm64" is not compatible with the current system, falling back to building from source
npm error ^[[?25l^[[?25hCMAKE_BUILD_TYPE=Release
npm error CMake Warning at llama.cpp/ggml/src/ggml-cpu/CMakeLists.txt:146 (message):
npm error   ARM -march/-mcpu not found, -mcpu=native will be used
npm error Call Stack (most recent call first):
npm error   llama.cpp/ggml/src/CMakeLists.txt:445 (ggml_add_cpu_backend_variant_impl)
...
npm error ./.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/node_modules/node-llama-cpp/llama/llama.cpp/ggml/src/ggml-cpu/arch/arm/quants.c: In function ‘ggml_vec_dot_q3_K_q8_K’:
npm error ./.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/node_modules/node-llama-cpp/llama/llama.cpp/ggml/src/ggml-cpu/ggml-cpu-impl.h:301:27: error: implicit declaration of function ‘vld1q_s8_x4’; did you mean ‘vld1q_s8_x2’? [-Werror=implicit-function-declaration]
npm error  #define ggml_vld1q_s8_x4  vld1q_s8_x4
npm error                            ^~~~~~~~~~~
...
```

这是ARM架构下编译过程产生的问题，当前系统编译器版本较旧，不支持一些特定的扩展指令集（具体也是通过AI才知道），这么说只需要更新成最新的编译器即可，执行以下指令：

```
sudo apt install gcc
```

但提示当前已是最新版本，因此通过包管理器更新的方式行不通，最后在AI的指导下通过源码编译安装：

```
# 安装依赖
sudo apt install wget libgmp-dev libmpfr-dev libmpc-dev
# 下载源码
wget https://ftp.gnu.org/gnu/gcc/gcc-11.2.0/gcc-11.2.0.tar.gz
tar -xzvf gcc-11.2.0.tar.gz
cd gcc-11.2.0
# 下载依赖库
./contrib/download_prerequisites

# 编译安装
mkdir build && cd build
../configure --prefix=/usr/local/gcc-11.2.0 --enable-languages=c,c++ --disable-multilib
make -j$(nproc)
sudo make install

#  配置环境变量
echo 'export PATH=/usr/local/gcc-11.2.0/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

这时系统存在两套编译器，需要设置优先级，否则系统仍默认使用原先的版本：

```
sudo update-alternatives --install /usr/bin/gcc gcc 新版gcc安装路径 100
sudo update-alternatives --install /usr/bin/g++ g++ 新版g++安装路径 100
```

单纯通过

```
gcc --version
g++ --version
```

指令并不能表明系统编译时使用的版本。

做完这一步，再次尝试安装，又出现了新的错误：

```
npm error
npm error [node-llama-cpp] To resolve errors related to Vulkan compilation, see the Vulkan guide: https://node-llama-cpp.withcat.ai/guide/vulkan
npm error Not searching for unused variables given on the command line.
npm error -- The C compiler identification is GNU 11.2.0
npm error -- The CXX compiler identification is GNU 11.2.0
...
npm error [100%] Built target llama-addon
npm error [node-llama-cpp] A prebuilt binary was not found, falling back to using no GPU
npm error [node-llama-cpp] The prebuilt binary for platform "linux" "arm64" is not compatible with the current system, falling back to building from source
npm error ^[[?25l^[[?25hCMAKE_BUILD_TYPE=Release
npm error CMake Warning at llama.cpp/ggml/src/ggml-cpu/CMakeLists.txt:146 (message):
npm error   ARM -march/-mcpu not found, -mcpu=native will be used
npm error Call Stack (most recent call first):
npm error   llama.cpp/ggml/src/CMakeLists.txt:445 (ggml_add_cpu_backend_variant_impl)
...
npm error [node-llama-cpp] Failed to build llama.cpp with no GPU support. Error: Error: /lib/aarch64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.29' not found (required by ./.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/node_modules/node-llama-cpp/llama/localBuilds/linux-arm64/Release/llama-addon.node)
可以看到，问题在GLIBCXX 版本不兼容（Vulkan库缺失可以不管），gcc的版本更新了，但系统中安装的 libstdc++6 库版本过旧，此时使用包管理起也是不能更新到对应版本的，最终我采用了静态链接的方式（AI指导）：
export LDFLAGS="-static-libstdc++ -static-libgcc"
```

终于能正常安装了，一路下来踩了挺多坑，但也学到不少。

最后提醒一下OpenClaw使用及配置过程中的一些小细节，首先是模型配置，Base URL部分，参考对应的模型平台文档以外，还需注意不同工具的适配要求，比如一般Base URL会以v1结尾，但如Cherry Studio、Claude Code，不加v1不影响，但OpenClaw这里需要自己加上（不加不会报错，但不能正常调用模型）.

另外一个问题是网关token，启动OpenClaw后，启动Dashboard会要求配置网关token，这个在.openclaw/openclaw.json文件中有，但需要在网页上控制-概览-网关访问选项上进行配置并尝试连接才生效，这里需注意网页缓存问题，如果token不生效，建议清空一下网页缓存。

还有一个问题是权限，26.3.2版本使用了新的权限管理规则，并对默认权限进行了收紧，现在默认仅具备聊天权限（messaging）,通过飞书等聊天软件配置的权限不受影响，但如果需要操作本地文件等权限，需要对tools.profile进行更新。