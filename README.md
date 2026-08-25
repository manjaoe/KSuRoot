# KSuRoot

针对 Galaxy S23 Ultra 美版 `SM-S918U1` 的 KSuRoot 分支。最终 runner 基于 CVE-2026-43499 内核提权链，并在成功获取临时 root 后加载 KernelSU，同时安装 KernelSU Manager。

## 最终支持条件

当前内置 runner 只匹配以下设备和固件：

| 项目 | 值 |
|---|---|
| 型号 | `SM-S918U1` |
| 固件 | `BP4A.251205.006.S918U1UES8FZG1` |
| 内核 | `5.15.189-android13-8-33413713-abS918U1UES8FZG1` |
| `sched_blocked_reason` event id | `108` |

设备不匹配时不要运行本项目的 runner。

## 最终流程中的偏移量

### 当前美版 runner 是否使用 `offsets.h`

当前最终版 `SM-S918U1 runner` 使用的是已经编译好的 payload 资产。偏移量在 Payload 生成阶段使用，KSuRoot 安装时不再重新计算；内核位置确认和利用所需的逻辑已经包含在 `cve-2026-43499-app.so` 中。

`sched_blocked_reason` 的 event id `108` 是固件兼容性检查项，不是内核偏移量。当前 runner 还会在运行时输出 `locating-kernel`、`kernel-location-ready`、`temporary-root-ready` 等阶段，只有这些阶段和目标固件完全匹配时才继续 late-load。

偏移量只在 Payload 生成和验证阶段使用；最终 KSuRoot 安装时直接运行已经验证过的美版 Payload。

### Payload 文件生成与集成流程

这次美版 payload 的真实来源是“港版脚本 + 国行脚本”的移植流程：先对比两个已验证脚本的共同执行阶段、文件布局、helper 调用和 KernelSU late-load 顺序，再替换成美版 `SM-S918U1` 的版本信息、event id、偏移量和配套文件，形成 `S918U1 closed-candidate`。最后通过真机验证脚本，才把成功产物集成到 KSuRoot。

完整流程如下：

