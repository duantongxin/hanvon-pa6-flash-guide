# 汉王 PA6 / Hanvon PA6 (RK3566) 刷机与 Root 记录

> 一台 RK3566 墨水屏电纸书的完整 root 过程记录：从误判 BL 锁状态，到四条失败方案，
> 再到最终用官方固件包内 oot.img 配合 Magisk Alpha 成功 root。

[![device](https://img.shields.io/badge/device-Hanvon%20PA6%20(RK3566)-blue)]()
[![android](https://img.shields.io/badge/Android-11%20(SDK%2030)-green)]()
[![magisk](https://img.shields.io/badge/Magisk-Alpha%2031100-orange)]()

---
# 汉王 PA6（RK3566）刷机 / Root 全过程记录

> 设备：汉王 PA6（`HANVON_EBOOK`），Rockchip RK3566 + RK817，Android 11 / SDK 30
> 结论：**成功获得 Magisk Alpha root**，代价是内核被降级且不可回滚。
> 记录时间：2026-11-14

---

## 一、设备信息

| 项目 | 值 |
|---|---|
| 型号串 | `ro.product.model = PA6`，`ro.product.device = HANVON_EBOOK` |
| SoC | Rockchip **RK3566** + RK817 PMIC |
| fastboot product | `evb_rk3568`（板级配置名，**不是**真实型号） |
| 系统 | Android 11 / SDK 30 / `RQ2A.210505.003` |
| 构建 | `eng.hanvon.20260126.194718`（2026-01-26） |
| 内核 | `Linux 4.19.193+ #150 SMP PREEMPT Mon Jan 26 15:27:57 CST 2026` |
| 屏幕 | 6 英寸墨水屏 758×1024，212ppi |
| 内存 | 2 GB（1.95 GB 可用） |
| 存储 | Samsung eMMC 29.8 GB / 61,071,360 扇区 / 512B |
| 安全状态 | `ro.boot.verifiedbootstate = orange`、`ro.boot.flash.locked = 0` |

### 分区表（GPT，`rkdeveloptool ppt` 输出）

```
NO  LBA        Name
00  00004000   uboot
01  00006000   trust
02  00008000   waveform      ← 墨水屏波形
03  00009000   misc
04  0000B000   dtbo
05  0000D000   vbmeta
06  0000D800   boot          ← 32 MiB，跨 28.3–60 MB
07  0001D800   security
08  0001F800   recovery      ← 96 MiB
09  0004F800   backup
10  0010F800   cache
11  001CF800   metadata
12  001D7800   super         ← 动态分区
13  007EB800   logo
14  007F3800   userdata
```

---

## 二、第一个坑：BL 其实早就解锁了

`fastboot getvar unlocked` 返回 `no`，**这是假值**。

瑞芯微 U-Boot 没有实现该变量（AOSP `fb_getvar` 里 `unlocked` 是弱符号，默认返回 `"no"`），`secure: yes` 同理是常量。

**权威判据只能从 Android 侧读：**

```sh
adb shell getprop ro.boot.verifiedbootstate   # orange = 已解锁
adb shell getprop ro.boot.flash.locked        # 0 = 未锁定
```

因此：

```sh
fastboot flashing unlock      # 返回 OKAY，但什么都没变
fastboot oem unlock           # remote: not implemented
fastboot oem unlock-go        # remote: unknown oem command
fastboot flashing unlock_critical  # 同样 OKAY，无效
```

以上全是**空操作**，不是失败。当时在这里浪费了大量时间。

---

## 三、四条走不通的取 boot.img 路线

### ① 全网找固件 —— 没有

- 汉王官网下载区只有 T618 / N510 等 2010 年代机型
- `PA6`、`HANVON_EBOOK` 无任何公开固件索引
- 设备内**无任何 OTA 客户端**（priv-app 与 `pm list packages -s` 均无 updater/FOTA）
- `download.hw99.com/hanwang/ebook/` 返回 403，目录不可遍历

### ② Loader (rockusb) 读分区 —— 32 MiB 硬限制

先要打通驱动。Rockchip 老驱动（`rockusb.sys` v3.7 / v3.5）只支持到 PID `320A`，不认 RK3566 的 `350A`。

**解决方案：自签证书 + 目录签名 + WinUSB INF**

```ini
[Version]
Signature="$Windows NT$"
Class=USBDevice
ClassGuid={88BAE032-5A81-49F0-BC3D-A4FF138216D6}
CatalogFile=rk350a.cat

[Mfr.NTamd64]
%DEVNAME%=WinUSB_Install, USB\VID_2207&PID_350A

[WinUSB_Install]
Include=winusb.inf
Needs=WINUSB.NT
```

```powershell
# 自签代码签名证书
$cert = New-SelfSignedCertificate -Type CodeSigningCert -Subject "CN=DSH Driver Signing" `
        -CertStoreLocation Cert:\LocalMachine\My
# 加入信任存储
$s = New-Object System.Security.Cryptography.X509Certificates.X509Store("Root","LocalMachine")
$s.Open("ReadWrite"); $s.Add($cert); $s.Close()
# 生成并签名目录
New-FileCatalog -Path $pkg -CatalogFilePath "$pkg\rk350a.cat" -CatalogVersion 2.0
Set-AuthenticodeSignature -FilePath "$pkg\rk350a.cat" -Certificate $cert -HashAlgorithm SHA256
# 安装
pnputil /add-driver "$pkg\rk350a_winusb.inf" /install
```

> 未签名的 INF 会被拒绝：`第三方 INF 不包含数字签名信息`。

驱动打通后 `rkdeveloptool ld` 成功：`Vid=0x2207,Pid=0x350a Loader`。

**但读取有 32 MiB 上限：**

```
rcb → Read Secure Mode: enabled

LBA 0xFFFF (32MiB-1)  → 真实数据
LBA 0x10000 (32MiB)   → CC CC CC（伪造填充，工具仍报 100% 成功）
```

boot 分区从 28.3 MB 开始，只能读到 32 MB 为止（3.7 MB 真实数据）。

> ⚠️ 注意：`cc` 填充会让人误以为分区是空的。曾据此错误判断"ramdisk 不存在"，后已推翻。

### ③ fastbootd `fetch` —— Android 11 不支持

```sh
adb reboot fastboot
fastboot getvar is-userspace     # yes，确实进了 fastbootd
fastboot fetch boot boot.img
# → Unable to get max-fetch-size. Device does not support fetch command.
```

`fetch` 是 Android 12 才加入 fastbootd 的功能。

> 附带经验：`fastboot`/`adb` 在 Windows 上按 **接口 GUID** 枚举设备，必须设成
> `{F72FE0D4-CBCB-407d-8814-9ED673D0DD6B}`，否则即使驱动装好也认不到。

### ④ MaskROM —— BootROM 不支持读

`rkdeveloptool rd 3`（`RST_RESETMASKROM_SUBCODE`）可软件复位进 MaskROM，`ld` 显示 `Maskrom`。

但：

| 命令 | 结果 |
|---|---|
| `rfi` | `Read Flash Info failed!` |
| `rid` | 30s 超时 |
| `rl 0xD800 8`（Loader 下可读的低地址） | 25s 超时 |
| `rl 0x10000 8` | 25s 超时 |

**连低地址都读不出** → BootROM 根本没实现 READ_LBA。MaskROM 的命令集只用于救砖刷写，不含读取。

### 其他失败尝试

- **recovery root shell**：`adb reboot recovery` 能进（"no command" 界面），但 adbd 报 `unauthorized`——recovery 无确认对话框，`/data` 未挂载拿不到 `adb_keys`。
- **Magisk App 修补**：`invalid cpio magic`（因为拿到的是被 32MiB 截断的假镜像）。
- **`fastboot boot`**：返回 `Booting OKAY`，但设备重新枚举回 fastboot，未真正启动——U-Boot 无内存启动能力。

---

## 四、最终方案

### 前提：官方整包 `update.zip`

```
1,176,707,007 B（1.10 GB）
ota-type = BLOCK
pre-device = HANVON_EBOOK
```

内含 `boot.img`（32,049,152 B）、`dtbo.img`、`uboot.img`，以及五个 `.new.dat.br`。

### 致命细节：包比设备旧 8 个月

| | 构建号 | 内核 |
|---|---|---|
| 包内 | `eng.hanvon.20250516.141854`（2025-05-16） | `#72` |
| 设备 | `eng.hanvon.20260126.194718`（2026-01-26） | `#150` |

升级脚本第一行就是反降级断言：

```sh
(!less_than_int(1747376331, getprop("ro.build.date.utc"))) || \
  abort("E3003: Can't install this package over newer build.")
```

设备 `ro.build.date.utc = 1769428005 > 1747376331` → **整包 OTA 被直接拒绝**。

所以只能**单独使用包内的 `boot.img`**，接受内核降级。

### 1. 校验包内 boot.img

```
magic        ANDROID!
kernel_size  30,951,432
ramdisk_size   762,052      gzip 压缩，真实数据
second_size    208,384
page_size        2,048
header_ver           2
```

### 2. Magisk Alpha 命令行修补

无需 App 点按，直接把工具链推到设备执行：

```sh
# 从 Magisk APK 提取：
#   lib/arm64-v8a/libmagiskboot.so → magiskboot
#   lib/arm64-v8a/libmagiskinit.so → magiskinit
#   lib/arm64-v8a/libmagisk.so     → magisk
#   lib/arm64-v8a/libinit-ld.so    → init-ld
#   lib/arm64-v8a/libbusybox.so    → busybox
#   assets/boot_patch.sh, assets/util_functions.sh

cd /data/local/tmp/magisk
KEEPVERITY=true KEEPFORCEENCRYPT=true MAGISKBIN=$PWD \
  sh boot_patch.sh /data/local/tmp/boot_pkg.img
```

结果：

```
ramdisk 762,052 → 790,177   (+28,125)
CHECKSUM 已重算，内核字节不动
```

### 3. 刷入

```sh
adb reboot bootloader
fastboot flash boot new-boot.img     # 32,075,776 B
fastboot reboot
```

```
Sending 'boot_a' (31324 KB)  OKAY
Writing 'boot_a'             OKAY
```

### 4. 验证

```
sys.boot_completed        = 1
kernel                    = 4.19.193+ #72   ← 已降级
ro.boot.verifiedbootstate = orange
ps  → magiskd (uid 0)
magisk -v                 → 0972f67d-alpha:MAGISK:R
```

### 5. 授权 su

`adb shell su` 默认被 Magisk 拒绝（shell 的默认策略是拒绝）。

**Magisk → 超级用户 → Shell → 允许**，之后：

```
$ adb shell su -c id
uid=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
```

---

## 五、最终状态

| 项 | 状态 |
|---|---|
| Root | ✅ Magisk Alpha 31100 |
| 内核 | ⚠️ `#72`（2025-05），与 2026-01 系统版本不匹配 |
| BL | 保持解锁 |
| 稳定性 | 首次刷后短暂掉线，断电重启后稳定运行 |
| 回滚 | 只能用包内原版 `boot.img`（同为 `#72`），**无法恢复原厂 `#150` 内核** |

### 附带完成的配置

```sh
# 省电（持久化到 /data/adb/service.d/99-powersave.sh）
echo schedutil > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
echo 1104000   > /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq
echo 400000000 > /sys/class/devfreq/fde60000.gpu/max_freq
echo 528000000 > /sys/class/dmc/max_freq
settings put global window_animation_scale 0
settings put global transition_animation_scale 0
settings put global animator_duration_scale 0

# Scene 悬浮窗
adb shell appops set com.omarea.vtools SYSTEM_ALERT_WINDOW allow
```

---

## 六、经验教训

1. **先确认真实锁状态**，只看 `ro.boot.verifiedbootstate` 与 `ro.boot.flash.locked`，不要信 `getvar unlocked`。
2. **固件包是唯一正路**。国内电纸书的第三方固件在公网基本不存在，直接找客服。
3. **rockusb 的读不可信**：`> 32 MiB` 返回 0xCC 且报成功；`wl`（写）不受限，`rl`（读）受限。
4. **`gpt` 是写分区表，`ppt` 才是打印**，一个字母之差，操作前务必看清 usage。
5. **Windows 驱动可以自签**：自签证书 + `New-FileCatalog` + `Set-AuthenticodeSignature` + `pnputil`，不需要 Zadig。
6. **版本必须对齐**。内核差 8 个月能启动，但属于踩线运行，驱动模块 vermagic 不一致。
7. **刷 boot 前必须有回滚手段**。这次原厂内核不可恢复地丢失了。

---

## 七、复现速查

```sh
# 1. 确认解锁
adb shell getprop ro.boot.verifiedbootstate      # 必须 orange

# 2. 从官方整包取匹配版本的 boot.img
unzip update.zip boot.img

# 3. Magisk 修补
KEEPVERITY=true KEEPFORCEENCRYPT=true MAGISKBIN=$PWD \
  sh boot_patch.sh boot.img

# 4. 刷入
adb reboot bootloader
fastboot flash boot new-boot.img
fastboot reboot

# 5. 授权
# Magisk → 超级用户 → Shell → 允许
```

