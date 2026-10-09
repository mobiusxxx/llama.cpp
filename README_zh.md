# 在 Windows 上使用 llama.cpp：模型量化与服务部署

本文档详细介绍了在 Windows 系统上使用 `llama.cpp` 进行大型语言模型（LLM）量化，并通过 `llama-server` 进行 CPU 部署的完整流程。

## 目录

- [在 Windows 上使用 llama.cpp：模型量化与服务部署](#在-windows-上使用-llamacpp模型量化与服务部署)
  - [目录](#目录)
  - [1. 环境准备与模型量化](#1-环境准备与模型量化)
    - [1.1 下载 `llama.cpp`](#11-下载-llamacpp)
    - [1.2 配置 Windows 编译环境](#12-配置-windows-编译环境)
    - [1.3 准备原始 LLM 模型文件](#13-准备原始-llm-模型文件)
    - [1.4 运行量化命令](#14-运行量化命令)
    - [1.5 验证量化模型](#15-验证量化模型)
  - [2. 服务部署与测试](#2-服务部署与测试)
    - [2.1 编译 `llama.cpp`](#21-编译-llamacpp)
    - [2.2 启动 `llama-server`](#22-启动-llama-server)
    - [2.3 本地连接测试](#23-本地连接测试)

---

## 1. 环境准备与模型量化

### 1.1 下载 `llama.cpp`

首先，克隆 `llama.cpp` 的官方代码仓库：

```bash
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
```

### 1.2 配置 Windows 编译环境

在 Windows 上编译 `llama.cpp`，推荐使用 `CMake` 和 `Visual Studio`。

*   **安装 Visual Studio:**
    从 [Visual Studio 官网](https://visualstudio.microsoft.com/zh-hans/downloads/) 下载并安装 Visual Studio Community 版本。在安装过程中，请确保选中 **“使用 C++ 的桌面开发”** 工作负载。

*   **安装 CMake:**
    从 [CMake 官网](https://cmake.org/download/) 下载并安装最新版本的 CMake。在安装过程中，请确保将 CMake 添加到系统路径中。

*   **安装 Python 依赖:**
    `llama.cpp` 的某些脚本需要 Python 依赖。推荐使用 `uv` 进行虚拟环境和依赖管理。

    1.  **安装 uv:**
        ```bash
        pip install uv
        ```

    2.  **创建并激活虚拟环境:**
        ```bash
        # 创建虚拟环境
        uv venv

        # 激活虚拟环境 (Windows)
        .\.venv\Scripts\activate
        ```

    3.  **安装依赖:**
        `uv` 会自动使用 `pyproject.toml` 或 `requirements.txt` 文件来安装依赖。
        ```bash
        # 根据 pyproject.toml 同步依赖
        uv pip sync

        # 或者，如果项目只使用 requirements.txt
        # uv pip install -r requirements.txt
        ```

### 1.3 准备原始 LLM 模型文件

您需要从 Hugging Face 或其他来源下载预训练的 LLM 模型文件。这些文件通常是 `.pth` 或 `.bin` 格式。

### 1.4 运行量化命令

`llama.cpp` 提供了 `convert.py` 脚本，用于将 Hugging Face 格式的模型转换为 GGUF 格式。

```bash
python convert.py models/7B --outtype q4_0 --outfile converted.gguf
```

*   `models/7B`: 替换为您下载的原始模型路径。
*   `--outtype`: 指定量化类型，例如 `q4_0` (4-bit), `q8_0` (8-bit), `f16` (16-bit float) 等。
*   `--outfile`: 指定输出的 GGUF 文件名。

### 1.5 验证量化模型

量化完成后，您可以使用 `perplexity` 工具来评估模型的困惑度，以验证量化过程没有严重损害模型性能。

```bash
# 首先需要编译出 perplexity.exe
./perplexity -m converted.gguf -f wiki.test.raw
```

---

## 2. 服务部署与测试

### 2.1 编译 `llama.cpp`

使用 CMake 生成 Visual Studio 项目，然后进行编译。

1.  **创建 build 目录:**
    ```bash
    mkdir build
    cd build
    ```

2.  **生成项目文件:**
    ```bash
    cmake ..
    ```

3.  **编译:**
    ```bash
    cmake --build . --config Release
    ```

    编译成功后，您会在 `build\bin\Release` 目录下找到 `llama-server.exe` 和其他可执行文件。

### 2.2 启动 `llama-server`

使用以下命令启动 `llama-server`：

```bash
./llama-server.exe -m converted.gguf -c 32000 -b 4096 --host 0.0.0.0 -t 8 --device none --n-gpu-layers 0
```

*   `-m`: 指定加载的 GGUF 模型文件。
*   `-c`: 设置上下文长度。
*   `--host`: 绑定的主机地址，`0.0.0.0` 表示监听所有网络接口。
*   `-t`: 指定使用的 CPU 线程数。
*   `--device none --n-gpu-layers 0`: 禁用 GPU 加速，确保在没有 CUDA 支持的环境中运行。
*   `-b`: 设置批处理大小，默认值为 512。

### 2.3 本地连接测试

服务启动后，您可以通过 `curl` 或浏览器访问 `http://localhost:8080` 来测试服务是否正常。openai格式的请求示例如下：

```bash
curl --request POST \
    --url http://localhost:8080/v1/completion \
    --header "Content-Type: application/json" \
    --data '{
        "model": "converted.gguf",
        "messages": [
            {
                "role": "user",
                "content": "你好"
            }
        ]
    }'
```
