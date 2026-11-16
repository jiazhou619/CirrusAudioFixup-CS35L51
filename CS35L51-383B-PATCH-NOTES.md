# CS35L51 / Lenovo Yoga Slim 7 Carbon 14ACN6（82L0）适配改动说明

本分支（`cs35l51`）在 CS35L51 基础支持之上，针对 **Lenovo Yoga Slim 7 Carbon 14ACN6（机型 82L0，AMD Ryzen 7 5800U）** 完成了实机可用的适配。本文件记录全部代码改动及原因。

## 硬件背景

| 项目 | 值 |
| --- | --- |
| 机型 | Lenovo Yoga Slim 7 Carbon 14ACN6 / 82L0 |
| 平台 | AMD Ryzen 7 5800U（Renoir） |
| 声卡 | Realtek ALC287，HDA PCI `pci1022,15e3` |
| 主板 PCI SSID | `0x17AA383B`（本机实际值） |
| Codec SSID | `0x17AA3856` |
| 功放 ACPI HID | `CLSA0102` |
| 功放芯片 | Cirrus CS35L51（Halo 系列，与 CS35L41 共用寄存器映射/DSP 启动流程） |
| I2C 控制器 | AMD `AMDI0010`，地址 `0x40` / `0x41` |
| 固件来源 | 从本机 Windows 驱动提取（`halo_cspl_RAM_revB2_29.49.0.wmfw` + `17AA383D_20210811_A0.bin`） |

上游原版按 `0x17AA383D` 这个 SSID 适配，而本机上报的是 `0x17AA383B`。此外发现两个实机问题：音量为保守默认值、偶发 I2C 读错误会永久锁存功放导致低音丢失。以下 5 处改动逐一解决。

## 改动清单

### 1. `Platform/Board/PlatformProfile.hpp` — 授权 383B SSID

让板级探测同时接受 `383D` 和本机的 `383B`：

```cpp
// 改前
(!strcmp(profile.acpiHid, "CLSA0102") && pciSubsystemId == 0x17AA383D)
// 改后
(!strcmp(profile.acpiHid, "CLSA0102") && (pciSubsystemId == 0x17AA383D || pciSubsystemId == 0x17AA383B))
```

不这样做的后果：`Cirrus_Platform_Authorized = False`，驱动直接跳过，`Amp_Discovery = NONE`。

### 2. `Devices/Amplifiers/CS35L41/Board/BoardProperties.hpp` — 新增 383B 板级表项

```cpp
{0x17aa383b, 2, false, {0, 1, 0, 0}, 0, -1, -1, 0, 0, 0},
```

声明本机有 2 颗功放、I2C 索引映射 `{0,1,0,0}`。不这样做则 `Cirrus_Board_Authorized = False`。

### 3. `Devices/Amplifiers/CS35L41/CS35L41Firmware.cpp` — SSID quirk 映射

在 `kSsidQuirks` 中把 383B 映射到已验证可用的 383D 固件集：

```cpp
{0x17AA, 0x383B, 0x17AA, 0x383D},
```

固件本身就是从本机 Windows 提取的，因此复用 383D 的 WMFW/bin 是正确且安全的。

### 4. `Devices/Amplifiers/CS35L41/Resources/Generated/FirmwareData.inc` — 383B 固件条目 + 增益调音

新增 `0x17AA, 0x383B` 固件记录，复用 383D 的 wmfw / 左右声道 bin；同时新增一段 companion tuning，把 DSP 的 `pcmGain` 从保守默认 **17** 提到驱动允许的安全上限 **20**：

