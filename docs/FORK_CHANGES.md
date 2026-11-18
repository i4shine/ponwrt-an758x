# Fork 变更说明

上游：`wuziwei/ponwrt-ZN504-HG5585F`（master）
本仓库：`ufo1cn/ponwrt-ZN504-HG5585F`（fork，master）

本文件说明本次 fork 后新增/修改了哪些文件、每处改动解决什么问题。所有改动**不含**对上游既有工作流、DTS、分区表、镜像规则的修改。

---

## 一、一句话结论

| 目标 | 落点 |
| --- | --- |
| 支持我们前面编译的上层应用 | `configs/pon-userspace.config`（选包）+ `target/linux/airoha/an7581/config-6.18`（内核内置项） |
| 增加 HG5382A 固件支持 | 新增 `build-firmware-manual.yml` 的机型锁定与产物收集（上游机型定义与 `CONFIG_TARGET_DEVICE_...hg5382a=y` 本来就有，缺口是产物被 `find` 名字过滤静默丢弃） |
| 为固件编译软件包 | 新增 `build-packages-incremental.yml` |
| 增量编译 | 该工作流的滚动缓存（dl / ccache / build_dir / staging_dir）+ `only` 只编指定包 |
| kmod 与固件同内核树 | 该工作流先 `target/linux/compile` 再 `package/kernel/linux/compile`，vermagic 与固件一致 |
| 旧仓库自有包整理迁入 | `package/pon-userspace/**` + `patches/feeds/**` |

---

## 二、新增文件

### 1. `.github/workflows/build-firmware-manual.yml`（新增，由上游 `build-zn504.yml` 改写）

- **基线**：直接沿用上游 `build-zn504.yml` 的全部骨架 —— 依赖安装列表、`dl` 缓存 key、`customize.sh`、`make download`、编译命令、ccache 存取、产物 sha256、Artifact 上传、Release 发布。
- **触发**：删除 `push` 自动触发，只保留 `workflow_dispatch`（手动运行）；新增 `devices` 输入。
- **相对上游的四处改动**：
  1. 配置追加 `configs/pon-userspace.config`（上游只合 `an7581.config` + `release.config`）；
  2. `make defconfig` 之后做机型锁定（先删掉全部 `CONFIG_TARGET_DEVICE_airoha_an7581_DEVICE_*`，再写入本次目标），之后**不再** `defconfig`（避免机型行被重算掉）；其它机型关闭后不再重复建 rootfs；
  3. 产物收集的 `find` 加入 `*hg5382a*`（上游只匹配 `zn504*` / `hg5585f-ct*` / `hg5585f-cu*`，HG5382A 会被静默丢弃）；
  4. 新增「Verify requested firmware exists」步骤，缺目标 `.itb` 直接失败。
- **输入**：`devices`（both / hg5585f-ct / hg5382a）、`cache_mode`、`upload_release`。

### 2. `.github/workflows/build-packages-incremental.yml`（新增，仅手动）

- **触发**：只有 `workflow_dispatch`。
- **目的**：用**与固件完全相同的内核源码树与 .config** 编 kmod 与上层包，产物 apk 可直接装进固件。
- **同 ABI 的做法**：
  `tools/compile` → `toolchain/compile` → `target/linux/compile`（+`install`）→ `package/kernel/linux/compile` → 逐个 `package/<路径>/compile`。
  kmod 的 vermagic 取自同一棵 `build_dir/target-*/linux-an7581/`，因此和固件逐字节一致。
- **增量**：
  1. `dl`、`.ccache`、`build_dir/host`、`staging_dir/host`、`build_dir/target-*`、`staging_dir/target-*` 全部走滚动缓存（key 带 `run_id`，`restore-keys` 前缀命中上一轮，因此每轮都能存、下一轮能续）；
  2. 命中后 `make` 自身按时间戳跳过未变目标；
  3. 输入 `only` 可只编指定 apk（例 `tempinfo,camlive`），单包改动只重编该包；
  4. `cache_mode=完全重建` 可强制丢弃全部缓存。
- **不静默丢弃**：按 `configs/pon-userspace-packages.txt` 逐包编译并核对真实 apk；目录缺失、无 apk 产出、编译失败都会让 job 失败，并把 OK/FAIL/SKIP 写进 `build_log/package-summary.txt` 与 Step Summary。
- **输入**：`device`、`cache_mode`、`reuse_build_tree`、`only`、`with_kernel`（首次或改内核配置时必须 true）。

### 3. `configs/pon-userspace.config`（新增）

