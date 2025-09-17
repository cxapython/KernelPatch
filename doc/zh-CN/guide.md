# 使用指南（中文）

## 简介
KernelPatch 允许在无源码与无符号表的前提下对 Linux/Android 内核进行补丁与代码注入，支持内联函数 Hook 与 syscall 表 Hook，并提供可运行于内核态的 KPM 模块机制和用户态的 SuperCall 交互接口。Android 用户推荐配合 APatch 使用。

- **架构**: 仅支持 arm64
- **内核版本**: 理论 3.18 - 6.6
- **前置条件**: 目标内核需启用 `CONFIG_KALLSYMS=y`

## 组件概览
- `tools/` kptools：离线分析和给内核镜像打补丁、内嵌额外项（KPM/脚本等）
- `kernel/` kpimg：引导期接管并注入能力，导出 SuperCall
- `kpms/` KPM 示例：可在运行时按需加载的内核模块（非 LKM）
- `user/` SuperCall 头文件：在目标系统用户态使用的调用封装

## 环境准备
- 安装 arm64 裸机交叉编译器（aarch64-none-elf-）。
- 准备 Android NDK（如需在 Android 上编译用户态示例）。

## 构建步骤
### 1) 构建 kpimg（内核侧镜像）
```bash
export TARGET_COMPILE=aarch64-none-elf-
cd kernel
export ANDROID=1  # Android 版本（包含 su 相关能力）
make
```
构建产物：`kernel/kpimg`

### 2) 构建 kptools（离线补丁工具）
- 使用 Makefile：
```bash
export ANDROID=1
cd tools
make
```
- 或使用 CMake：
```bash
cd tools && mkdir -p build && cd build
cmake ..
make
```
构建产物：`tools/kptools`

### 3) 构建示例 KPM（以 hello 为例）
```bash
export TARGET_COMPILE=aarch64-none-elf-
cd kpms/demo-hello
make
```
构建产物：`kpms/demo-hello/hello.kpm`

## 将 KernelPatch 注入到内核镜像（离线打补丁）
使用 `kptools -p` 对目标内核镜像打补丁，并设置 superkey（SuperCall 的鉴权用）。可在同一次操作中把 `.kpm` 作为“额外项”内嵌，并配置触发事件与参数。

常用事件见 `kernel/include/preset.h`：
- `pre-kernel-init`（默认）
- `post-kernel-init`
- `pre-init-first-stage` / `post-init-first-stage`
- `pre-exec-init` / `post-exec-init`
- `pre-init-second-stage` / `post-init-second-stage`

示例：将 `hello.kpm` 以内嵌方式加入，并在 `pre-kernel-init` 触发，传入参数 `foo=bar`：
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
说明：
- `-i`：原始内核镜像
- `-k`：`kpimg`
- `-s`：superkey（必填）
- `-o`：输出镜像
- `-M`：嵌入一个“额外项”文件路径
- `-T`：额外项类型，KPM 用 `kpm`
- `-N`：额外项名（不指定时 KPM 会自动用模块自带 name）
- `-V`：触发事件
- `-A`：传给模块 init 的参数字符串（可选）

完成后将 `/path/to/boot-kernel.patched.img` 打包回设备（如 Android 的 boot.img），按设备流程刷入。

## 在设备上动态加载 KPM（不改内核镜像）
前提：设备已运行打好补丁的内核（即 KernelPatch 已安装并导出 SuperCall）。

- 用户态接口位于 `user/supercall.h`。你可以在自己的 App/CLI 中直接调用，也可以参考 `user_deprecated/` 内的示例 CLI（如 `kpm.c` 中的命令集合实现）。

常用调用：
- 加载模块：`sc_kpm_load(key, "/data/local/tmp/hello.kpm", "foo=bar", NULL)`
- 控制模块：`sc_kpm_control(key, "kpm-hello-demo", "your-ctl-args", out, outlen)`
- 卸载模块：`sc_kpm_unload(key, "kpm-hello-demo", NULL)`
- 列表/信息：`sc_kpm_list` / `sc_kpm_info`

其中 `key` 为打补丁时设置的 superkey。模块名来自 KPM 源码中的 `KPM_NAME("kpm-hello-demo")`。

## 在 APatch 上安装与运行 demo
APatch 是基于本项目的 Android 端解决方案。安装与使用通常有两种思路：

- 方式 A：离线把 KPM 内嵌到内核镜像，按事件自动触发（见上文 kptools -p）。
- 方式 B：设备运行后，使用 APatch 的用户态入口（例如其 Manager 或 CLI）通过 SuperCall 加载 `.kpm`。

由于 APatch 的具体 UI/CLI 形态可能会随版本更新，以下给出通用、版本无关的做法，确保可用：

1) 将 demo 模块拷贝到手机（举例）：
```bash
adb push kpms/demo-hello/hello.kpm /data/local/tmp/
adb shell chmod 644 /data/local/tmp/hello.kpm
```

2) 在手机上以具备权限的进程运行用户态加载（三选一）：
- 集成 `user/supercall.h` 到你的 App/CLI，调用 `sc_kpm_load("your-super-key", "/data/local/tmp/hello.kpm", "foo=bar", NULL)`
- 使用你已有的 APatch Manager/CLI（若提供）触发“加载模块”，并输入 superkey 与 KPM 路径
- 参考 `user_deprecated/` 的 `kpm` 子命令实现，自行编译一个简易 CLI：
  - 交叉编译（示例）：
    ```bash
    export ANDROID_NDK=/path/to/ndk
    cd user_deprecated
    mkdir -p build/android && cd build/android
    cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
          -DCMAKE_BUILD_TYPE=Release \
          -DANDROID_PLATFORM=android-33 \
          -DANDROID_ABI=arm64-v8a ../..
    cmake --build .
    ```
  - 推到设备后运行：
    ```bash
    adb push ./kpatch /data/local/tmp/
    adb shell "/data/local/tmp/kpatch kpm load /data/local/tmp/hello.kpm 'foo=bar'"
    ```

3) 验证模块是否成功：
- 通过 `dmesg` 或 `logcat` 查看 `pr_info` 输出（hello 模块会打印初始化日志和版本号）
- 使用列表/信息接口：
  - `sc_kpm_list(key, buf, len)`
  - `sc_kpm_info(key, "kpm-hello-demo", buf, len)`

4) 调用控制接口（对应 `hello.c` 中 `KPM_CTL0/CTL1`）：
- CTL0 带字符串参数并返回消息：
  - 例如：`sc_kpm_control(key, "kpm-hello-demo", "echo test", out, sizeof(out))`
- CTL1 为 3 个指针参数的控制入口（示例中仅打印）：
  - 需根据你的用户态封装设计提供对应接口

5) 卸载模块：
```c
sc_kpm_unload("your-super-key", "kpm-hello-demo", NULL);
```

## 常见问题
- 内核开不了机/崩溃：确认 `kpimg` 与目标内核版本/布局匹配；检查 `kptools` 输出与符号解析是否正常。
- SuperCall 返回鉴权失败：确认使用的 superkey 与打补丁时一致；或使用 su 机制并确保 UID 已授权。
- 无法加载 `.kpm`：检查路径权限与 SELinux，上层可先以调试目录 `/data/local/tmp` 验证；必要时通过 APatch 的策略开关放行。

## 参考
- `doc/en/build.md`, `doc/en/guide.md`
- 头文件：`user/supercall.h`、`kernel/include/preset.h`、`kernel/include/kpmodule.h`
- 示例：`kpms/demo-hello`、`kpms/demo-inlinehook`、`kpms/demo-syscallhook`
