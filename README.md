# UniXNU on Xiaomi Pad 6S Pro (sheng) — Flash Research Log

小米 Pad 6S Pro 12.4（代号 `sheng`）刷入 UniXNU / iPadOS 27 的完整实验记录：
从立项调研、两条构建线的成败，到刷机之夜的引导门槛定位与启动配方。
记录时间：2026-11-15 至 2026-11-19（持续更新）。

> 种子（installation seed / chip id）属于设备私密材料，不在本仓库记录任何数值或哈希。

---

## 1. 目标与构建基线

- 设备：Xiaomi Pad 6S Pro 12.4（`sheng`，Snapdragon 8 Gen 2 / SM8550，256 GB），bootloader 已解锁，anti=1。
- 目标系统：UniXNU，iPadOS 27.0.1 (24A446) 配置。
- 设备树：Murderd1 的 `codex/self-hosted-mac` 分支 @ `52d7600`；核心：`Murderd1/UniXNU` @ `d830d23`；uefi-sheng @ `66421cb`。
- 2026-11-18 起第二条构建线：**官方线**——删旧 fork、按官方 `UniXNU/UniXNU-sheng` @ `c8f92b2` 重新 fork；全树对比确认朋友树只是官方的旧快照（7 个 submodule 有 5 个 pin 不同），此后以官方文档与官方代码为准（见第 2 节 11-18 / 11-19 条目）。
- 云端构建 #41（GitHub Actions，macos-26-intel，`init_data=true` 全新种子）构建成功，产物：
  - `boot.img`（185,712,640 B，header v0，内含 UEFI/DarwinBoot 桩 + payload）
  - first-install userdata 镜像（14,732,319,540 B，刷入后约 232 GiB 逻辑卷）
  - installation seed（私密保存，不入库）

## 2. 历程

### 2026-11-15 — 立项与调研
- 确认设备与目标：sheng、256 GB、bootloader 已解锁；只做 iPadOS 27，不碰其他版本。
- 调研结论先行：该移植无 Apple ID / iCloud / AirDrop（即使蓝牙/USB/Wi-Fi 在上游已实现），FairPlay 加密的 IPA 不能运行——预期管理到位后再动手。
- 确立刷写模型（见第 3 节），平板数据完成备份。
- 找到 Murderd1 的设备树：他本人在 2026-11-11 至 11-13 有多次自托管 CI 成功记录（#12–#25），是当时唯一有真机验证迹象的 sheng 方案。

### 2026-11-15 至 11-16 — 本地 Mac 构建线
- 在本地 Hackintosh 上沿上游配对路线构建，逐个修掉：`sched_clutch` 问题、12 符号基线链接、seal 失败。
- 最终卡在 prelink 布局：内核布局放不下，本地线止步于此（后被云端线证明是同一堵墙）。

### 2026-11-16 至 11-17 上午 — 上游配对云端线（#1–#39）
- GitHub Actions 上连续 39 次构建迭代，问题逐个收敛，最终 #39（2026-11-17 09:16）死在 prelink 的 **192 KB "Native code would collide with collection data"** 墙：336 个未解析引用全部指向内核本体，不是缺符号，是布局死结。
- 结论：上游配对路线在当前组合下走不通，止损。

### 2026-11-17 10:40 — 策略转向：照搬已验证的整套
- 定下纪律：**照搬就逐项照搬——代码、配置、运行环境（runner 架构）全部与原作者的成功配置一致，不做任何"等价替换"。** 此前本地线与云端线的多次失败，本质都是在某个环节换了未经验证的组合。
- #40 曾尝试用直链分发种子文件，因链接 403 失败——种子分发方式就此改为不经第三方直链（这也是后来所有工件改走 GitHub Releases 的原因）。

### 2026-11-17 11:37 — #41 成功
- 用作者的树 + 作者的工作流（只把自检出钉到他的仓库与提交）在 macos-26-intel 上构建，**#41 成功**：images、first-install userdata、构建日志、种子一整套产物齐全。
- 11:46 另一条 ARM 分支线 #42 也成功，但它生成了**独立的种子谱系**——两条线的种子与镜像严禁混用，#42 就此隔离封存，唯一主线是 #41。

