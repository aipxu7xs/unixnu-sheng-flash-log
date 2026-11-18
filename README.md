# UniXNU on Xiaomi Pad 6S Pro (sheng) — Flash Research Log

小米 Pad 6S Pro 12.4（代号 `sheng`）刷入 UniXNU / iPadOS 27 的完整实验记录：
引导门槛的定位过程、已证实的结论、失败的死路、以及当前可用的启动配方。
记录时间：2026-11-15 至 2026-11-18（持续更新）。

> 种子（installation seed / chip id）属于设备私密材料，不在本仓库记录任何数值或哈希。

---

## 1. 目标与构建基线

- 设备：Xiaomi Pad 6S Pro 12.4（`sheng`，Snapdragon 8 Gen 2 / SM8550，256 GB），bootloader 已解锁，anti=1。
- 目标系统：UniXNU，iPadOS 27.0.1 (24A446) 配置。
- 设备树：Murderd1 的 `codex/self-hosted-mac` 分支 @ `52d7600`；核心：`Murderd1/UniXNU` @ `d830d23`；uefi-sheng @ `66421cb`。
- 云端构建 #41（GitHub Actions，macos-26-intel，`init_data=true` 全新种子）构建成功，产物：
  - `boot.img`（185,712,640 B，header v0，内含 UEFI/DarwinBoot 桩 + payload）
  - first-install userdata 镜像（14,732,319,540 B，刷入后约 232 GiB 逻辑卷）
  - installation seed（私密保存，不入库）

## 2. 刷写模型

- 唯一需要刷入的分区是 `userdata`（`fastboot flash userdata`，约 400 秒），iPadOS 的系统与数据都在这一个镜像里。
- `boot.img` 在作者的设计里通过 `fastboot boot` 临时加载；在**本机**这条路走不通（见结论 1），实际使用的是作者脚本里记录的第二条路：刷入备用槽 `boot_b` + `set_active b`，a 槽保留原厂 Android 作为退路。
- 回到 Android：Mi Flash `flash_all`（**永远不要 `flash_all_lock`**，跨区锁回有变砖风险）。

## 3. 已证实的结论（全部有对照实验支撑）

### 结论 1：本机 ABL 只接受 header v4 的 boot 镜像，`fastboot boot` 路径关闭
- 任何 v0 镜像走 `fastboot boot`：第一次返回"空 OKAY"（设备实际没有复位），第二次报 `Failed to load/authenticate boot image: Load Error`。
- 对照实验：原厂 v4 `boot.img` 刷入 `boot_b`（vendor_boot 为原厂）→ 设备正常离开 fastboot、出现 Mi 启动画面、约 81 秒后才因 userdata 是 UniXNU 而退出。**槽位通路与 ABL 本身健康，门只在镜像格式。**

### 结论 2：原厂 dtbo 覆盖层会杀死"外来" DTB —— 擦除 dtbo 后同一镜像立即存活
- 现象：vendor_boot 里放主线 DTB（144,735 B）→ 6 秒回 fastboot；放原厂 DTB → 内核存活 100 秒以上。换 DTB 的 ID（结论 5）、换容器、关 AVB 都不改变。
- 决定性实验：同一份镜像（v4 内核 + 主线 DTB），只把 `dtbo_b` 从"原厂"改为"**fastboot erase 完全擦除**"，结果从 6 秒死亡变为 103 秒以上稳定存活。
- 机理：ABL 在交棒前会把 dtbo 分区的覆盖层套到 vendor_boot 的基 DTB 上。原厂 DTB 带 `/__symbols__`，主线 DTB 没有，覆盖层套用在外来 DTB 上失败并中止启动。
- 注意：只刷一个 4 KiB 的"空 dtbo 头"**不算**擦除——分区是 24 MiB，`fastboot flash` 只写文件长度，残留的旧覆盖层仍会被套用（此坑实测踩过，结论一度被误判为"已排除"）。

### 结论 3：与"底包"（国行 / 全球版原厂固件）无关
- 全球版 OS3.0.305.0 基线与国行 OS3.0.306.0 基线下，所有关键行为一致（v0 拒绝、Load Error、6 秒死、擦 dtbo 后存活）。早期"ABL 地区差异"的假设已被对照实验否证。

### 结论 4：与 AVB 校验无关
- 给 `vbmeta` / `vbmeta_a` / `vbmeta_b` 写入 `--disable-verity --disable-verification` 后行为无任何变化，排除。

### 结论 5：DTB 的硬件 ID 不是门（ID 池实验）
- 原厂 vendor_boot 的 DTB 区是 8 个**通用 Kalama SoC** DTB（按 msm-id 分变体：519/536/600/601/603/604，board-id 全为 `<0,0>` 通配），不是 sheng 专属树。
- 主线 DTB 自述 msm-id `<536>`（KalamaP）、board-id `<0x10008, 0>`。曾假设 ABL 按 CDT 选不中它。
- 实验：把主线 DTB 复制 8 份、每份 ID 改成一个原厂变体（board-id 清零通配），结果仍是 6 秒死——**选择理论被否证**，真凶是结论 2 的覆盖层（选中之后才被套死）。

### 结论 6：reader 内核的 USB 串口沉默，疑点在 Type-C 驱动模块
- 主线内核在正确 DTB 上存活时，USB gadget 始终不枚举。内核 config 核查（ianchb 7.3.4 的 deb 内 config 原文）：gadget 全套（`USB_CONFIGFS_SERIAL`、`DWC3`、`DWC3_QCOM`、QMP PHY）均为 `=y`；
- 但 `CONFIG_TYPEC_QCOM_PMIC=m`，其模块真名 `qcom_pmic_tcpm.ko` 不在原版 reader ramdisk 里（只有一个 nvmem 模块）→ Type-C 角色无人上报 → 双角色 dwc3 不激活 UDC → 总线全程无声。
- 修复版 ramdisk（v2）已补入 `tcpm.ko` + `qcom_pmic_tcpm.ko` + `ucsi_ccg.ko`（vermagic 全部匹配），待实测。

