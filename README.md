# monitor-agent

跨平台预编译二进制集合，派生自官方 Linux agent [`monitor-probe/agent`](https://github.com/monitor-probe/agent)（Cargo 包名 `monitor-agent` v1.1.0）。

协议层（WebSocket / JSON-RPC / 重连 / 上报节拍）完全复用原版，各平台的差异只在**采集层**：macOS 用新写的 BSD 版 `collect_darwin.rs`，MT7621 走 musl 静态链接的 Linux 版。上报字段集合与 Linux 版逐字段一致（见文末），hub 侧无需任何改动。

---

## 1. 下载（Release `v1.1.0`）

| 文件 | 平台 | 可执行格式 | 大小 | sha256 |
|---|---|---|---|---|
| `monitor-agent-aarch64` | Linux aarch64 | ELF64 | — | 见 Release 页面 |
| `monitor-agent-darwin-arm64-v3` | macOS Apple Silicon | Mach-O arm64 | 1.13 MB | `7bd7aba0c7f734ee0135bd73aa4181e4a6dad1ca765b0265882f57cd6c65bdc5` |
| `monitor-agent-darwin-x86_64-v3` | macOS Intel | Mach-O x86_64 | 1.36 MB | `7e21762a0afb8b3f18d51b06350816e86eeba2fdf9dca9c4cf8747463ec8f797` |
| `monitor-agent-mt7621` | MT7621 路由器（OpenWrt / mipsel） | ELF32 mipsel 静态 musl | 1.88 MB | `8a5d37c3c5bbf6c3ed2822753881e8e95fa83ff90b68dff067c3a80107603c31` |

> 两个 macOS 二进制为 **v3**：修复了网卡流量计数器 `NET_RT_IFLIST2` 的 selector 探测（真机实测命中 selector 6）。`-v3` 即该修复版，是当前推荐版本。

---

## 2. 通用命令行

所有平台命令行一致：

```bash
monitor-agent --server <hub-url> --token <node-token> [options]

  --server <url>       Hub 根地址，如 https://status.xiguan.net
  --token <token>      hub 面板生成的节点 token（64 hex）
  --interval <secs>    上报周期，默认 1
  --iface <list>       只统计这些网卡的流量，逗号分隔；`-名字` 表示从默认集合扣除
  --insecure           允许 ws:// 明文（token 明文传输，仅限内网 ip:port）
```

---

## 3. 各平台用法

### 3.1 Linux（aarch64 / x86_64）

直接用 hub 自带脚本（它认 systemd / OpenRC，并拉对应架构的 Linux ELF）：

```bash
curl -fsSL https://<hub>/install.sh | sh -s -- \
     --server https://<hub> --token <token> --interval 1
```

### 3.2 macOS（Apple Silicon / Intel）

> ⚠️ hub 自带的那条 Linux 命令在 Mac 上**不能用**：`install.sh` 只认 systemd/OpenRC，且它从 `${SERVER}/agent/$ARCH` 拉的是 Linux ELF。macOS 必须走本仓库的 darwin 二进制 + launchd。

```bash
# 1) 下载并清 Gatekeeper 隔离属性（未签名二进制首次运行会被拦）
curl -fsSL -O https://github.com/99969/monitor-agent/releases/download/v1.1.0/monitor-agent-darwin-x86_64-v3
xattr -dr com.apple.quarantine monitor-agent-darwin-x86_64-v3   # arm64 换对应文件名

# 2) 前台跑一次，确认面板“在线”（Ctrl+C 退出）
./monitor-agent-darwin-x86_64-v3 --server https://<hub> --token <token> --interval 1

# 3) 确认无误后装成 launchd 常驻（关掉终端也不掉）
sudo install -m 755 monitor-agent-darwin-x86_64-v3 /usr/local/bin/monitor-agent
sudo sh -c 'cat > /Library/LaunchDaemons/com.monitor.agent.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.monitor.agent</string>
  <key>ProgramArguments</key><array>
    <string>/usr/local/bin/monitor-agent</string>
    <string>--server</string><string>https://<hub></string>
    <string>--token</string><string><token></string>
    <string>--interval</string><string>1</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/var/log/monitor-agent.log</string>
  <key>StandardErrorPath</key><string>/var/log/monitor-agent.log</string>
</dict></plist>'
sudo chown root:wheel /Library/LaunchDaemons/com.monitor.agent.plist
sudo chmod 600 /Library/LaunchDaemons/com.monitor.agent.plist
sudo launchctl load -w /Library/LaunchDaemons/com.monitor.agent.plist
```

- macOS 网卡命名为 `en0` / `en1`（Wi-Fi、以太网、雷雳），可用 `--iface en0` 限定。
- agent 只读取普通 sysctl / Mach 接口，**不需要 root**，也无需 Full Disk Access。
- 动态依赖仅有 `libiconv.2` 与 `libSystem.B`，其余（含 ring/rustls 的 TLS）全部静态链接，干净 Mac 可直接运行。

### 3.3 MT7621（mipsel，OpenWrt 路由器）

`monitor-agent-mt7621` 是 **32 位 MIPS（MIPS32R2）、小端、soft-float、静态链接 musl** 的 ELF，**无任何外部依赖**，拷到路由器即可运行。

```bash
# 1) 传到路由器（OpenWrt 默认 root）
scp monitor-agent-mt7621 root@192.168.1.1:/usr/bin/monitor-agent
ssh root@192.168.1.1 "chmod +x /usr/bin/monitor-agent"

# 2) 前台测试，确认上线
ssh root@192.168.1.1 "/usr/bin/monitor-agent --server https://<hub> --token <token> --interval 5"

# 3) 常驻（OpenWrt 用 /etc/rc.local，重启后自动拉起）
ssh root@192.168.1.1 "echo '/usr/bin/monitor-agent --server https://<hub> --token <token> --interval 1 &' >> /etc/rc.local"
```

> 若路由器为 procd 体系且希望由 procd 托管，可写成 `/etc/init.d/monitor-agent` 脚本（启用 `procd_set_param command` 指向该二进制）；上面 `/etc/rc.local` 是最简单可靠的方案。

---

## 4. 从源码交叉编译

**宿主环境（Windows 沙盒，无 MSVC / 无 WSL）统一依赖：**

- Rust：`rustup toolchain install nightly-x86_64-pc-windows-gnu --profile minimal --component rust-src`（**必须用 gnu 宿主**而非 msvc，否则 msvc 的 `link.exe` 会被 Git Bash 的 `link` 抢占而失败）。
- 交叉 C 编译器 / 链接器：**Zig 0.13.0**（`zig cc` 自带 Mach-O / ELF 链接器，无需 Xcode 或 GNU 交叉工具链）。
- 源码：`monitor-probe/agent`（即本仓库对应的 `monitor-agent` v1.1.0）。

### 4.1 macOS（aarch64 / x86_64）

关键配方：

| 环节 | 方案 |
|---|---|
| Rust 目标 std | `nightly` + `-Z build-std=std,panic_abort`（`rustup` 的 darwin std 预编译包拉不到，改用本地 `rust-src` 从源码编 std） |
| macOS SDK | 只需链接期用到的 `.tbd` 文本桩，从 `phracker/MacOSX-SDKs`（MacOSX11.3.sdk）抽取约 35 个文件组成最小 sysroot |
| 链接期 sysroot | Zig 的 `-isysroot` 不生效，须显式 `-L <sdk>/usr/lib -F <sdk>/System/Library/Frameworks` |
| 宿主链接 | 显式 `+nightly-x86_64-pc-windows-gnu`，让 gnu 宿主用 rustc 自带 `rust-lld` 链接 build script（避免 `+nightly` 回落到 msvc） |

```bash
cd build/agent
cargo +nightly-x86_64-pc-windows-gnu build \
      -Z build-std=std,panic_abort --release \
      --target aarch64-apple-darwin
# x86_64 同理换 --target x86_64-apple-darwin
```

两个目标均零 warning 通过。相关封装见 `build/zig-cc-darwin.py`（Zig wrapper）、`build/fetch_sdk.py`（拉 `.tbd`）。

采集层实现要点（`collect_darwin.rs`）：

- 网卡流量**不能**用 `getifaddrs` 的 32 位 `ifa_data`（千兆链路几分钟回绕一次，会污染 hub 的速率累加），改用 `NET_RT_IFLIST2` routing socket 解析 64 位 `if_data64`。
- `vm.swapusage` 单位是**字节**不是页（Apple 自己的 `sysctl(8)` 也直接除 1024² 出 MiB，没乘 pagesize），乘上去会让 swap 虚高 4096 倍。
- APFS 多卷按底层 `diskN` 去重后再累加磁盘容量。

### 4.2 MT7621（mipsel soft-float）

关键配方：

- 目标：`-Z build-std=std,panic_abort --target mipsel-unknown-linux-musl`（32 位 MIPS 已从预编译 std 降级为 Tier3，必须 build-std）。
- mipsel 链接器：`zig cc -target mipsel-linux-musl`。**Zig 的 Windows 包不自带 musl 的 crt 启动对象**，需从 `musl.cc` 的 `mipsel-linux-musl-cross.tgz` 抽出 `crt1.o / crti.o / crtn.o / libc.a / libgcc.a` 放进 `mipsel-sysroot/lib`，链接器 `-L` 指向它；`-lunwind` 由 Zig 自带。
- **浮点**：必须给 Zig 传 `-msoft-float -mabi=32`（否则默认 double-float，与 Rust 的 soft-float 对象 ABI 冲突，链接期报重复/错位符号）。
- 链接 wrapper（`build/zig-cc-mipsel.py`）必须：① 过滤掉 rustc 传来的 `--target=`；② **删掉** rustc 显式传入的裸 crt 对象（`crt1.o/crti.o/crtbegin.o/crtend.o/crtn.o`），让 Zig 自己只生成一份（否则重复符号 `__start`/`_start`/`_init`）；③ 注入 `-L mipsel-sysroot/lib`。
- 宿主（gnu）链接器：winlibs MinGW `mingw64/bin/gcc.exe`（Zig 的 gnu 目标不自带 `kernel32`/`msvcrt` 导入库）；需 `RUSTFLAGS=-C target-feature=+crt-static` 避免找 `libgcc_s.so`。
- 坑：`ring` 的 C 对象若早于 `-msoft-float` 被缓存会编成 double-float → 清 `target/.../release/build/ring` 重编。

产物 `build/build_out/monitor-agent-mt7621`：ELF32 / LSB / `e_machine=8 (EM_MIPS)` / MIPS32R2 / 静态链接 / soft-float。

---

## 5. 上报字段契约（与 Linux 版一致）

```
Facts   : hostname os kernel arch virt cpu_name cpu_cores mem_total
          swap_total disk_total agent_version ipv4 ipv6
Metrics : boot_id iface uptime cpu load[3] mem_total mem_used
          swap_total swap_used disk_total disk_used
          net_rx_total net_tx_total net_rx net_tx tcp udp procs
```

macOS 仅有的语义差异：`swap` 是 `/private/var/vm` 下按需增长文件、总量每轮重读；`boot_id` 用 `kern.boottime`（epoch 秒）代替；`virt` 仅识别 Apple 虚拟化框架；挂载遮蔽提示恒为空。MT7621 走标准 Linux musl 路径，字段与 Linux 完全一致。

---

## 6. 许可

沿用上游 `monitor-probe/agent` 的 MIT 许可。