### 2026-11-17 下午 — Windows 暂存与逐项校验
- 全部工件在 Windows 侧按 `package.manifest.json` 逐项核对大小与 SHA-256（boot.img / payload.img / kernel.macho / userdata）。
- 修掉刷机脚本的一个真 bug：fastboot 把 `product: sheng` 写到 stderr，被 PowerShell 的 Stop 策略误判为致命错误——改为分离 stdout/stderr 后预检才通过。
- 种子完成四处备份（Windows 两处、VM 一处、原始包一处），入仓文件加只读锁。
- 教训入账：早期构建的 userdata-root-27 镜像一度被凭印象断言"能与任何 boot 配对"，读过作者打包脚本后纠正——他的 first-install 镜像多了一步恢复 IPSW 的 RSA3072 签名 root 哈希表并校验，配对关系未经验证的镜像一律退役。**镜像配对必须读脚本确认，不能凭印象。**

### 2026-11-17 晚 — 刷机之夜
- userdata 三度刷入成功（每次 19/19 OKAY）；全球版基线对照、备用槽路径、BootLog 读取器攻坚与 dtbo 决定性实验，全部记录在下文第 4–6 节。

### 2026-11-18 — BootLog 终局、路线纠正与官方线立项
- **不向作者提问**：其仓库 Issue 对非协作者关闭，评论渠道也排除——最终决定消息搁置、不发出，三个问题改为内部自解（BootLog / reader 路线）。
- reader v3.1 / v3.2 两次实测：都是约 3 秒离开 fastboot、300 秒内无任何 USB 设备出现（无串口、无其他设备）。
- **KernelSU root 打通**（patch `init_boot_a`；sheng 是 init_boot 机型，root 补的是 init_boot 不是 boot），获得双尾读取通道：
  - `vendor_boot_b` 的日志环约 99.6% 非零高熵、零个有效标记 → reader 从未留下记录；
  - userdata 最后 32 MiB **全零** → 2026-11-17 21:12 那次启动的 BootLog 已被国行恢复后的首次开机格式化覆盖，**不可恢复，此问关闭**。教训入账：**BootLog 必须赶在任何 Android 格式化碰 userdata 之前读。**
- 现状落点：a 槽国行原厂 OS3.0.306.0 + KernelSU root 供日常使用与取证；b 槽保留 reader 组合，`dtbo_b` 处于擦除状态。
- **路线纠正（当晚自检）**：11-17 晚那次启动尝试之后，诊断逐渐变成主线、产品步骤原地停滞——承认走偏。当晚起回归官方文档（unixnu.com 的 building / guide/boot / requirements）为第一依据。
- 澄清一个误判：作者本人也是 Windows + GitHub 云端构建，并非 Mac 直连刷机；本机的 Hackintosh 与刷机无关，已决定不再修、计划移除 macOS（顺序：先刷机、后移除；Windows 与 OpenCore 共用一块 EFI，只删 APFS 容器，绝不能动 EFI）。
- 官方线立项：删旧 fork → 按官方主线 `c8f92b2` 重新 fork → 开 Actions → 产物保留期 1 天改 14 天 → 芯片种子从 #41 的 `payload.img` 里找回（ADT 的 `unique-chip-id` 属性，是软件身份种子）并由本人重新设为仓库 secret。

### 2026-11-19 凌晨至上午 — 官方构建连环坑（进行中）
官方 `package.yml` 在上游仓库至今 0 次运行记录——这条工作流从没在 GitHub 托管 runner 上完整跑过，我们是第一个踩全程的人。坑按出现顺序：

1. **tools 步骤 ipsw 装不上**：Homebrew 当前校验器拒绝 `blacktop/tap/ipsw` 的 cask 定义（`zap` stanza 报错），与架构无关。改法：brew 行去掉它，改装官方 release 的 universal 二进制（v3.1.749）。
2. **同一步内 PATH 不生效**：`$GITHUB_PATH` 只影响后续步骤，同一步的自检看不到刚装的二进制——ipsw、cargo 各栽一次。改法：同一步内再 `export PATH` 一次。
3. **mesa 补丁 #13 被 macOS 的 git 拒绝**：`git apply` 对 `turnip-frame-diagnostics.patch` 只报 `error: dev/null: No such file or directory`（Linux 上同一补丁干净通过）。专门的诊断工作流三方对照：原版失败 / 补 `index` 行仍失败（该理论否证）/ GNU `patch -p1` 成功。改法：在核心分叉里给 `git apply` 加 `patch -p1` 兜底，正向应用与身份校验的反向应用都覆盖。
4. **cargo 三连坑**：① Homebrew 只装裸 `rustup`、从不跑 `rustup-init`，工具链装了却没有 cargo shim；② 桥接脚本调用 `cargo +1.98.0 …`，只有 rustup 代理能处理 `+toolchain`；③ runner 上的 `/usr/local/bin/rustup` 是个 155 字节的 shim 脚本，重 exec 时把代理的 argv[0] 弄丢，软链、硬链代理全废。四轮 `cargo-diag` 诊断工作流定位后，改法：桥接改调 `rustup run 1.98.0 cargo`。**#15 首次越过桥接阶段**（此前所有运行都死在 mesa 或桥接之前）。
5. **WiFi 源码没人拉**：`drivers/Ath12kWifi/deps/` 被 gitignore，官方全流程没有任何一步调用拉取脚本，`prepare-supplicant.py` 一上来 `--check` 就失败（本地复现一致）。改法：package 步骤前新增一步显式拉取三份源码（Linux 内核 / FreeBSD / wpa_supplicant）。
6. **FreeBSD 锁定提交对不上**：锁文件钉死一个旧提交，但 `releng/15.1` 是移动分支，拉到的永远是分支尖端。改法：拉取前先按锁定提交号预置（按 SHA fetch 已验证可行）。Linux 那份用固定 tag，无此问题。
7. **sparse-checkout 没启用**：预置的裸仓库没开稀疏检出，后续 `git sparse-checkout add` 直接 `fatal`。改法：预置块里补一行 `git sparse-checkout init --cone`（本地已复现 + 验证）。

