# kernel_xiaomi_raphael-5.4 (ban-sky snapshot)

Redmi K20 Pro / Mi 9T Pro（代号 **raphael**，SM8150）的 5.4 第三方内核，来源：

- 上游：<https://github.com/HeliumStudio-Dev/kernel_xiaomi_raphael-5.4>
- 分支：`zundamon-miui-5.4`，快照提交 `c50effadd8f6cfc012f1af15daf05967a90dbe91`（v1.0-alpha5）
- 本仓库为**单分支快照**（不是 fork），保留了 `upstream` 远端，方便后续 `git fetch upstream` 对比/合并。

## 这个分支是干什么的

上游提交信息显示它面向 **MIUI / 老 HyperOS 的 vendor**：

- `power: supply: Restore legacy ABI used by QC3 charging service` —— 恢复 MIUI QC3 充电服务依赖的 legacy power-supply ABI；
- `drivers: power: supply: Import Xiaomi changes` —— 把小米 SMB5 / FG Gen4 私有控制 forward-port 到 5.4；
- 上游自己注明：PD policy engine 的问题仍被推迟、27W 峰值未测。

也就是说：**5.4 不等于"澎湃 OS 4 内核"**。目前跑澎湃 OS 4 移植包的 K20 Pro 实测是 4.14（`4.14.357-Zundamon-Evo`），5.4 主要配合 AOSP / OSS vendor 的 ROM（如 LineageOS 23.x、DerpFest 17，社区称 RaphGhost 5.4.302）。混刷前先确认 ROM 的 vendor 代际。

## 本快照做的修复

1. **NFC 驱动没编进内核**（P0）
   - `arch/arm64/configs/raphael_defconfig` 增加 `CONFIG_NFC_QTI_I2C=y`。
   - 设备树 `nq@28`（`compatible = "qcom,sn-nci"`，i2c@a84000，地址 0x28）本来就在，但 `drivers/nfc/Kconfig` 里该符号是 tristate、无 default、无 select，所以之前根本没有驱动绑定，`/dev/nq-nci` 不存在。
2. **NFC 中断触发方式**
   - `drivers/nfc/qti/nfc_i2c_drv.c`：`IRQF_TRIGGER_RISING` → `IRQF_TRIGGER_HIGH`。SN100 的 INT 是电平有效（DT 里 `interrupts = <47 0>`，pinctrl 上拉），handler 屏蔽中断期间电平保持高，上升沿会丢包。
3. **背光映射**（P0）
   - `techpack/display/msm/sde/sde_connector.c`：eDP 分支不再解引用未初始化的 `dsi_display`；空 panel 走线性映射；`brightness_max_level - 1` 与负 `bl_range` 都做了保护（原来是除零 / GCC `?:` 扩展）。
4. **视频队列结构体尺寸**
   - `techpack/video_kona/msm/vidc/msm_vidc.c`：显式 `q->buf_struct_size = sizeof(struct vb2_v4l2_buffer)`。
   - 说明：vb2 的 `vb2_queue_init()` 本身会把 0 补成 `sizeof(struct vb2_v4l2_buffer)`，所以这条不是修 bug，只是把依赖显式化。
5. **QC3 18W 不上报**（P1）
   - `drivers/power/supply/qcom/smb5-lib.c`：class A（18W）分支补 `power_supply_changed(chg->usb_psy)`；class B 本来就有。之前 `HVDCP3_TYPE` 会一直停在 NONE，用户态看不到 18W。
6. **`input_current_now` 恒 0 + 刷屏**（P1）
   - `drivers/power/supply/qcom/smb5-iio.c`：补 `case PSY_IIO_INPUT_CURRENT_NOW`，走 `smblib_get_prop_usb_current_now()`（`smb5-lib.c:5110`，读 `chg->iio.usbin_i_chan` 并做 buck/boost 缩放）。原先这个通道没有 case，恒返回 `-EINVAL`，`usb_cur_in` 恒 0 且每轮 reg_work 打印 `get prop 6 is not supported`。
7. **ESD IRQ 计数不平衡 + 裸改 istate**（P1）
   - `techpack/display/msm/dsi/dsi_panel.c`：`irq_to_desc()` 判空再动 `istate`；disable 前查 `irqd_irq_disabled()`，避免重复 `disable_irq_nosync()`。
   - `techpack/display/msm/dsi/dsi_drm.c`：`dsi_bridge_disable()` 对 seamless/VRR/dyn-clk 模式提前返回，与 enable 对称。之前 enable 早返回、disable 却照样关 ESD IRQ，seamless 模式下 ESD 保护等于永久失效。
8. **KernelSU branch-link 与 CFI 冲突风险**（P1）
   - `arch/arm64/configs/raphael_defconfig`：关掉 `CONFIG_KSU_HACK_ARM64_BRANCH_LINK`（该 Kconfig 自述 "NOT recommended for pre-4.17 kernels"，且 `CONFIG_CFI_CLANG=y` 并存时 inline 改写 text 可能触发 CFI trap）。KSU 仍走 LSM hooks。

