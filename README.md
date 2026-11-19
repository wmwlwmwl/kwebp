# kwebp

`kwebp` 是 Kangle Web Server 的 WebP 图片转换扩展。

## 编译要求

- Linux 或 Windows
- CMake 3.12 或更高版本，并满足所选 libwebp 源码的 CMake 版本要求（部分发行版中的命令名为 `cmake3`）
- C99 编译器
- Kangle 开发头文件
- Linux 需要 libwebp 源码和 JPEG、PNG、TIFF 开发库。启用 GIF 支持时还需 giflib 开发库。

构建参数：

- `KANGLE_DIR`：Kangle **安装目录**，例如 `/vhs/kangle`。
- `KANGLE_SOURCE_DIR`：可选的 Kangle **源代码目录**。若源码位于 `kwebp` 同级的 `kangle` 目录，可省略该参数。
- `LIBWEBP_DIR`：Linux 下指向 libwebp 源码目录，kwebp 构建时会一并编译。

## Windows 编译

先运行相邻 Kangle 项目的 `build-windows.ps1` 生成开发头文件。在 kwebp 工程目录运行 `.\build-windows.ps1`，默认生成 `build/Release/kwebp.dll`。

脚本默认使用 Kangle 源码中 `deps/windows-x64` 的 libwebp 和 Kangle 的 `build/install`。可通过 `-BuildDirectory`、`-DependencyRoot` 或 `-KangleDirectory` 指定对应目录。

## Linux 编译和安装 kwebp

构建会把指定的 libwebp 源码静态链接进 `kwebp.so`。

```sh
git clone https://github.com/webmproject/libwebp.git
git clone https://github.com/netcccyun/kwebp.git
cd kwebp

cmake -S . -B build \
  -DKANGLE_DIR=/vhs/kangle \
  -DKANGLE_SOURCE_DIR=/path/to/kangle-source \
  -DLIBWEBP_DIR=../libwebp \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
cmake --build build --target install
```

CentOS 7 下若遇到 `_mm256_cvtsi256_si32` 未定义的链接错误，可给上述 CMake 配置命令添加 `-DWEBP_ENABLE_SIMD=OFF`，关闭 libwebp 的 SIMD 优化。

安装完成后应生成：

- `/vhs/kangle/bin/kwebp.so`
- `/vhs/kangle/ext/kwebp.dso.xml`

最后重启或重载 Kangle，使扩展配置生效。