运行总账（官方线，Intel 为主线；arm64 多次因 GitHub 托管 runner 容量在排队约 15 分钟后被取消，已暂停派单）：

| 运行 | 结果 | 死点 |
|---|---|---|
| #1 | 排队取消 | arm64 runner 容量 |
| #2 | 失败 5m36s | tools：ipsw cask |
| #4 | 失败 11m41s | tools：同一步 PATH（ipsw） |
| #5 / #6 / #7 | 失败 | package：mesa 补丁 #13 |
| #8 | 失败 40m45s | mesa 已过；桥接缺 cargo |
| #10 | 失败 10m48s | tools：同一步 PATH（cargo） |
| #12 | 失败 25m37s | 桥接 `+toolchain` 需 rustup 代理 |
| #14 | 失败 28m03s | 桥接：rustup shim 吞 argv[0] |
| #15 | 失败 45m38s | 桥接已过；WiFi 源码未拉取 |
| #16 | 失败 10m06s | fetch：FreeBSD 提交不匹配 |
| #17 | 失败 12m19s | fetch：sparse-checkout 未启用 |
| #18 | 2026-11-19 08:58 触发 | 在跑（截至本记录） |

- 附带事件：#16 失败后 GitHub 一度停用了 fork 的全部工作流，#17 触发前在页面上手动重新启用，此后未再复发。
- 纪律延续：每次修复同时落到 main 与 pkg-arm64 两个分支，只派 Intel 一条线；产物（installation seed）每轮由 always() 步骤备份，跨轮不混用。

## 3. 刷写模型

- 唯一需要刷入的分区是 `userdata`（`fastboot flash userdata`，约 400 秒），iPadOS 的系统与数据都在这一个镜像里。
- **官方流程（现行主线）**：`fastboot flash userdata`（官方构建的 userdata 镜像）→ `fastboot reboot bootloader` → `fastboot boot boot.img` 临时启动；成败按官方检查点阶梯判定：UniXNU 启动菜单 → 绿带（交棒内核前）→ 蓝带（内核入口）→ 品红带（开 MMU 前）。
- `boot.img` 在作者的设计里通过 `fastboot boot` 临时加载；在**本机**这条路走不通（见结论 1），实际使用的是作者脚本里记录的第二条路：刷入备用槽 `boot_b` + `set_active b`，a 槽保留原厂 Android 作为退路。注：该异常是在朋友树旧快照镜像上测得的；官方镜像的布局（`uefi-chunks` ramdisk 压缩、`SHENG_BOOT_GUARD`）与试过的任何镜像都不同，官方线构建成功后将按官方流程原样复测 `fastboot boot`，不在复测前下结论。
- 回到 Android：Mi Flash `flash_all`（**永远不要 `flash_all_lock`**，跨区锁回有变砖风险）。

## 4. 已证实的结论（全部有对照实验支撑）

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

## 5. 实验矩阵（boot_b / vendor_boot_b / dtbo_b）