```cpp
const uint8_t cs35l41_17aa383b_gain20_tuning[] = {
    0x35, 0x4a, 0x9a, 0x10,   // magic 0x109A4A35
    0x01, 0x00, 0x00, 0x00,   // version 1
    0x20, 0x00, 0x00, 0x00,   // size 32
    0x01, 0x00, 0x00, 0x00,   // entries 1
    0x00, 0x00, 0x00, 0x00,   // index 0
    0x00, 0x00, 0x00, 0x00,   // type 0 (gain)
    0x10, 0x00, 0x00, 0x00,   // recordSize 16
    0x14, 0x00, 0x00, 0x00    // gain = 20
};
```

- 数据格式遵循 `Tuning.hpp` 的解析规则（16 字节头 + 一条 type=0 的 gain record）与 Linux `cs35l41_hda.c` 一致。
- 20 是驱动解析明确允许的上限（`gain > 20` 会被直接拒绝），在官方安全范围内。
- 生效后运行时可见 `Cirrus_Tuning_Source = EMBEDDED_BINCFG`、`Cirrus_Tuning_PCM_Gain = 20`。

### 5. `Devices/Amplifiers/CS35L41/CS35L41Playback.cpp` — I2C 瞬态故障自愈

**问题**：实机偶发一次 I2C 读错误（如 2478 次读里错 1 次，`FailureReg=0x10098`），驱动会将功放锁存为 `FAULT_LATCHED_OUTPUT_DISABLED` 并关闭输出，导致低音永久消失，只能重启恢复。根因是原版恢复逻辑只认 `DIAG_PLL_UNLOCKED` 一种故障，`I2C_TRANSFER` 故障不在白名单里，直接进入 `else { return; }`。

**修复**（3 处，仅改恢复判定，不碰启动/初始化路径）：

1. route 分支的 `latestFailure` 判定加入 `DIAG_I2C_TRANSFER`。
2. 引入 `recoverableFailure = (PLL_UNLOCKED || I2C_TRANSFER)`，恢复入口条件改为 `(pllRecoveryPending || recoverableFailure) && recoverableFailure`。
3. 故障记录处 `pllRecoveryPending` 在 `I2C_TRANSFER` 时也置位。

重试有 `pllRecoveryAttempts < 3` 上限防抖，恢复成功后 `commitCS35L41Playback` 会把计数归零，不影响同一轮之外的新故障。

## 配套的非代码改动（在 EFI 侧）

代码之外，实机还需要两项配置，详见 EFI 仓库（`EFI-for-5800u`）的 `AUDIO-FIX-USAGE.md`：

- boot-args 去掉 `-cirrusnodsp`，让驱动以 DSP 模式运行（否则是 DAC 旁路，无增益、无补偿）；
- 一个 LaunchAgent 守护进程，在每次登录/唤醒后重放 HDA verb 激活 Node `0x17`（底层低音单元 pin）。本机 `alcid=21` 的 layout 默认没有激活该 node，导致 codec 路由不通、功放拿不到 BCLK/I2S 流。

## 验证结果

编译 `BUILD SUCCEEDED`，vtable 校验 4 类 PASS，回归测试 62 项 PASS（`check_boot_evidence` 为仓库既有的已知失败项，与本次改动无关）。实机重启后运行时状态：

```
Driver_Verdict       = READY_DSP
Amplifier_Status     = READY / PLAYING
DSP_BOOT_VERDICT     = HEALTHY (i2c0 / i2c1)
Tuning_Source        = EMBEDDED_BINCFG
Tuning_PCM_Gain      = 20
Read_Fail / Write_Fail = 0
```

低音恢复，音量提升；偶发 I2C 抖动现在可自动恢复，不再锁死。

## 版本对照（kext 二进制 SHA256 前缀）

| SHA 前缀 | 说明 |
| --- | --- |
| `992250e5` | 最初的 DEVID 白名单版（EFI 仓库历史版本） |
| `5f6c33fd` | 加入 383B 三处授权/固件改动 |
| `38e80927` | 加入 gain20（**有 I2C 锁死缺陷，不要回退到此版**） |
| `087e0b69` | 当前版本：加入 I2C_TRANSFER 自愈（黄金版） |