1. **以两个区域脚本为基线**：保留港版和国行脚本中已经验证的 exploit 启动顺序、预热、日志阶段、helper 参数、ksud 暂存和 `--late-load` 流程。它们是流程模板，不是可以直接用于美版的 payload；区域特有的型号、固件、内核、event id 和偏移量必须全部替换。
2. **生成美版版本目录**：为 `SM-S918U1` 建立独立目录，记录 `BP4A.251205.006.S918U1UES8FZG1`、完整 `uname -r`、event id `108`，并复制一份 closed-candidate 脚本作为美版入口。不要直接修改港版或国行原脚本，避免串用参数。
3. **从美版固件提取偏移量**：使用匹配版本的 `boot.img`、`xbl_config.elf`、`kallsyms.txt` 和可选的 `llvm-objdump`。使用 GhostLock-Galaxy 的 `extract_target.py` 或等价解析器，示例命令：

   ```powershell
   python .\extract_target.py `
     <matching-boot.img> `
     --xbl-config <matching-xbl_config.elf> `
     --kallsyms <matching-kallsyms.txt> `
     --format c `
     --out offsets-with-phys.h
   ```

   `--xbl-config` 负责从 XBL/FDT 的 MemoryMap 确认 `kernel_phys_load`；如果输出为 `null` 或没有唯一的 Kernel 区域，必须先修复 XBL/FDT 输入，不能继续生成 payload。使用 `--llvm-objdump` 时，还会对内核反汇编并推导 `pselect_waiter_shift` 和日志相关偏移。
4. **合并到美版设备配置**：检查生成的 `offsets-with-phys.h/.json`，确认 `kernel_phys_load`、kallsyms 符号、BTF 结构字段和反汇编推导值一致，再将确认结果合并到 payload 源码的 `src/devices/s918u1/offsets.h`，并保持 `src/devices/s918u1/target.h` 中的美版地址布局和结构体覆盖项。`offsets.h` 只参与 exploit payload 编译，不用于生成 helper、ksud 或 Manager APK。
5. **按 payload 工程的原生入口编译**：在 Payload 源码目录中使用其 Makefile 和 Android NDK，目标配置必须指向 `src/devices/s918u1/target.h`：

   ```powershell
   make clean
   make ghostlock helper TARGET_CONFIG=src/devices/s918u1/target.h
   ```

   目标必须是 Android `arm64-v8a`，并确认 Makefile 使用的 NDK 编译器和 `TARGET_CONFIG` 指向美版配置。实际成功的 closed-candidate 脚本还会把生成的 payload、helper、ksud 按固定名称暂存到 `/data/local/tmp/`，再执行预热和 `/system/bin/true` 验证；不能只完成本地编译就认为 payload 可用。
6. **按脚本输出组织文件**：将美版编译或脚本流水线输出对应到以下角色：

   ```text
   cve-2026-43499-app.so       exploit payload
   cve-2026-43499-root        root helper
   ksud-s25u-kdp               KernelSU daemon
   KernelSU_Manager_*.apk      KernelSU Manager
   ```

   `.so` 和 helper 必须来自同一次美版构建/验证，不能将港版或国行的 helper 与美版 `.so` 混用；`ksud` 还必须与目标 KernelSU late-load 方案匹配。
7. **脚本真机验证**：在完全匹配的美版设备上执行 closed-candidate 脚本，确认日志依次出现 `kernel-location-ready`、`temporary-root-ready` 和 `exploit completed`，随后确认 `--late-load` 成功并检查 `/proc/modules` 中的 `kernelsu`。发生重启或阶段缺失时，回查脚本参数、偏移量和配套版本，不要通过手工填充 `kernel_phys_load` 绕过检查。
8. **集成到 KSuRoot**：只有脚本在目标设备上验证成功后，才把四个最终文件放入 `app/src/main/assets/s918u1/`：

   ```text
   app/src/main/assets/s918u1/cve-2026-43499-app.so
   app/src/main/assets/s918u1/cve-2026-43499-root
   app/src/main/assets/s918u1/ksud-s25u-kdp
   app/src/main/assets/s918u1/KernelSU_Manager_v3.2.5_32525.apk
   ```

9. **重新构建并检查 APK**：

   ```powershell
   .\gradlew.bat clean assembleDebug
   & "$env:ANDROID_HOME\build-tools\36.0.0\zipalign.exe" -c -P 16 -v 4 .\app\build\outputs\apk\debug\app-debug.apk
   ```

10. **固化产物清单**：记录脚本版本、型号、固件、内核、event id、`offsets.h`/XBL/FDT 来源、四个资产的 SHA-256 和 APK SHA-256。版本变化时重新走两个区域脚本的对比、偏移量提取、美版脚本验证和资产集成流程，不能只替换一个 `.so`。

## 使用流程

1. 启动 Shizuku，并在 KSuRoot 请求权限时点击允许。
2. 安装本项目构建出的 APK。
3. 在主页的 Payload 来源中选择 **SM-S918U1 runner**。
4. 点击安装并确认。应用会自动检查型号、固件、内核和内置资产。
5. 应用通过 Shizuku 执行 400 次 `/system/bin/true` 预热。
6. 应用使用与成功脚本相同的 shell 命令触发 runner：

   ```text
   CVE43499_ROOT_HELPER=/data/local/tmp/cve-2026-43499-root \
   EXPLOIT_ATTEMPTS=1 \
   LD_PRELOAD=/data/local/tmp/cve-2026-43499 \
   /system/bin/true
   ```

7. 检测到 `temporary-root-ready` 和 `exploit completed` 后，应用暂存 ksud 并执行：

   ```text
   /data/local/tmp/cve-2026-43499-root --late-load
   ```

8. 检查 `kernelsu` 是否出现在 `/proc/modules`。
9. 通过 Shizuku 自动安装并启动 `KernelSU_Manager_v3.2.5_32525.apk`。

root 和 KernelSU 的这条链是内存态的。设备重启后状态会清除，需要重新打开 Shizuku，再从第 3 步执行。

## 内置资产

资产位于 `app/src/main/assets/s918u1/`，运行时会暂存到 `/data/local/tmp/`：

| 文件 | 用途 | SHA-256 |
|---|---|---|
| `cve-2026-43499-app.so` | 美版 runner payload | `B78E79C5D78B001F006B0B93E03BDA4D58F75100F261324BD4C732164F9907EA` |
| `cve-2026-43499-root` | root helper | `41886FF26677C7E2069A30495C2227DA9B3A4561FD40156AA6A9A993E035CB39` |
| `ksud-s25u-kdp` | KernelSU daemon | `11329C52ADF28130D75290BD095FB6831C0682F85817534A531C773933AEFE8E` |
| `KernelSU_Manager_v3.2.5_32525.apk` | KernelSU 管理器 | `1417081413BF7AB1DE8E440ECBCB62685037C8F28F048F0F8B79E305B31AB916` |

## 从源码构建

构建环境：

- JDK 21
- Android SDK Platform 37
- Android NDK `27.2.12479018`
- CMake `3.22.1`
- Android Gradle Plugin `9.2.1`

在项目目录执行：

```powershell
.\gradlew.bat assembleDebug
```

APK 输出位置：

```text
app/build/outputs/apk/debug/app-debug.apk
```

native 库使用 16 KB ELF page alignment；打包后可检查 APK ZIP 对齐：

```powershell
& "$env:ANDROID_HOME\build-tools\36.0.0\zipalign.exe" -c -P 16 -v 4 .\app\build\outputs\apk\debug\app-debug.apk
```

项目中 `app/src/main/cpp/CMakeLists.txt` 的 `-Wl,-z,max-page-size=16384` 不要删除，否则部分设备会提示未进行 16 KB 对齐。

## 项目结构

- `app/src/main/java/`：KSuRoot UI、设备匹配、Shizuku 执行和安装流程。
- `app/src/main/assets/s918u1/`：美版 S918U1 最终 runner 资产。
- `app/src/main/cpp/`：native probe，已配置 16 KB ELF 对齐。
- `app/src/main/jniLibs/arm64-v8a/libbs.so`：原有 Bundled 离线 payload。
- `app/build/outputs/apk/debug/`：最近一次构建产物，可通过 Gradle 重新生成。
- `artifacts/`：已验证的最终 APK 和 SHA-256 清单。

`artifacts/s9180-国行ZG1-root.zip` 和 `artifacts/s9180港版ZG1-root.zip` 是保留的港版/国行脚本包，里面包含对应的 `run_root_dm3q.ps1`、Payload、ksud 和 Manager，用于对照和复现脚本流程。

## 参考项目清理说明

最终版本已将美版 runner 所需的 payload、helper、ksud 和 Manager APK 固化到本项目中，运行时不依赖外部参考目录。

为避免重复文件和误用其他地区的源码，工作区中的以下本地参考项目已经删除：

- `Root-My-Galaxy`
- `Root-My-Galaxy-Payloads`
- `Root-My-Galaxy-SM-S918B`
- `S9180`

它们仅用于此前的源码结构、偏移量验证和美版 runner A/B 测试；港版/国行脚本本身保留在上述 ZIP 中。最终运行流程已整理并集成在本项目中。

## 故障排查

- 如果运行记录出现 `正在下载 payload`、`Payload: dm3q-S9180...` 或 `检查 GitHub 支持清单`，说明这次运行实际走了 Online 来源，不是美版 runner。重新安装包含 `assets/s918u1/` 的最新 APK，并确认运行日志出现 `使用已验证的美版 S918U1 runner 链`。
- `CANNOT LINK EXECUTABLE ... ld-linux-aarch64.so.1` 表示被执行的是在线 Payload 的动态链接依赖，不表示内置美版 Payload 缺失；先检查上一条来源判断，再不要继续重复运行同一个在线 Payload。
- `operation failed`：先确认设备没有已经加载 `kernelsu`，重启后再运行；不要在同一次启动中重复触发 exploit。
- Shizuku 失败：确认 Shizuku 服务运行，并在 KSuRoot 中允许权限。
- 16 KB 对齐错误：使用最新构建产物，并确认没有删除 CMake 的 16 KB linker 参数。
- Manager 未安装：查看日志中的 `KernelSU Manager auto-install failed`，确认 Shizuku shell 可以执行 `pm install`；root/KernelSU 已加载时可手动安装 `KernelSU_Manager_v3.2.5_32525.apk`。

## 免责声明

本项目仅用于自有设备的安全研究和学习。提权、内核利用和 KernelSU 加载可能导致数据丢失、系统不稳定、保修失效或设备损坏，请自行承担风险。

致谢：
[KSuRoot](https://github.com/hmascs/KSuRoot)
[Root-My-Galaxy](https://github.com/BuSung-dev/Root-My-Galaxy)
[GhostLock-Galaxy](https://github.com/wxxsfxyzm/GhostLock-Galaxy))
酷安@大尾巴狼__大佬提供的S23U港版和国行zg1版本脚本 - https://www.coolapk.com/feed/73320085?s=ODc0YjI1ZjMxNTU1ZDJnNmE4YzZjMzd6i1656

## 许可证

本项目沿用 Apache License 2.0，详见 [LICENSE](LICENSE)。
