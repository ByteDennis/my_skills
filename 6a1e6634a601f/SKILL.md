---
id: 6a1e6634a601f
name: Disk Exhaustion: ext4 Corruption and SATA I/O Failures
tags: [linux, disk, diagnostics]
updated_at: 2026-06-06T02:18:09.021756Z
---

## 场景

挂载点 `/mnt/d5179bee-9063-4651-a04b-3303a5304da7`（`/dev/sda`，**Seagate ST8000DM004 8TB SATA 内置盘**）报「写满 / No space left on device」。

**结论：盘没真满。** `df` 显示只用了 55%（3.8T/7.3T），inode 仅 4%。「写满」是**文件系统损坏 + 底层 I/O 错误**造成的假象。

## 诊断流程（按执行顺序）

### 1. 先确认是否真满 —— 块用量 + inode 用量

```bash
# 注意：本机 df 被 alias 成 duf，用 \df 或 /usr/bin/df 调真二进制
\df -h  /mnt/d5179bee-9063-4651-a04b-3303a5304da7   # 块用量 55%
\df -i  /mnt/d5179bee-9063-4651-a04b-3303a5304da7   # inode 用量 4%
```

两者都不满 → 不是普通占满，往文件系统/硬件方向查。

### 2. 内核日志 —— 找 ext4 / I/O 错误

```bash
sudo dmesg -T | grep -iE "EXT4|sda|I/O error|bitmap|orphan|e2fsck"
```

本次命中：

```
EXT4-fs (sda): warning: mounting fs with errors, running e2fsck is recommended
EXT4-fs (sda): Errors on filesystem, clearing orphan list.
EXT4-fs error (device sda): ext4_validate_block_bitmap:423: bg 54515: bad block bitmap checksum
```

### 3. 翻「出事那次开机」的日志（关键技巧）

系统常常已重启，出错记录在**上一个 boot**。日志持久化时可直接翻：

```bash
journalctl --list-boots                       # 找出事的 boot 编号（如 -1）
sudo journalctl -b -1 -k | grep -iE "sda|EXT4|I/O error|bitmap"
```

命中 **69 次** SATA 链路错误（真正根因）：

```
sd 0:0:0:0: [sda] FAILED Result: hostbyte=DID_BAD_TARGET
I/O error, dev sda, sector 11081351944 op READ
EXT4-fs warning (device sda): ... comm lsof: error -5 reading directory block
```

`DID_BAD_TARGET` = 内核连这块 SATA 盘都寻址不到，盘在掉线/不响应（连 sector 0 都读不出）。

### 4. 文件系统自带的错误计数器

```bash
sudo tune2fs -l /dev/sda | grep -iE "state|error|Last checked|Mount count"
```

```
Filesystem state:   clean with errors
FS Error count:     8659
First error time:   Fri Dec 27 22:24:00 2024   # 损坏从 2024 年底就开始
Last checked:       Tue Dec 10 15:31:43 2019   # 6 年没 fsck 过
```

### 5. 确认是内置 SATA 还是 USB 外置

```bash
lsblk -o NAME,SIZE,TYPE,TRAN,MODEL,SERIAL /dev/sda
# sda 7.3T disk sata ST8000DM004-2CX188 → 内置 SATA，排查方向是数据线/电源/控制器
```

## 为什么会假报「写满」

`bad block bitmap checksum`（块位图校验和损坏）→ ext4 块分配器读到损坏位图找不到可用块 → 对写入返回 **ENOSPC（No space left on device）**，即使实际有 3.2T 空闲。这就是「假写满」。

## 处理建议（按顺序）

1. **先备份**能读出的重要数据（盘在掉线，别急着写它）。
2. **查链路硬件**：内置 SATA，`DID_BAD_TARGET` 多为数据线/电源线松动老化或控制器/背板问题 → 换数据线、换 SATA 口。
3. **离线修文件系统**（务必先 umount，绝不能挂载状态下跑）：
   ```bash
   sudo umount /mnt/d5179bee-9063-4651-a04b-3303a5304da7
   sudo e2fsck -fy /dev/sda
   ```
4. **看盘健康度**（smartmontools 默认可能没装）：
   ```bash
   sudo apt install smartmontools
   sudo smartctl -H -A -l error /dev/sda   # Reallocated / Pending / Uncorrectable
   ```
5. ST8000DM004 是叠瓦盘（SMR），8659 次错误且持续至今 → 盘很可能在退化，修完也要尽快迁移数据并考虑换盘。

## 速查命令清单

| 目的 | 命令 |
|------|-------------------|
| 真实块/inode 用量 | `\df -h <mnt>` / `\df -i <mnt>` |
| 当前 ext4/IO 错误 | `sudo dmesg -T \| grep -iE 'EXT4\|sda\|I/O error'` |
| 列出历史开机 | `journalctl --list-boots` |
| 翻上次开机日志 | `sudo journalctl -b -1 -k \| grep -iE 'sda\|EXT4'` |
| FS 错误计数器 | `sudo tune2fs -l /dev/sda` |
| 总线类型/型号 | `lsblk -o NAME,SIZE,TYPE,TRAN,MODEL /dev/sda` |
| 离线修复 | `sudo umount <mnt> && sudo e2fsck -fy /dev/sda` |
| 硬件健康 | `sudo smartctl -H -A -l error /dev/sda` |