## 版本与产物

- 版本串（`CONFIG_LOCALVERSION`）：`-Zundamon-1.0-rc1`，`uname -r` 为 **`5.4.302-Zundamon-1.0-rc1`**。
- 刷机包：`Zundamon-5.4.302-1.0-rc1.zip`（AnyKernel3 结构，只替换 boot 里的 kernel）。**不含 `dtbo.img`**：boot 的 dtb 必须和 ROM 的 dtbo 同代，覆盖 ROM 的 dtbo 容易开不了机，dtbo 镜像单独放在 `raphael-5.4-image` artifact 里备用。命名规则 `Zundamon-<内核版本>-<版本号>[-rcN]`，workflow 顶部的 `KNAME/KVERNUM/KSTAGE` 三个 env 控制。
- Actions 里下载 artifact `Zundamon-5.4.302-1.0-rc1`，同一目录下有 `.sha256`。

## 构建

`.github/workflows/build-kernel.yml`：ubuntu-22.04 + clang/lld + `gcc-aarch64-linux-gnu`，`make O=out ARCH=arm64 raphael_defconfig` 后编 `Image dtbs`，产物（`Image`、`*.dtb`、`*.dtbo`、config）上传为 artifact。

构建结果：**run 44986012908（commit `850a1692f`）success**，artifact `raphael-5.4-image`（22.5 MB）。在 Actions 页面下载即可。

## 构建验证的边界

CI 只证明**能编过**（`Image`/dtb/dtbo 生成、内核镜像打包成功），不等於刷入后一定稳定：PD/27W、相机、NFC 实机行为还需要真机验证（本仓库未做本机编译，也没有真机日志）。

## 已知未修 / 待验证

- **PD 策略引擎**：上游明确"deferred"，27W 峰值未测；`smb5-lib.c` 里 `input_suspend` 与 `current_max` 共用 `usb_icl_votable` 的 `USER_VOTER`，后写覆盖前写（待验证）。
- **main psy 从不发 uevent**：`smb5_init_usb_main_psy()` 注册的 `"main"` psy 指针被丢弃，`smb1390-charger-psy.c` 却在按名字监听它，QC3/健康度刷新会滞后。
- **health 数值 ABI 写读不对称**：写 10 被重映射成 WARM，读回来是字符串 `"Warm"`，用户态 `atoi` 得 0。
- **`POWER_SUPPLY_PROP_INPUT_CURRENT_NOW` → `PSY_IIO_INPUT_CURRENT_NOW`(0x06) 没有 case**：`smb5-iio.c` 里没有这个通道，恒返回 `-EINVAL`，`smb5-lib.c` 不查返回值，`usb_cur_in` 恒 0 并周期性打印 `get prop 6 is not supported`。
- **Codec2 legacy flag 双向转换对所有客户端生效**（`msm_v4l2_vidc.c`），legacy EOS/READONLY/PERF_MODE 与新标志位冲突，新 HAL 可能被误解释；需要按客户端区分。
- **`dsi_panel_esd_irq_ctrl`** 的 `istate` 裸改、`irq_to_desc()` 未判空、enable/disable 计数不平衡。
- **KernelSU branch-link + `CONFIG_CFI_CLANG`** 同时开启（`raphael_defconfig:104/711`），有 CFI trap 风险，建议只用 LSM hooks。
- **「卡二」（卡第二屏/开机动画循环）**：内核里没有 SIM/UIM 代码，`pil_modem` 与 glink 节点完整，`CONFIG_MHI_QCOM=n` 只影响外部 MHI；该症状更可能是 NON-HLOS/基带固件、qcril/HAL 或移植包本身，需要 radio log 才能定位。
- NFC 在澎湃 OS 4 上的崩溃另有一条**用户态**根因：OS4 的 `libsn100nfc-nci.so`（MW15）新增 MIFARE Classic 重传，连续 AUTH 把 PN553 的 I2C 拖死（`nq-nci: nfc_write: failed to write after 3 retries` / `i2c_geni ... status:0x4d3`），内核侧驱动不是主因。

## 刷机提醒

- 内核必须与 ROM 的 vendor 匹配；5.4 不要盖在 4.14 的移植包上（社区明确 "clean flash required — new kernel generation"）。
- 刷前备份 `boot` / `dtbo`；有 ramoops 的话，出事能拿到 `console-ramoops`。
- 确认版本：`magiskboot unpack boot.img` 或 `unpack_bootimg --boot_img boot.img` 看 `kernel` 的版本串。
- 包是 AnyKernel3：TWRP/OrangeFox 里直接刷 zip，脚本会 dump 现有 boot、替换 kernel；zip 里**不含** `dtbo.img`，dtbo 分区保持 ROM 原样；需要单独刷 dtbo 的从 `raphael-5.4-image` artifact 取。
- CI 打包后会跑 `unzip -l` 校验 `Image`、`anykernel.sh`、`tools/ak3-core.sh` 都在，缺一个就直接判失败，所以绿色运行的包至少结构是完整的。
