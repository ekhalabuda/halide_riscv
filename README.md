# Halide language experiments on RISC-V

This repository contains a number of examples written in [Halide](https://github.com/halide/Halide) language which work on RISC-V CPU with RVV 0.7.1.
Follow the steps below to reproduce the experiments or some of their parts.

**NOTE**: At this moment project relies on the Ahead-Of-Time (AOT) compilation of Halide kernels. However there are precompiled files for easy start.

Project uses [OpenCV](https://github.com/opencv/opencv/) as a reference implementation for some algorithms.
Also, we benchmark it to compare with Halide implemention.
Fetch OpenCV source code by submodules update after clonning the repository.

```bash
git submodule update --init
```

## Build and go

1. Download THead toolchain: [Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.1-20220906.tar.gz](https://occ.t-head.cn/community/download?id=4090445921563774976) (registration needed)
2. Clone Halide source code (no build required)

    ```bash
    git clone --depth 1 https://github.com/halide/Halide
    ```

3. Build a project for RISC-V CPU:

    ```bash
    export PATH=$HOME/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.1/bin/:$PATH

    cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_TOOLCHAIN_FILE=$HOME/halide_riscv/riscv64-071.toolchain.cmake \
        -DHalide_INCLUDE_DIRS=$HOME/Halide/src/runtime \
        -S halide_riscv -B build_rv64

    cmake --build build_rv64 -j$(nproc --all)
    ```

4. Transfer build directory to the RISC-V board and run `test_algo` (accuracy tests) or `perf_algo` (performance tests).

    ```bash
    scp test_algo perf_algo libalgos.so opencv-prefix/src/opencv-build/lib/* sipeed@x.x.x.x:/home/sipeed/
    ```

    ```bash
    export LD_LIBRARY_PATH=./
    ./test_algo
    ./perf_algo
    ```

## Performance results

HW: Sipeed Lichee RV Dock (Allwinner D1 aka XuanTie C906 CPU)

OS: [20211230_LicheeRV_debian_d1_hdmi_8723ds](https://mega.nz/folder/lx4CyZBA#PiFhY7oSVQ3gp2ZZ_AnwYA/folder/xtxkABIB)

<table>
    <thead>
        <tr>
            <th>Algorithm</th>
            <th>Input</th>
            <th>OpenCV (no RVV)</th>
            <th>OpenCV (RVV)</th>
            <th>Halide (no RVV)</th>
            <th>Halide (RVV)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td rowspan=2>BGR2Gray</td>
            <td>1080x1920x3 (interleaved)</td>
            <td>32.18ms</td>
            <td>264.66ms</td>
            <td>34.18ms</td>
            <td>30.78ms</td>
        </tr>
        <tr>
            <td>1080x1920x3 (planar)</td>
            <td>--</td>
            <td>--</td>
            <td>38.13ms</td>
            <td>6.65ms</td>
        </tr>
        <tr>
            <td>Box filter</td>
            <td>input: 1080x1920<br>output: 1078x1918</td>
            <td>75.17ms</td>
            <td>139.16ms</td>
            <td>198.02ms</td>
            <td>62.89ms</td>
        </tr>
        <tr>
            <td>Histogram</td>
            <td>input: 1080x1920x3<br>output: 256x3</td>
            <td>57.35ms</td>
            <td>67.32ms</td>
            <td>92.44ms</td>
            <td>--</td>
        </tr>
        <tr>
            <td rowspan=2>Convolution<br>input: 1x16x128x128<br>kernel: 32x16x3x3<br>stride: 1, pad: 0</td>
            <td>(FP32) NCHW</td>
            <td>829.13ms</td>
            <td>338.02ms</td>
            <td>4713.57ms</td>
            <td>698.27ms</td>
        </tr>
        <tr>
            <td>(FP32) NHWC</td>
            <td>--</td>
            <td>--</td>
            <td>1357.81ms</td>
            <td>418.95ms</td>
        </tr>
    </tbody>
</table>

## Generate AOT kernels

If you want regenerate AOT artifacts or add new algorithms, build the project on x86:

1. Build LLVM from https://github.com/dkurt/llvm-rvv-071/tree/rvv-071

    ```bash
    cmake -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_PROJECTS="clang;lld" \
      -DLLVM_TARGETS_TO_BUILD="RISCV" \
      -DLLVM_ENABLE_TERMINFO=OFF -DLLVM_ENABLE_ASSERTIONS=ON \
      -DLLVM_ENABLE_EH=ON -DLLVM_ENABLE_RTTI=ON -DLLVM_BUILD_32_BITS=OFF \
      -GNinja \
      -S llvm-project/llvm -B llvm-build

    cmake --build llvm-build -j4
    ```

2. Build Halide with the following patch (tested on revision https://github.com/halide/Halide/commit/7963cd4e3c23856b82567c99e0a3d16035ffe895):

    ```patch
    diff --git a/src/CodeGen_RISCV.cpp b/src/CodeGen_RISCV.cpp
    index ba9abe04d..454558d11 100644
    --- a/src/CodeGen_RISCV.cpp
    +++ b/src/CodeGen_RISCV.cpp
    @@ -151,6 +151,7 @@ string CodeGen_RISCV::mattrs() const {
                arch_flags += ",+zvl" + std::to_string(target.vector_bits) + "b";
            }
    #endif
    +        arch_flags += ",-zve64x";
        }
        return arch_flags;
    }
    diff --git a/src/autoschedulers/CMakeLists.txt b/src/autoschedulers/CMakeLists.txt
    index 9b88f0a66..10088bb9b 100644
    --- a/src/autoschedulers/CMakeLists.txt
    +++ b/src/autoschedulers/CMakeLists.txt
    @@ -24,6 +24,6 @@ endfunction()

    add_subdirectory(common)

    -add_subdirectory(adams2019)
    +# add_subdirectory(adams2019)
    add_subdirectory(li2018)
    add_subdirectory(mullapudi2016)
    ```

    ```bash
    export LLVM_ROOT=$HOME/llvm-build

    cmake -DLLVM_DIR=$LLVM_ROOT/lib/cmake/llvm \
        -DClang_DIR=$LLVM_ROOT/lib/cmake/clang \
        -DCMAKE_BUILD_TYPE=Release \
        -DWITH_TESTS=OFF \
        -DWITH_TUTORIALS=OFF \
        -DWITH_PYTHON_BINDINGS=OFF \
        -S Halide -B halide-build

    cmake --build halide-build -j4
    cmake --install halide-build --prefix halide-install
    ```

3. Build a project on x86

    ```bash
    export LD_LIBRARY_PATH=$HOME/halide-build/src/autoschedulers/mullapudi2016:$LD_LIBRARY_PATH

    cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DHalide_DIR=$HOME/halide-install/lib/cmake/Halide \
        -S halide_riscv -B build

    cmake --build build -j$(nproc --all)
    ```

4. Run `perf_algo` once and find the generated `*.h` and `*.s` files in the working directory.
