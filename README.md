# warp-sh-fixed — fscarmen/warp `menu.sh` 修复版

对 `https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh`（v3.2.7）的 Rock/EL9 修复版。

## 为什么有这个项目

上游 `menu.sh` 在 **Rocky/Alma/CentOS 9（RPM）** 上运行 `l`（官方 client、WARP 模式）时存在多处故障，
典型症状是 `bash menu.sh l` 以 `EXIT=1` 失败且不生成 `/usr/bin/warp`：

1. `settings()` 里 `sed -i ... /usr/lib/systemd/system/warp-svc.service` 无文件存在性保护，
   EL9 RPM 上该 unit 在 `/etc/systemd/system/`（`/usr/lib/...` 不存在）→ `sed: can't read` 直接中断流程。
2. `client_install` 的 RPM 路径未装 **EPEL**（依赖 `libappindicator-gtk3`）→ `dnf install cloudflare-warp` 失败。
3. 首次安装流程把冲突的 wg-quick/cloudflare-warp 卸载后重装，历史 `registration` 丢失时
   再入分支被顶部 `error` 挡成**死代码**，`warp` 命令永远不生成。
4. Luban（官方 client 全接管）对 IPv6 强校验 `until [[ -n $WAN4 && -n $WAN6 ]]`——
   无公网 IPv6 的机房必失败，装上也会被误判"未就绪"。
5. `warp o`（开关菜单）依赖 `wg-quick` 判断，官方 client 环境下误报"WARP has not been installed yet"。

修复后：`bash menu.sh l` 在 EL9 上 `EXIT=0`、`/usr/bin/warp` 正常生成、Luban 全部关键步骤
从 `error`（致命退出）降级为 `warning`（继续安装）、拿不到 WARP IP 时 62 秒内自行退出并给出排查指引。

完整逐条判定见 **[AUDIT.md](AUDIT.md)**。

## 一键运行

直接在目标服务器（root）执行即可，无需 clone：

```bash
# 一键进入交互菜单
bash <(curl -sSL https://raw.githubusercontent.com/wanjiban/warp-sh-fixed/main/fixed/menu.sh)

# 或跳过菜单直接以指定模式安装（推荐：Rocky 9 用官方 client 全接管）
bash <(curl -sSL https://raw.githubusercontent.com/wanjiban/warp-sh-fixed/main/fixed/menu.sh) l
bash <(curl -sSL https://raw.githubusercontent.com/wanjiban/warp-sh-fixed/main/fixed/menu.sh) c
bash <(curl -sSL https://raw.githubusercontent.com/wanjiban/warp-sh-fixed/main/fixed/menu.sh) e
```

脚本运行时会自动部署到 `/etc/wireguard/menu.sh` 并创建 `/usr/bin/warp` 软链，
后续直接用 `warp` 命令即可（`warp l` / `warp r` / `warp s` …）。

校验：文件 sha256 应为
`c3a22eea07031c8a20f39268a0d1a6410c195df992d8073f88cf475c7df5da24`。

## 目录结构

```
warp-sh-fixed/
├── AUDIT.md           ← 逐条判定与修复建议
├── README.md
├── fixed/menu.sh      ← ★ 修复版（可直接运行 / 作为一键源）
├── upstream/menu.sh   ← 上游原版（gitlab main，129066 B）
├── patches.diff       ← 上游 → 补丁版 的完整 diff
└── patches-fixed.diff ← 补丁版 → 修复版 的完整 diff
```

## 复现 / 验证

在另一台 Rocky 9 服务器上完整体验：

```bash
curl -sSL https://raw.githubusercontent.com/wanjiban/warp-sh-fixed/main/fixed/menu.sh | bash -s l
```