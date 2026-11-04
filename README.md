# kwebp

`kwebp` 是 Kangle Web Server 的 WebP 图片转换扩展。

## 编译要求

- Linux 或 Windows
- CMake 3.12 或更高版本（部分发行版中的命令名为 `cmake3`）
- C99 编译器
- Kangle 开发头文件
- libwebp 及 JPEG、PNG、TIFF 开发库；启用 GIF 支持时还需 giflib 开发库

构建参数：

- `KANGLE_DIR`：Kangle **安装目录**，例如 `/vhs/kangle`。
- `KANGLE_SOURCE_DIR`：可选的 Kangle **源代码目录**。若源码位于 `kwebp` 同级的 `kangle` 目录，可省略该参数。
- `LIBWEBP_DIR`：指向 libwebp 源码目录，其 `build` 子目录中应已有编译产物。

## 编译 libwebp

```sh
git clone https://github.com/webmproject/libwebp.git
cd libwebp
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON
cmake --build build -j
cmake --build build --target install
```

CentOS 7 下可在上述 libwebp 配置命令中添加 `-DWEBP_ENABLE_SIMD=OFF`。
该选项关闭 libwebp 的 SIMD 优化，解决编译时出现 `_mm256_cvtsi256_si32` 未定义的链接错误。

## 编译和安装 kwebp

```sh
git clone https://github.com/netcccyun/kwebp.git
cd kwebp

cmake -S . -B build \
  -DKANGLE_DIR=/vhs/kangle \
  -DKANGLE_SOURCE_DIR=/path/to/kangle-source \
  -DLIBWEBP_DIR=/path/to/libwebp \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
cmake --build build --target install
```

安装完成后应生成：

- `/vhs/kangle/bin/kwebp.so`
- `/vhs/kangle/ext/kwebp.dso.xml`

最后重启或重载 Kangle，使扩展配置生效。
