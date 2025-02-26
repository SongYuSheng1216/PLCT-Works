# Run DeepSeek-R1-Distill Model TH1520 RVV Comparison

## 测试目的

本次测试旨在对比TH1520 上运行不同指令集下的 DeepSeek-R1 Distill Model的性能。测试通过运行llama-bench得到测试结果。

（指令集差异为 开/不开 v0p7 或 XTheadVector ）

## 测试环境

#### 配置信息

* OS:Debian GNU/Linux trixie/sid
* RevyOS-Release: 20250123_195216
* Board: Licheepi 4A (16GB + 128GB)

#### 工具链

​	后续的编译工作需要使用 RVV0p7 指令集， 而大多数编译器仅具有对 RVV1p0 的支持。因此，需要下载为该系列芯片打造的编译器。PLCT 提供了相关专用工具链，可以使用 `ruyi` 工具下载，也可以手动下载。

* **rv64gc**：

```shell
wget https://mirror.iscas.ac.cn/ruyisdk/dist/RuyiSDK-20231212-Upstream-Sources-HOST-riscv64-linux-gnu-riscv64-unknown-linux-gnu.tar.xz
```

* **rv64gcv0p7**：

```shell
wget https://mirror.iscas.ac.cn/ruyisdk/dist/RuyiSDK-20240222-T-Head-Sources-T-Head-2.8.0-HOST-riscv64-linux-gnu-riscv64-plctxthead-linux-gnu.tar.xz
```

随后解压文件夹，将工具链bin文件夹加入环境变量$PATH

($PATH 中先放入需要测试的指令集对应的工具链，后续测试另一指令集时再在$PATH加入其对应工具链)

```shell
tar -xvf [file-name]
export PATH=file/……/bin:$PATH
```

## 下载模型

使用pip安装依赖

```shell
pip install -U huggingface_hub
```

**下载模型**

```shell
huggingface-cli download unsloth/DeepSeek-R1-Distill-Qwen-1.5B-GGUF --local-dir unsloth/DeepSeek-R1-Distill-Qwen-1.5B
```

本报告中使用unsloth/DeepSeek-R1-Distill-Qwen-1.5B-GGUF模型进行测试

## 测试步骤

### OpenBLAS

1. **clone**

```shell
git clone git@github.com:OpenMathLib/OpenBLAS.git
```

2. **compile**

* rv64gc

```shell
sudo make HOSTCC=riscv64-unknown-linux-gnu-gcc TARGET=RISCV64_GENERIC CC=riscv64-unknown-linux-gnu-gcc FC=riscv64-unknown-linux-gnu-gfortran
```

* rv64gcv0p7

```shell
sudo make HOSTCC=riscv64-unknown-linux-gnu-gcc TARGET=C910V CC=riscv64-plctxthead-linux-gnu-gcc FC=riscv64-plctxthead-linux-gnu-gfortran
```

3. **install**

```shell
sudo make install PREFIX=[toolchain_directory]/riscv64-unknown-linux-gnu/sysrootf
```

### llama.cpp

1. **clone**

```shell
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
```

2. **patch**

   由于在llama.cpp/ggml/src/ggml-cpu/CMakeLists.txt中，默认为rv64gcv，因此要适配正确的v0p7架构，需要手动修改

* rv64gc

  不做修改

* rv64gcv0p7

```
--- a/ggml/src/ggml-cpu/CMakeLists.txt
+++ b/ggml/src/ggml-cpu/CMakeLists.txt
@@ -308,7 +308,7 @@ function(ggml_add_cpu_backend_variant_impl tag_name)
     elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "riscv64")
         message(STATUS "RISC-V detected")
         if (GGML_RVV)
-            list(APPEND ARCH_FLAGS -march=rv64gcv -mabi=lp64d)
+            list(APPEND ARCH_FLAGS -march=rv64gcv0p7 -mabi=lp64d)
         endif()
     elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "s390x")
         message(STATUS "s390x detected")
```

3. **compile**

* rv64gc

```shell
CC=riscv64-unknown-linux-gnu-gcc FC=riscv64-unknown-linux-gnu-gfortran cmake -B build -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=OpenBLAS -DGGML_RVV=OFF -DBLAS_INCLUDE_DIRS=[toolchain_directory]/riscv64-unknown-linux-gnu/sysroot/include  -DBLAS_LIBRARIES=[toolchain_directory]/riscv64-unknown-linux-gnu/sysroot/lib/libopenblas.so

cmake --build build --config Release -j4
```

* rv64gcv0p7

```shell
CC=riscv64-plctxthead-linux-gnu-gcc FC=riscv64-plctxthead-linux-gnu-gfortran cmake -B build -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=OpenBLAS -DGGML_RVV=ON -DBLAS_INCLUDE_DIRS=[toolchain_directory]/riscv64-unknown-linux-gnu/sysroot/include  -DBLAS_LIBRARIES=[toolchain_directory]/riscv64-unknown-linux-gnu/sysroot/lib/libopenblas.so

cmake --build build --config Release -j4
```

4. **move lib**

为了正确链接`libopenblas.so`，我们将`llama.cpp`编译生成的`libopenblas.so`，我们将`libopenblas.so`移动到`llama.cpp/build/bin`中

```shell
find [toolchain_directory]/riscv64-unknown-linux-gnu/sysroot \
-name "libopenblas*" | xargs -I {} cp {} ~/llama.cpp/build/bin
```

---

### llama-bench

通过llama-bench进行测试

```shell
cd llama.cpp/build/bin
./llama-bench -m [path/to/model/model-name] -t 4 --progress
```

* **rv64gc**

Result as following

[![asciicast](https://asciinema.org/a/wcG19iheLU8FudyspnOUdPbtB.svg)](https://asciinema.org/a/wcG19iheLU8FudyspnOUdPbtB)

​	`pp512` 测试中 `t/s=2.18` → 每秒处理 2.18 个 token

​	`tg128` 测试中 `t/s=0.6` → 每秒生成 0.6 个 token

* **rv64gcv0p7**

Result as following

[![asciicast](https://asciinema.org/a/r4xw3CBa6JI3uNcKLlDEhV0OR.svg)](https://asciinema.org/a/r4xw3CBa6JI3uNcKLlDEhV0OR)

​	`pp512` 测试中 `t/s=3.05` → 每秒处理 3.05个 token

​	`tg128` 测试中 `t/s=0.47` → 每秒生成 0.47 个 token