固件/包的选包清单，按用途分段：

- 自有应用：`tempinfo`、`camlive`、`go2rtc`、`gensio`、`ser2net` 及 4 个 `luci-app-*`（camlive / go2rtc / ser2net / btsp）；
- 蓝牙串口前端依赖：`bluez-daemon`、`bluez-libs`、`dbus`（**不引 `bluez-alsa`**：它只在第三方仓库 `raphikWasHere/bluealsa4openwrt` 里有源码，按你的决定不引入，`luci-app-blueaudio` 因此也不参与编译）；
- USB 主机/存储/串口：`kmod-usb-core`、`kmod-usb3`、`kmod-usb-xhci-hcd/-mtk`、`kmod-usb-storage(-uas/-extras)`、`kmod-usb-serial(-ch341/-cp210x/-ftdi/-pl2303)`、`kmod-usb-acm`、`kmod-usb-hid`、`kmod-usb-net(-cdc-ether/-cdc-ncm/-rtl8152/-asix-ax88179)`；
- 摄像头/音频：`kmod-video-core`、`kmod-video-videobuf2`、`kmod-video-uvc`、`audio-support`、`kmod-input-core`、`kmod-sound-core`、`kmod-usb-audio`、`alsa-utils`；
- 蓝牙：`kmod-bluetooth`、`kmod-btusb`、`kmod-btbcm`、`kmod-btmtk`、`kmod-hci-uart`；
- USB 无线网卡：`kmod-rtw88*`、`kmod-rtl8xxxu`、`kmod-rtl8192cu`、`linux-firmware-rtw88`；
- 文件系统/共享：`kmod-fs-ext4/-vfat/-exfat/-ntfs3/-cifs`、`block-mount`、`nfs-kernel-server` 及 utils；
- 诊断：`kmod-netlink-diag`、`kmod-inet-diag`。

> 这些包名全部取自我们此前在 `ufo1cn/PonWrt-HG5382A` 上真实编过的清单，不是猜测名。

### 4. `configs/pon-userspace-packages.txt`（新增）

增量编译清单，格式 `<构建路径>|<apk 名>|<required|optional>`，顺序即编译顺序；当前清单无 `optional` 项（`bluez-alsa` / `luci-app-blueaudio` 已移除）。

### 5. `package/pon-userspace/**`（新增，31 个文件）

从旧仓库 `pon_userspace/` 迁移并整理：

```
package/pon-userspace/
├── tempinfo/                 Makefile + files/tempinfo（LuCI 温度行取数脚本）
├── camlive/                  Makefile + src/camlive.c（UVC 推流后端）
├── go2rtc/                   Makefile + files/{go2rtc.init,go2rtc.config}
├── gensio/                   Makefile（ser2net 的依赖，原 ser2net4/gensio）
├── ser2net/                  Makefile + files/
├── luci-app-camlive/         Makefile + root/（配置、init、controller、cbi、i18n）
├── luci-app-go2rtc/          Makefile + root/（controller、cbi、view）
├── luci-app-ser2net/         Makefile + luasrc/ + root/etc/config
└── luci-app-btsp/            Makefile + root/
```

> `luci-app-blueaudio` 未迁入：其硬依赖 `bluez-alsa` 只有第三方源码，按决定不引入。

与原仓库的差异：旧仓库把这三个 ser2net 相关包放在 `pon_userspace/ser2net4/` 子目录、由工作流拆开部署；这里按 OpenWrt 多包目录规范拆平到 `package/pon-userspace/`，`make` 能直接按 `package/pon-userspace/<pkg>/compile` 增量编译。

### 6. `patches/feeds/**`（新增，3 个文件）

| 文件 | 来源 | 用途 |
| --- | --- | --- |
| `patches/feeds/pon_userspace/0001-gpon-luci.patch` | 旧仓库同名 | GPON 状态页 LuCI 集成，`customize.sh` 打进 `feeds/pon_userspace` |
| `patches/feeds/pon_drivers/0001-gpon-full.patch` | 旧仓库同名 | GPON 驱动补丁 |
| `patches/feeds/pon_drivers/rtl8723bu/Makefile` | 旧仓库同名 | 给 pon_drivers feed 增加 rtl8723bu 无线包定义 |

旧仓库里的 `0002/0003/0004-add-luci-app-*.patch.bak` 是当年把 LuCI 前端塞进 `pon_userspace` feed 的备份方案；现在前端已作为独立包放在 `package/pon-userspace/`，这些 `.bak` 不再需要，**未迁入**。

---