## 4. 实验矩阵（boot_b / vendor_boot_b / dtbo_b）

| # | boot | vendor_boot 的 DTB | dtbo | 结果 |
|---|---|---|---|---|
| 1 | 原厂 v4 | 原厂 8 DTB | 原厂 | 81 s，内核活（死于挂载 UniXNU userdata，预期内） |
| 2 | v4 打包（gzip 内核） | 主线 DTB | 原厂 | 6 s 死 |
| 3 | v4 打包（raw 内核） | 主线 DTB | 原厂 | 6 s 死 |
| 4 | v4 打包（raw 内核） | 原厂 8 DTB（hybrid） | 原厂 | **103 s 存活**，无串口 |
| 5 | v4 打包（raw 内核） | 主线 DTB（v3） | 原厂 | 6 s 死 |
| 6 | v4 打包（raw 内核） | 主线 DTB ×8（ID 池） | 原厂 | 6 s 死 |
| 7 | v4 打包（raw 内核） | 主线 DTB（v3） | 4 KiB 空头（残留旧层） | 6 s 死 |
| 8 | v4 打包（raw 内核） | 主线 DTB（v3） | **erase 全擦** | **103 s+ 存活**，无串口 |

规律：DTB 是原厂的 → 活；DTB 是主线的且 dtbo 还在 → 死；dtbo 擦掉 → 主线 DTB 也活。

## 5. BootLog 读取器（reader）

- 原理：UniXNU 的 UEFI/内核把 BootLog 写在 userdata 的**最后 32 MiB**。作者树自带 `package/linux-log`：一个主线 Linux 镜像，启动后把该窗口读进 RAM 并通过 USB CDC ACM 串口提供 shell，`scripts/read-sheng-log.py` 抓取后用 `rebootbl` 回到 fastboot。只读块设备，不挂载。
- 镜像来源：ianchb/sm8550-mainline 7.3.4 的 `boot_sheng_singleboot.img`（v0）+ 同版 deb 的内核/DTB。
- v4 改造要点：
  - vendor_boot 用原厂容器结构（头部/ramdisk 表/bootconfig 逐字节保留），只换 ramdisk 与 DTB 区；
  - 单boot 镜像里的"内核"段其实是 **gzip 压缩流**（文件名为 `Image.gz-dtb`），v4 路径不解压，必须换成解开的 raw arm64 Image（43,751,936 B）+ 尾随 DTB；
  - 主线 DTB 中 SMMU 节点名为 `iommu@15000000`（下游叫 `apps-smmu`），按节点名比对会误判"缺失"。
- Windows 侧读取脚本为原脚本的平台层移植（pyserial 按 VID 0x1D6B / PID 0x0104 找串口），协议与解码逐字节一致。

## 6. 当前状态与下一步

- ✅ 引导配方已打通：**v4 打包 + vendor_boot（原厂容器 + 目标 DTB）+ `fastboot erase dtbo_b` + 备用槽启动**。
- ⏳ reader v2（补 Type-C 模块）待刷机验证：出现串口 → 读取 BootLog → 定位 UniXNU 本体的死亡点（现有证据只定位到"UEFI/DarwinBoot 载荷交接段、XNU 显示代码之前"：构建日志显示内核带蓝/品红进度条补丁，而实测一次都没出现过）。
- ⏳ UniXNU 本体的 v4 重打包（内核桩 + payload 原样，换 v4 头）待 BootLog 结论后执行。
- ⏳ 已向作者提出三个问题（其仓库关闭了 Issue，渠道待定）：
  1. 他那台 sheng 上 `fastboot boot` 是否可用？固件/ABL 版本？
  2. 他验证过的启动是否都基于已有安装（原始 Data 种子）？`init_data=true` 冷首装是否走过、是否有坑？
  3. UEFI/DarwinBoot 是否使用 ABL 递来的 DTB？

## 7. 工件索引（aipxu7xs/UniXNU-sheng 的 Releases）

| Release | 内容 |
|---|---|
| `logreader-20261117` | reader v0 镜像 + Windows 读取脚本 |
| `logreader-v4-20261117` | v4 boot（gzip 内核版）+ 手工 vendor_boot |
| `logreader-v4raw-20261117` | v4 boot（raw 内核版，43,900,928 B） |
| `dtbo-empty-20261117` | 4 KiB 空 dtbo 头（注：不足以清除分区，见结论 2） |
| `logreader-v2-20261118` | reader ramdisk v2（补 Type-C 模块，1,819,804 B） |

关键镜像哈希：
- UniXNU #41 boot.img：185,712,640 B（发布于构建工件）
- boot_v4_raw.img：sha256 `692c5692835ffc135697c2873910d0af4baa74538d5544a6752100433acf1381`
- vendor_boot_v3.img：sha256 `7664803aacc9cdda9500fee3ba2e4ae0601dfac483dfb9a87078462faba62933`
- reader_initramfs_v2.cpio.gz：sha256 `fb068ddb77d8cf46b3bd93682bea0581ff98db002c1a4d8c4b746b57396095c9`

## 8. 安全红线

- 只用 `flash_all`，**绝不 `flash_all_lock`**。
- 刷国行原厂包前核对 anti_version（本机 anti=1）。
- userdata 里的现存 BootLog 在读取前**不要**重刷 userdata 覆盖。
- 种子文件多处备份、私密保存；任何公开记录不写种子数值与哈希。
