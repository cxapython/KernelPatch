# KPM 快速入门：生成与安装到手机

本文聚焦两件事：
- 如何在本项目中生成一个 `.kpm` 模块（以 `kpms/demo-hello` 为例）
- 如何把生成的 `.kpm` 安装/加载到手机（APatch/KernelPatch 已运行）

## 一、生成 .kpm（以 hello.kpm 为例）

### 1. 准备交叉编译器
- 推荐使用 Homebrew 安装：
```bash
brew install aarch64-elf-gcc
```
- 注意：示例 `Makefile` 期望 `TARGET_COMPILE` 前缀存在。
  - 若安装的是 `aarch64-elf-gcc`，请使用 `TARGET_COMPILE=aarch64-elf-`
  - 若使用 Arm 官方工具链（`aarch64-none-elf-gcc`），请使用 `TARGET_COMPILE=aarch64-none-elf-`

### 2. 编译 hello.kpm
从仓库根目录执行：
```bash
export KP_DIR=$(pwd)
export TARGET_COMPILE=aarch64-elf-   # 或 aarch64-none-elf-
cd kpms/demo-hello
make
```
编译成功后，得到：
- `kpms/demo-hello/hello.kpm`

## 二、安装/加载到手机（APatch/KernelPatch）

前置条件：
- 目标设备已刷入/运行一个包含 KernelPatch 的内核（或已安装 APatch 解决方案）
- 已知打补丁时设置的 `superkey`（用于用户态 SuperCall 鉴权）
- 设备已开启 USB 调试并能通过 `adb devices` 识别

### 方案 A：运行时动态加载（推荐调试）
1) 推送 `.kpm` 到手机：
```bash
adb push kpms/demo-hello/hello.kpm /data/local/tmp/
adb shell chmod 644 /data/local/tmp/hello.kpm
```
2) 通过用户态 SuperCall 加载 `.kpm`：
- 方式 1：在你的 App/CLI 内集成 `user/supercall.h`，直接调用：
```c
// key 为打补丁时设置的 superkey
sc_kpm_load(key, "/data/local/tmp/hello.kpm", "foo=bar", NULL);
```
- 方式 2：构建仓库内的简易 CLI（示例在 `user_deprecated/`，目标是生成 `kpatch` 可执行文件）：
```bash
# 需要 Android NDK 与 CMake
export ANDROID_NDK=/path/to/ndk
cd user_deprecated
mkdir -p build/android && cd build/android
cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
      -DCMAKE_BUILD_TYPE=Release \
      -DANDROID_PLATFORM=android-29 \
      -DANDROID_ABI=arm64-v8a ../..
cmake --build .
# 推送与加载
adb push ./kpatch /data/local/tmp/
adb shell /data/local/tmp/kpatch kpm load /data/local/tmp/hello.kpm "foo=bar"
```
3) 验证：
- 查看内核日志（`pr_info`）：`adb shell dmesg | grep "kpm hello"` 或 `logcat`
- 查询模块：
  - C API：`sc_kpm_list(key, buf, len)` / `sc_kpm_info(key, "kpm-hello-demo", buf, len)`
  - CLI：`/data/local/tmp/kpatch kpm list` / `... kpm info kpm-hello-demo`
4) 调用控制接口：
- CTL0（字符串入参并回显）：
  - C API：`sc_kpm_control(key, "kpm-hello-demo", "echo test", out, sizeof(out))`
  - CLI：`kpatch kpm ctl0 kpm-hello-demo "echo test"`
5) 卸载模块：
- C API：`sc_kpm_unload(key, "kpm-hello-demo", NULL)`
- CLI：`kpatch kpm unload kpm-hello-demo`

### 方案 B：内嵌到内核镜像（开机按事件自动加载）
使用 `tools/kptools` 将 `hello.kpm` 以内嵌“额外项”的方式写入内核镜像，并配置自动触发事件：
```bash
cd tools
./kptools \
  -p \
  -i /path/to/boot-kernel.img \
  -k ../kernel/kpimg \
  -s your-super-key \
  -o /path/to/boot-kernel.patched.img \
  -M ../kpms/demo-hello/hello.kpm \
  -T kpm \
  -N hello \
  -V pre-kernel-init \
  -A "foo=bar"
```
将输出镜像刷回设备后，模块会在设定的事件点（如 `pre-kernel-init`）自动加载。

> 常见事件在 `kernel/include/preset.h` 中定义：`pre-kernel-init`（默认）、`post-kernel-init`、`pre-init-first-stage`、`post-init-first-stage`、`pre-exec-init`、`post-exec-init`、`pre-init-second-stage`、`post-init-second-stage`。

## 常见问题
- 找不到交叉编译器：用 `brew install aarch64-elf-gcc`，并设置 `TARGET_COMPILE=aarch64-elf-`
- 加载失败/鉴权错误：检查 `superkey` 是否与打补丁时一致；或确认调用方 UID 是否已被 APatch/KernelPatch 允许
- 无日志输出：确认 `dmesg`/`logcat` 读取权限；必要时降低 SELinux 限制或使用开发目录 `/data/local/tmp`

## 参考
- 示例源码：`kpms/demo-hello/hello.c`
- 用户态头文件：`user/supercall.h`
- 工具：`tools/kptools`
- 英文文档：`doc/en/build.md`, `doc/en/guide.md`