## 三、修改文件

| 文件 | 改动 | 目的 |
| --- | --- | --- |
| `target/linux/airoha/an7581/config-6.18` | 末尾追加 18 行内核符号 | 见下 |
| `scripts/customize.sh` | 新增第 8 节 | 应用 `patches/feeds/**`（feed 补丁 + rtl8723bu 包定义）；**不**再拉第三方 bluez-alsa |
| `.github/workflows/build-zn504.yml` | 删除 `push` 触发块 | 上游原工作流改为只手动运行，防止提交即自动重编 |

`config-6.18` 追加内容与理由：

```
CONFIG_NFS_FS / NFS_V4 / NFSD / NFSD_V4 / LOCKD / SUNRPC / GRACE_PERIOD = y
    NFS server 全链内核内置（沿用此前“内置吧”的决定）。
    因此 configs/pon-userspace.config 只选用户态 nfs-kernel-server，
    不去选 kmod-fs-nfs* —— 内置后再选 kmod 只会得到空壳 apk。
CONFIG_CRYPTO_MD5 / SHA256 / AES = y      NFSD_V4 的加密依赖，显式内置避免被裁掉
CONFIG_DEBUG_FS / DEVMEM = y             tailscale/zerotier/honk/diskman 依赖
CONFIG_NETLINK_DIAG / INET_DIAG = y      ss/netlink 诊断（zerotier、tproxy 排查）
CONFIG_NL80211_TESTMODE = y              cfg80211 在 CI 无 TTY 下 syncconfig 会卡死，必须显式定值
```

> UVC / 蓝牙 / USB 串口 / rtw88 等驱动**没有**写进 `config-6.18`：它们由对应 kmod 包的 `KCONFIG` 在配置阶段置为 `=m`，这样既能进固件，也能在增量工作流里单独重编，改名/升级不用动内核碎片。

---

## 四、刻意未改的文件

| 文件 | 为什么不动 |
| --- | --- |
| `.github/workflows/release.yml` | 上游发布流程与本次目标无关。 |
| `configs/an7581.config` | `CONFIG_TARGET_DEVICE_airoha_an7581_DEVICE_fiberhome_hg5382a=y` 与 `fiberhome_an4475-g2=y` 本来就已选中，机型定义在 `target/linux/airoha/image/an7581.mk` 里也完整；缺的只是**产物收集**，已在新增工作流里补上。 |
| `target/linux/airoha/image/an7581.mk` | HG5382A 的 `Device/fiberhome_hg5382a` 与 `TARGET_DEVICES +=` 均存在，无需新增。 |
| DTS / 分区表 / `scripts/integrate-temperature.py` | 分区与温度 RPC 已与我们的设备一致，本次不触碰。 |

---

## 五、使用方式

1. 固件：Actions → **Build PonWrt (HG5585F-CT & HG5382A) manual** → Run workflow，`devices` 选 `both`（或单个），首次建议 `cache_mode=使用缓存`、`upload_release=false`。
2. 软件包：Actions → **Build Packages (same kernel tree, incremental) manual** → 选 `device`；首次或刚改过 `config-6.18` 时 `with_kernel=true`，之后可 `with_kernel=false` + `only=<apk 名>` 做单包增量。
3. 三个工作流（含改造后的上游 `Build PonWrt (ZN504XG-D & HG5585F)`）都只有 `workflow_dispatch`，不点就绝不会跑。

## 六、已知限制

- `build_dir/target-*` / `staging_dir/target-*` 缓存体积可达数 GB，GitHub 单条缓存上限 10GB；保存步骤已设 `continue-on-error`，缓存超限不会让编译失败，只是下一轮退化为重新编译。
- 三个工作流（两个新增 + 改造后的 `build-zn504.yml`）都只有 `workflow_dispatch`，不会因提交自动跑。
- NFS 内置后，`kmod-fs-nfs*` 若被依赖链选中会生成空壳 apk（预期行为，不影响运行）。

## 七、变更历史

1. 首版：fork 后一次性引入两个新工作流、配置、包源码、补丁与内核配置项。
2. 二次修订（按用户要求）：
   - 不引入第三方 `bluez-alsa`，移除 `luci-app-blueaudio`（包源码、选包、编译清单、customize.sh 注入全部删除）；
   - 上游 `build-zn504.yml` 删除 `push` 触发，改为只手动；
   - `build-firmware-manual.yml` 明确以 `build-zn504.yml` 为基线，保留其步骤骨架，仅改触发方式、配置合并、机型锁定与产物收集/校验。