| # | boot | vendor_boot 的 DTB | dtbo | 结果 |
|---|---|---|---|
| 1 | 原厂 v4 | 原厂 8 DTB | 原厂 | 81 s，内核活（死于挂载 UniXNU userdata，预期内） |
| 2 | v4 打包（gzip 内核） | 主线 DTB | 原厂 | 6 s 死 |
| 3 | v4 打包（raw 内核） | 主线 DTB | 原厂 | 6 s 死 |
| 4 | v4 打包（raw 内核） | 原厂 8 DTB（hybrid） | 原厂 | **103 s 存活**，无串口 |
| 5 | v4 打包（raw 内核） | 主线 DTB（v3） | 原厂 | 6 s 死 |
| 6 | v4 打包（raw 内核） | 主线 DTB ×8（ID 池） | 原厂 | 6 s 死 |
| 7 | v4 打包（raw 内核） | 主线 DTB（v3） | 4 KiB 空头（残留旧层） | 6 s 死 |
| 8 | v4 打包（raw 内核） | 主线 DTB（v3） | **erase 全擦** | **103 s+ 存活**，无串口 |

规律：DTB 是原厂的 → 活；DTB 是主线的且 dtbo 还在 → 死；dtbo 擦掉 → 主线 DTB 也活。

## 6. BootLog 读取器（reader）

- 原理：UniXNU 的 UEFI/内核把 BootLog 写在 userdata 的**最后 32 MiB**。作者树自带 `package/linux-log`：一个主线 Linux 镜像，启动后把该窗口读进 RAM 并通过 USB CDC ACM 串口提供 shell，`scripts/read-sheng-log.py` 抓取后用 `rebootbl` 回到 fastboot。只读块设备，不挂载。
- 镜像来源：ianchb/sm8550-mainline 7.3.4 的 `boot_sheng_singleboot.img`（v0）+ 同版 deb 的内核/DTB。
- v4 改造要点：
  - vendor_boot 用原厂容器结构（头部/ramdisk 表/bootconfig 逐字节保留），只换 ramdisk 与 DTB 区；
  - 单boot 镜像里的"内核"段其实是 **gzip 压缩流**（文件名为 `Image.gz-dtb`），v4 路径不解压，必须换成解开的 raw arm64 Image（43,751,936 B）+ 尾随 DTB；
  - 主线 DTB 中 SMMU 节点名为 `iommu@15000000`（下游叫 `apps-smmu`），按节点名比对会误判"缺失"。
- Windows 侧读取脚本为原脚本的平台层移植（pyserial 按 VID 0x1D6B / PID 0x0104 找串口），协议与解码逐字节一致。

## 7. 当前状态与下一步

- ✅ 引导配方已打通：**v4 打包 + vendor_boot（原厂容器 + 目标 DTB）+ `fastboot erase dtbo_b` + 备用槽启动**（这是 reader 线的配方；UniXNU 本体改走官方流程，见下）。
- ✅ BootLog 之问已关闭：21:12 那次的日志被国行恢复的首次开机格式化覆盖，不可恢复；今后任何尝试若失败，先读 userdata 尾部、再谈恢复。
- ✅ 向作者提问一事已关闭：决定不发出，草稿留档未寄（仓库 Issue 关闭，本人最终拍板自己解决）。原三个问题备查：
  1. 他那台 sheng 上 `fastboot boot` 是否可用？固件/ABL 版本？
  2. 他验证过的启动是否都基于已有安装（原始 Data 种子）？`init_data=true` 冷首装是否走过、是否有坑？
  3. UEFI/DarwinBoot 是否使用 ABL 递来的 DTB？
- ⏳ reader v3.3（原厂容器 + 只换 reader ramdisk 的混合配方）设计已定稿，**降级为备用**：仅当官方线启动失败且需要一份新鲜 BootLog 时才构建（构建还差 Windows 侧提供国行原厂 `vendor_boot.img`）。
- ⏳ **官方构建 #18 在跑**（2026-11-19 上午）：成功后整套产物同源下载校验 → 按官方流程刷机（flash userdata → fastboot boot → 检查点阶梯判定），a 槽 Android 全程不动。
- ⏳ KernelSU root 去留待定（建议保留：它是目前唯一可用的取证读取通道；恢复只需重刷原厂 `init_boot_a` + `boot_a`）。
- ⏳ 刷机成功后移除 Hackintosh 的 macOS（只删 APFS 容器，保留 EFI 与 Windows 启动项）。

## 8. 工件索引（aipxu7xs/UniXNU-sheng 的 Releases）

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

## 9. 安全红线

- 只用 `flash_all`，**绝不 `flash_all_lock`**。
- 刷国行原厂包前核对 anti_version（本机 anti=1）。
- userdata 里的现存 BootLog 在读取前**不要**重刷 userdata 覆盖。
- 种子文件多处备份、私密保存；任何公开记录不写种子数值与哈希。
