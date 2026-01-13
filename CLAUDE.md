# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个基于 GPU 加速的 Tron（波场）靓号地址生成器，使用 C++ 和 OpenCL 实现。程序通过 GPU 并行计算快速生成符合特定前缀/后缀规则的 Tron 地址。

## 构建与运行

### Mac 平台
```bash
# 构建
make

# 运行（基本用法）
./profanity.x64 --matching profanity.txt

# 运行（跳过集成显卡）
./profanity.x64 --matching profanity.txt --skip 1

# 运行（指定前后缀匹配数量）
./profanity.x64 --matching profanity.txt --prefix-count 1 --suffix-count 8
```

### Windows 平台
使用 Visual Studio 编译，需要安装：
- NVIDIA 显卡驱动
- Visual Studio（包含 C++ 桌面开发组件）
- CUDA 工具包

### Linux 平台
```bash
# 安装依赖
# 需要先安装 cuda 驱动和 g++

# 编译
g++ Dispatcher.cpp Mode.cpp precomp.cpp profanity.cpp SpeedSample.cpp \
    -I./Curl/include -I./OpenCL/include -o profanity.x64
```

## 核心架构

### 主要组件

1. **profanity.cpp** - 程序入口
   - 解析命令行参数
   - 初始化 OpenCL 设备和上下文
   - 加载或编译 OpenCL 内核
   - 创建 Dispatcher 并启动地址生成

2. **Dispatcher.cpp/hpp** - 核心调度器
   - 管理多个 GPU 设备的并行计算
   - 实现 `createSeed()` 方法（已修复原版 profanity 的私钥安全漏洞）
   - 处理结果输出（文件/HTTP POST）
   - 地址格式转换（Ethereum 格式 -> Tron Base58 格式）

3. **Mode.cpp/hpp** - 匹配模式管理
   - 解析匹配规则文件（profanity.txt）
   - 支持两种规则格式：
     - 纯字符模式（如 `TTTTTTTTTTZZZZZZZZZZ`）
     - 完整地址模式（自动提取前后缀）

4. **OpenCL 内核**
   - `kernel_profanity.hpp` - 主要地址生成逻辑
   - `kernel_keccak.hpp` - Keccak-256 哈希算法
   - `kernel_sha256.hpp` - SHA-256 哈希算法

5. **precomp.cpp/hpp** - 预计算表
   - 包含椭圆曲线预计算数据，用于加速地址生成

### 关键参数

- `--matching` - 匹配规则文件或单个地址
- `--prefix-count` - 最少匹配前缀位数（0-10）
- `--suffix-count` - 最少匹配后缀位数（0-10）
- `--quit-count` - 生成指定数量后退出
- `--skip` - 跳过指定索引的 GPU 设备（用于过滤集成显卡）
- `--output` - 输出结果到文件
- `--post` - 通过 HTTP POST 发送结果到指定 URL

### 地址生成流程

1. OpenCL 内核在 GPU 上并行生成私钥
2. 通过椭圆曲线计算得到公钥
3. 对公钥进行 Keccak-256 哈希，取后 20 字节作为 Ethereum 格式地址
4. 转换为 Tron 格式：
   - 添加 `0x41` 前缀
   - 计算两次 SHA-256 哈希
   - 取前 4 字节作为校验和
   - Base58 编码得到最终 Tron 地址

### 安全修复

原版 profanity 存在私钥可爆破漏洞。本项目在 `Dispatcher.cpp:createSeed()` 中已修复：
- 使用 `std::random_device` 和 `std::mt19937_64` 生成高质量随机种子
- 为每个 64 位分量使用独立的随机数生成器

## 开发注意事项

- 开发调试时可使用 `-I` 参数设置较小值以加快启动速度
- Mac 环境可能需要指定 `-w 1` 以生成正确的私钥
- 部分平台需要使用 `-s` 参数跳过集成显卡设备
- 生成的地址务必进行验证，确保私钥与地址匹配
- 建议对生成的地址进行多签后再使用

## 文件结构

- `*.cpp/hpp` - C++ 源代码和头文件
- `kernel_*.hpp` - OpenCL 内核代码
- `profanity.txt` - 默认匹配规则文件
- `cache-opencl.*` - OpenCL 编译缓存（加快启动速度）
- `Curl/` - libcurl 库（用于 HTTP POST）
- `OpenCL/` - OpenCL 头文件
- `dist/` - 发布包目录
