# `menu.sh l` 补丁审核报告

**审核对象**：fscarmen/warp `menu.sh`（`bash menu.sh l`）的「N2/N3 补丁版」
**审核日期**：2026-09-13
**审核方式**：逐字 diff + N2/N3 现网取证 + N1 真机 A/B 复现 + 循环逻辑回归测试

---

## 0. 一致性核对（记录 ↔ 实物）

| 项 | 记录声称 | 实测 | 结论 |
|---|---|---|---|
| 补丁版字节数 | 129935 | 129935 | ✅ |
| N2 / N3 一致 | 一致 | sha256 均为 `42485540f25643c1d23191e8e1a761f36a3b22e2c08ff7bd1dbaed704f3bea35` | ✅ |
| 改动组数 | 共 5 组（正文实列 6 条） | 上游→补丁版 diff = **5 个 hunk**（1604 / 2412 / 2431 / 2451 / 2504） | ✅ 一致（第 6 条就落在第 5 个 hunk 内） |
| 上游原版 | 未记录 | 129066 B，sha256 `4fc224265e7e6f459eb0b72f6b74f3f95a78256b9c5f2e507953fae14e8c9bf5` | — |

> 结论：**修改记录与实物 100% 对得上**，没有"记录里有、文件里没有"或反之的情况。下面按记录编号逐条审核。

---

## 1. 结论速览

| 编号 | 改动 | 判定 | 说明 |
|---|---|---|---|
| F1 | `systemctl is-enabled/is-active` 加 `2>/dev/null`（~1607） | ✅ 正确（仅降噪） | 只消除了 `Failed to get unit file state…` 的噪声，不改控制流 |
| F2 | `settings()` 的 `sed` 加 `[ -f ]` 防丢文件（~2416） | ❌ **未达目的** | 判定的路径在 EL9 上根本不存在 → 守护 hook **从未**写入。上游是"报错跳过"，补丁版变成"静默跳过" |
| F3 | `warp-cli registration show 2>/dev/null`（~2434） | ✅ 正确（仅降噪） | 免费账户提示不再夹带 stderr |
| F4 | Luban 块全面降级致命性（~2454-2470） | ⚠️ **方向对、后果严重** | 真机证明核心症状被修好；但把循环唯一出口删掉 → **拿不到 IPv4 时永久挂死**（P0） |
| F5 | EPEL 前置 + `systemctl start` 加壳 + socket 等待加超时（~2505-2515） | ⚠️ 三取其一 | EPEL 与超时 ✅；`systemctl start` 的 `[ -f ]` 判断恒假（P0），且新增容错后"装不上"会变成"看起来成功"（P1） |
| F6 | `CLIENT=2` 分支同样防护（~2524） | ❌ 同 F2/F5 | 判断恒假，该行永不执行（当前靠 RPM `%post` 里的 `systemctl start` 侥幸生效） |

**总评**：这是一次**方向完全正确、且解决了用户实际痛点的补丁**，真机验证从 `EXIT=1` 变成 `EXIT=0`、
`/usr/bin/warp` 正常生成、WARP 出口生效。但其中 2 个判断路径写在 Rocky 上不存在的文件上，
以及 1 处删掉循环出口导致的挂死，都必须再修一轮才算真正闭环。

---

## 2. 验证方法与真机证据

### 2.1 实验台：全新 Rocky Linux 9 服务器

N1 是 Rocky Linux 9.8 / aarch64 全新服务器（未装 cloudflare-warp）：

* `rpm -q cloudflare-warp` → not installed；无 `/usr/bin/warp*`
* `systemctl show warp-svc` → `LoadState=not-found`；无 `/etc/wireguard/`
* **无 IPv6 默认路由**（`ip -6 route show default` 为空）——正是触发本 bug 的机房特征

### 2.2 上游原版：完整复现故障

```bash
wget -N https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh   # sha256 = 4fc22426…
bash menu.sh l
```

`notes/`(真机日志)（946 行）关键片段：

```
 582: Created symlink /etc/systemd/system/multi-user.target.wants/warp-svc.service → …
 939: sed: can't read /usr/lib/systemd/system/warp-svc.service: No such file or directory
 940:  Step 2/3: Setting Client Mode
 942:  Try 1
 943:  Try 2
 944:  Try 3
 945:  There have been more than 3 failures. The script is aborted.
 946: EXIT=1
```

失败后状态（`notes/`(真机日志)）：

| 检查项 | 结果 |
|---|---|
| `/usr/bin/warp` | **不存在** ← 用户报告的"安装不完整" |
| `/etc/wireguard/` | 空目录（`menu.sh` 未被搬进去，`language` 未写） |
| `CloudflareWARP` 接口 / ip rule 5207/5208 | 都没有 |
| 服务本身 | 已装、已 enabled、active，socket 也在 |

**根因确认**：RPM 装好了、服务也起来了，但脚本在 Luban 双栈校验
（`until [[ -n "$CFWARP_WAN4" && -n "$CFWARP_WAN6" ]]`）里因为**没有 IPv6** 而重试 3 次后
调用 `error()`（`error() { echo … && exit 1; }`）→ 直接 exit 1 → 后面的
`ln -sf /etc/wireguard/menu.sh /usr/bin/warp` 永远执行不到。
这与你的判断（"无 v6 机房不阻塞"是核心）**完全一致**。

### 2.3 补丁版：同机跑通

复位后（`rpm -e cloudflare-warp`、删除 unit/软链/`/etc/wireguard`，并**额外 `dnf remove epel-release`**
以复现"Rocky 9 缺 EPEL"场景），跑补丁版：

```
 10:  Step 1/3: Installing WARP Client...
      └─ (被 /dev/null 吞掉的) `yum -y install epel-release`  →  dnf history ID 36  Install 1 EE
      └─ yum -y install cloudflare-warp → Install 1 Package, aarch64 2026.7.1377.0-1.el9
 58:  Step 2/3: Setting Client Mode
 60:  Try 1
 61:  Got the WARP IP successfully
 62:  Create shortcut [warp] successfully      ← 目标达成
 66:  Congratulations! WARP Free Linux Client is working.
 69:  WARP Free IPv4: <WARP-IP> AE  Cloudflare, Inc.
 70:  WARP Free IPv6:                        ← 无 v6，正常留空
 95: EXIT=0
```

跑完后的状态（`notes/`(真机日志)）：

| 检查项 | 结果 |
|---|---|
| `/usr/bin/warp` | ✅ → `/etc/wireguard/menu.sh`（129935 B） |
| `/etc/wireguard/language` | ✅ `E` |
| `CloudflareWARP` | ✅ `172.16.0.2/32`，`warp-cli status` = **Connected** |
| Luban 规则 | ✅ `5207: from all lookup main suppress_prefixlength 0` / `5208: from 172.16.0.2 lookup 51820`（v6 对称） |
| 出口验证 | `curl --interface CloudflareWARP` → `<WARP-IP>`；默认出口 → `<VPS-IP>`（SSH 通道未受影响）|
| **`/etc/systemd/system/warp-svc.service`** | ⚠️ **没有 `ExecStartPost=warp z` / `ExecStop=warp x`**，且 `/usr/lib/systemd/system/warp-svc.service` 依然不存在 |

### 2.4 现网取证：N2 / N3

| 机器 | unit 里有无 hook | Luban 规则当前 | 说明 |
|---|---|---|---|
| N2 | ✅ 有（588 B，手工改的，`ExecStartPost` 已执行 status=0） | ❌ **没有**（`ip rule` 里无 5207/5208，table 51820 为空） | 见下 |
| N3 | ❌ 没有（533 B） | ✅ 有（5207/5208） | 靠自建 `warp-rules.service` + `warp-failover.service` 维护 |

N2 的 journal 解释了规则为什么会"自己消失"：

```
Sep 13 22:27:12 dbc warp-svc[…]: DelRule { tables: [51820] };
Sep 13 22:27:12 dbc warp-svc[…]: DelRoute(Output interface: 29; );
```

即 **Cloudflare 客户端自身的状态机在网络变化/断开重连时会删掉 51820 表的路由与规则**。
所以"往 vendor unit 里 sed 插 hook"这个思路本身就不耐久：

1. 客户端会自行 `DelRule`（N2 实测漂移）；
2. `dnf upgrade cloudflare-warp` 的 `%post` 会 `cp /opt/cloudflare-warp/warp-svc.service /etc/systemd/system/warp-svc.service`，
   **每次升级都会覆盖掉插进去的 hook**；
3. 而且在 Rocky 上这条 `sed` 因为路径不对，从来就没成功过（上游日志 939 行就是它）。

### 2.5 本地循环回归测试

`./tests/luban_loop_test.sh`（把两段循环逐字复制、桩掉外部命令、模拟"拿不到 IPv4"）：

```
▶ 上游原版（error 退出）      结果: 12s 内自行退出 → exit=1        ✓
▶ 补丁版（warning 不退出）    结果: 运行 12s 仍未退出 → 死循环     ✗
▶ 建议修复版（i=j 时 break）  结果: 12s 内自行退出 → exit=0        ✓  ← 验证 §6 的修法
```

---

## 3. 逐条审核

### F1 · `systemctl is-enabled/is-active warp-svc` 加 `2>/dev/null`（~1607 行）

```diff
-    [ "$(systemctl is-enabled warp-svc)" = enabled ] && CLIENT=2
-    if [[ "$CLIENT" = 2 && "$(systemctl is-active warp-svc)" = 'active' ]]; then
+    [ "$(systemctl is-enabled warp-svc 2>/dev/null)" = enabled ] && CLIENT=2
+    if [[ "$CLIENT" = 2 && "$(systemctl is-active warp-svc 2>/dev/null)" = 'active' ]]; then
```

**判定：✅ 正确，但只是降噪。**

* 成因属实：unit 不存在时 `systemctl` 会往 stderr 打 `Failed to get unit file state for warp-svc.service: No such file or directory`（N1 首次探测已复现）。
* 这段代码只在 `[ -x "$(type -p warp-cli)" ]` 成立时才会走到——也就是"二进制在、unit 没了"的
  半残状态（正是你们反复手工 `rpm -e` / 重装后遇到的状态）。
* 它不影响 `CLIENT` 判定结果，属于纯噪声修复。
* 顺带记录：脚本里另有约 30 处 `systemctl` 调用同样会把该噪声打到屏幕上，如需彻底干净建议统一加 `2>/dev/null`（低优先级）。

### F2 · `settings()` 的 `sed` 防丢文件（~2416 行）—— ❌ 未达目的

```diff
-    [ "$IS_LUBAN" = 'is_luban' ] && sed -i '…' /usr/lib/systemd/system/warp-svc.service && systemctl daemon-reload
+    if [ "$IS_LUBAN" = 'is_luban' ] && [ -f /usr/lib/systemd/system/warp-svc.service ]; then
+      sed -i '…' /usr/lib/systemd/system/warp-svc.service && systemctl daemon-reload
+    fi
```

**判定：❌ 判断路径错误 → 功能缺口被静默掩盖。**

* 事实（N1 真机）：RPM 的 `%post` 是
  `cp /opt/cloudflare-warp/warp-svc.service /etc/systemd/system/warp-svc.service` + `systemctl enable/start warp-svc`。
  **`/usr/lib/systemd/system/warp-svc.service` 在 EL9 上永远不存在**（Debian/Ubuntu 的 deb 才装在那里）。
* 后果：
  * 上游：`sed` 报错（日志 939 行），但**脚本不会因为这条 `&&` 短路而退出**（没有 `set -e`），
    真实损失是"`ExecStartPost=warp z` / `ExecStop=warp x` 没写进去"。
  * 补丁版：判断恒假 → **静默跳过**，损失一模一样，只是不再报错。
* 真机实证：补丁版跑完的 N1，unit 只有 533 B、`systemctl show -p ExecStartPost` 为空；
  N2 是 588 B、有 hook —— 差别来自人工改，而不是这行补丁。
* 更根本的问题（建议一并改）：即使把路径写对，这个方案也不耐久 —— 客户端会自行删规则（2.4 节 N2 journal），
  且 `dnf upgrade cloudflare-warp` 会覆盖 unit。

**建议**：不要 sed vendor unit，改成独立幂等单元（N3 已经在用）：

```bash
# /etc/systemd/system/warp-rules.service  (Type=oneshot, RemainAfterExit=yes, After=warp-svc.service)
IF=CloudflareWARP; IP=172.16.0.2; TBL=51820
[ -d /sys/class/net/$IF ] || exit 1
ip route replace default dev $IF table $TBL 2>/dev/null || true
ip rule show | grep -q "lookup main suppress_prefixlength 0" || ip rule add from all lookup main suppress_prefixlength 0 pref 5207
ip rule show | grep -q "from $IP lookup $TBL"                 || ip rule add from $IP lookup $TBL pref 5208
```

### F3 · 免费账户 fallback 的裸 `warp-cli` 静默（~2434 行）

**判定：✅ 正确（降噪）。** 该分支只在 fallback 免费账户时提示，stderr 内容无信息量。

### F4 · Luban 块全面降级致命性（~2454-2470 行）—— ⚠️ 核心修复，但引入 P0

拆成 4 个子改动分别判定：

| 子项 | 改动 | 判定 |
|---|---|---|
| 4a | `ip_case d is_luban` → `ip_case d is_luban >/dev/null 2>&1`（循环内外两处） | ✅ 无副作用 |
| 4b | `until [[ -n "$CFWARP_WAN4" && -n "$CFWARP_WAN6" ]]` → `until [ -n "$CFWARP_WAN4" ]` | ✅ 条件放宽正确（无 v6 不再阻塞） |
| 4c | 两处 `error " $(text 52) "` → `warning …` | ✅ 目的达成 |
| 4d | `i=j` 分支删除 `disconnect/rule_del/error`，改为两条 `warning` | ❌ **把循环唯一出口删掉了** |

* 4a 说明：`ip_case … is_luban` 只做全局变量赋值（`CFWARP_WAN4/6`、`CFWARP_COUNTRY4/6` …），
  重定向不影响赋值；N1 真机最终输出 `WARP Free IPv4: <WARP-IP>` 说明变量确实回填了。✅
* 4d 说明（**P0**）：循环体里唯一的出口就是 `until` 的条件本身。
  上游在 `i=j` 时 `error`（exit 1）终止；补丁版改 `warning` 之后，如果 `CFWARP_WAN4`
  **始终为空**（WARP 连不上、被墙、免费账户限流、IP 查询 API 不通……），循环将
  **永远重试下去**：`disconnect → rule_del → sleep 2 → connect → wait_for interface → rule_add → ip_case`
  无限循环。本地回归测试已证实（2.5 节）。
  更糟的是：挂死时流程同样走不到脚本末尾 → **`/usr/bin/warp` 仍然不会生成**，
  也就是你最初报告的症状在"WARP 拿不到 IP"的机房里会原样复发。

**建议（1 行即可闭环）**：

```bash
        if [ "$i" = "$j" ]; then
          warning " $(text 13) "
          warning " WARP IP 获取失败但已尽力，继续完成安装 "
          CFWARP_IP_FAILED=1
          break                      # ← 补上出口
        fi
      done
      [ -z "$CFWARP_IP_FAILED" ] && info " $(text 14) "     # text 14 = “Got the WARP IP successfully”
```

注意 `info " $(text 14) "` 在循环之后无条件打印，加了 `break` 之后要一并加判断，
否则会出现"明明没拿到 IP 却提示 Got the WARP IP successfully"。脚本末尾的"成功"提示同理，
建议统一挂到 `CFWARP_IP_FAILED` 上（见 P1-2）。

### F5 · cloudflare-warp 安装路径（~2505-2515 行）

```diff
 ${PACKAGE_UPDATE[int]}
+if grep -q "CentOS\|Fedora" <<< "$SYSTEM"; then
+  ${PACKAGE_INSTALL[int]} epel-release >/dev/null 2>&1 || true
+fi
 ${PACKAGE_INSTALL[int]} cloudflare-warp
-[ "$(systemctl is-active warp-svc)" != active ] && ( systemctl start warp-svc; sleep 2 )
+[ -f /usr/lib/systemd/system/warp-svc.service ] && { [ "$(systemctl is-active warp-svc 2>/dev/null)" != active ] && { systemctl start warp-svc >/dev/null 2>&1; sleep 2; }; }
+WARP_SOCK_TRY=0
 until [ -e /run/cloudflare-warp/warp_service ]; do
+  WARP_SOCK_TRY=$((WARP_SOCK_TRY+1))
+  [ "$WARP_SOCK_TRY" -ge 30 ] && { echo "warp_service 60s 未就绪…" >&2; break; }
   sleep 2
 done
```

| 子项 | 判定 | 证据 |
|---|---|---|
| EPEL 前置 | ✅ **必要且有效** | `dnf -q --disablerepo='epel*' repoquery --available libappindicator-gtk3` → **空**（该依赖只由 EPEL 提供）；N1 上先 `dnf remove epel-release` 再跑补丁版，dnf history 出现 `ID 36 install epel-release` → `ID 37 install cloudflare-warp` 成功 |
| `SYSTEM` 判断写法 | ✅ 可用（可读性差） | `REGEX[2]` 把 `centos\|red hat\|kernel\|alma\|rocky` 全部映射成 `RELEASE[2]="CentOS"`，所以 Rocky 上 `$SYSTEM = 'CentOS'`，`grep "CentOS\|Fedora"` 命中 |
| `systemctl start` 加 `[ -f ]` 壳 | ❌ **恒假，该行永不执行** | 同 F2：EL9 上是 `/etc/systemd/system/warp-svc.service`；当前能起来是因为 RPM `%post` 自带 `systemctl enable/start warp-svc`（真机 `rpm -q --scripts` 第 193-194 行） |
| socket 等待超时 | ✅ 正确 | 避免"服务起不来就无限等"；N1 上未触发（socket 正常出现） |
| 装包结果无校验 | ⚠️ P1 | `${PACKAGE_INSTALL[int]} cloudflare-warp` 没有 `||` 兜底，加上新增的容错后，一旦安装失败，脚本会一路"成功"跑完 |

**建议**：把"判文件"换成"判状态"，并补装后校验：

```bash
${PACKAGE_INSTALL[int]} cloudflare-warp
systemctl start warp-svc >/dev/null 2>&1 || true          # 直接尝试，无需判文件
[ -x "$(command -v warp-cli)" ] || warning " warp-cli 未就绪：cloudflare-warp 可能没装上，请手动检查 "
```

### F6 · `CLIENT=2` 分支同样防护（~2524 行）

**判定：❌ 同 F2/F5（同一行代码重复一次）。**

* `[ -f /usr/lib/systemd/system/warp-svc.service ]` 恒假 → 该分支下"服务没起就拉起来"的逻辑完全失效；
  一旦服务真的没起来，后面的 `settings()` 会：`warp-cli registration new` 失败 → 重试 10 次 →
  走 fallback 把硬编码免费账户写进 `/var/lib/cloudflare-warp/reg.json` → 最后照样打印"安装成功"。
* 修改建议同 F5：直接 `systemctl start warp-svc >/dev/null 2>&1 || true`，并把成功提示与真实状态挂钩。

---

## 4. 其它发现（补丁之外，按优先级）

### P0-1 循环挂死（= F4-4d）
见上。这是补丁**新引入**的问题，也是本轮审核中最需要立刻回滚的一项。

### P0-2 `/usr/lib/systemd/system/warp-svc.service` 恒不存在（= F2 / F5 / F6）
见上。建议后续统一以
`systemctl show -p FragmentPath --value warp-svc`（或 `systemctl list-unit-files`）判存在性，
不要硬编码发行版路径。

### P1-1 `/usr/bin/warp` 仍在流程最末尾创建
`mv … && chmod … && ln -sf … /usr/bin/warp` 位于 `client_install()` 倒数第几行，只要中途
`exit`、挂死、Ctrl-C、SSH 断线、超时被杀，就会留下"服务装好了但没有 `warp` 命令"的状态
——这正是你最初报告症状的**机制**。补丁只堵了几个 `error`，机制没变。
建议：`mkdir -p /etc/wireguard` 之后立刻软链，或用 `trap 'ln -sf …' EXIT` 兜底。

### P1-2 "装不上/没连上" 仍会打印成功
补丁把多处 `error` 降级为 `warning` 之后，末段无条件输出
`Congratulations! WARP Free Linux Client is working.`（N1 日志 66 行）。
即使 `cloudflare-warp` 没装成、socket 没出现、IP 没拿到，用户看到的仍是"成功"。
建议：用一个 `INSTALL_OK=0/1` 汇总各子步骤结果，失败时把末段换成明确失败提示（含排查命令）。

### P1-3 Luban 规则持久化方案本身不可靠
* 客户端会在网络变化/重连时 `DelRule { tables: [51820] }`（N2 journal 实证）；
* `dnf upgrade cloudflare-warp` 的 `%post` 会覆盖 `/etc/systemd/system/warp-svc.service`，抹掉 sed 插入的 hook；
* 结论：用"独立幂等 unit + 必要时 watchdog 周期校正"（N3 的 `warp-rules.sh` + `warp-failover.sh`）替代 sed。

### P2-1 N2 现网状态漂移（仅报告，未改动）
N2 的 unit 有 hook、`ExecStartPost` 也确实执行成功（22:27:13，status=0），但**当前**
`ip rule` 里没有 5207/5208、table 51820 为空 —— 该机此刻的 Luban 路由状态与设计不符。
需要的话可以让 `warp z` 重跑一次（`bash /etc/wireguard/menu.sh z`）并观察是否再次被客户端删掉。

### P2-2 EPEL 步骤吞错
`${PACKAGE_INSTALL[int]} epel-release >/dev/null 2>&1 || true` 无法区分"装好了"和"没装上"；
若机器上 `epel-release` 已装但 EPEL 仓库被 disable，这条也无法救场。
建议补一句 `dnf -q --disablerepo='*' --enablerepo='epel*' repolist >/dev/null 2>&1 || warning "EPEL 不可用"`。

### P2-3 `ExecStartPost=warp z` 会拖慢服务启动
`warp z` 跑的是整个 `menu.sh`，进入 `case z` 之前还要走 `check_cdn`（最长等 120 s）等逻辑。
N2 实测 2 s 完成（网络好），网络差时可能把 `warp-svc` 启动拖到分钟级。
若采纳 P1-3 的独立 unit，建议在里面直接内联 `ip rule/route`（像 N3 的 `warp-rules.sh` 那样），别再回调整个脚本。

### P3-1 fallback 免费账户是 2023 年的硬编码账户
`/var/lib/cloudflare-warp/reg.json` 里写死的 `registration_id/api_token/license` 全机房共用，
建议仅在明确告知用户后使用（当前只在注册失败 10 次后触发）。

### P3-2 脚本每次运行都上报统计
`statistics_of_run-times update menu.sh` 会访问 `https://stat.cloudflare.now.cc/updateStats`。
生产节点如需减少外联，可注释掉。

### P2-4 「Registration Missing」大小写不匹配（真机新发现）
新版客户端（`2026.7.1377`）输出的是
`Status update: Unable / Reason: Registration Missing due to: Manual deletion`（**大写 M**），
而上游判的是小写 `'Registration missing'` → 该分支在新客户端上永远不触发。
修法：`=~ [Rr]egistration\ [Mm]issing`（已本地验证两种拼写都能匹配）。

### P2-5 「注册丢失→重跑自愈」分支是死代码（真机新发现）
`client_install()` 顶部是 `[ "$CLIENT" -ge 2 ] && error " $(text 85) "`，
而同一个函数下面才有 `elif [[ "$CLIENT" = '2' && … Registration missing ]]`。
CLIENT=2（已安装+服务 enabled+active）时先被顶部 `error` 拦下 → 下面的 elif 永远不可达。
真机验证：`warp-cli registration delete` 后再跑 `bash menu.sh l` 会直接打印
`Client was installed. …` 并 `EXIT=1`，连 settings 都进不去。

---

## 5. 修复版 `fixed/menu.sh`（本轮产出）

在 N2/N3 补丁版基础上落地了 7 处修改，全部在 N1 真机验证过：

| # | 修改 | 对应问题 |
|---|---|---|
| 1 | Luban 循环 `i=j` 时 `CFWARP_IP_FAILED=1` + `break`；`text 14` 改为条件打印 | P0-1 挂死 |
| 2 | 新增 `warp_svc_unit()`（`systemctl show -p FragmentPath --value`）/ `warp_svc_start()`，替换三处 `[ -f /usr/lib/systemd/system/warp-svc.service ]` | P0-2 路径恒假 |
| 3 | Luban 规则持久化改为独立幂等 `warp-rules.service` + `/usr/local/bin/warp-rules.sh`（已存在则不覆盖），调用点放在接口就绪之后 | F2 功能缺口 / P1-3 |
| 4 | `mkdir -p /etc/wireguard` 之后立即 `mv + chmod + ln -sf /usr/bin/warp` | P1-1 半成品 |
| 5 | 末尾按真实状态判定（`warp-cli` / socket / 接口 / WAN4），未就绪改打印排查指引 | P1-2 假成功 |
| 6 | EPEL 装完显式 `dnf repolist` 校验；`cloudflare-warp` 安装失败给 warning；socket 超时给 warning | P2-2、F5 |
| 7 | 再入分支大小写 `[Rr]egistration\ [Mm]issing`；顶部 `CLIENT -ge 2` 的 `error` 放行"注册丢失" | P2-4、P2-5 |

### 真机验证结果（N1，Rocky 9.8 aarch64）

| 用例 | 命令 | 结果 | 证据 |
|---|---|---|---|
| A. 全新安装 | `bash menu.sh l`（未装 cloudflare-warp） | `EXIT=0`，`/usr/bin/warp` 生成，WARP Connected，出口 `<WARP-IP>`，ip rule 各 1 条 | `notes/`（真机日志） |
| B. 注册丢失重跑自愈 | `warp-cli registration delete` → `bash menu.sh l` | `EXIT=0`，自动重新注册并连上；`warp-rules.service` **enabled + active**（规则恰好 1 条，无重复） | `notes/`（真机日志） |
| C. WARP 拿不到 IP | `/etc/hosts` 把 IP 查询域名指向 127.0.0.1 → `bash menu.sh l` | **62 秒自行退出**（Try1/2/3 后 break），打印"安装未完成…"排查指引，**不打印 Congratulations**，`EXIT=0` | `notes/`(真机日志) |

对照：用例 C 在补丁版上若走全新安装路径会**永久挂死**（本地逐字回归测试 `tests/luban_loop_test.sh` 已证明）；
用例 B 在补丁版上根本进不去（死代码），会 `EXIT=1`。

> 注：A 轮首次尝试时 `warp-rules.service` 曾因"接口尚未就绪就调用"而 failed，
> 修复版已把调用点后移到 Luban 分支末尾、并在接口未就绪时不拉起，B 轮确认 active。

---

## 6. 建议的下一步（按性价比排序）

1. **加 1 行 `break`** 修掉 F4 挂死（P0-1）—— 这是唯一"现在就会咬人"的回归。
2. 三处 `/usr/lib/systemd/system/warp-svc.service` 判断改成 `systemctl show -p FragmentPath --value`
   或直接无条件 `systemctl start warp-svc >/dev/null 2>&1 || true`（P0-2）。
3. 装包/启动后做一次真实性校验（`command -v warp-cli`、`[ -e /run/cloudflare-warp/warp_service ]`），
   失败时给出明确警告，别再打 "Congratulations"（P1-2）。
4. 把 `/usr/bin/warp` 软链提前或用 `trap` 兜底（P1-1）。
5. Luban 规则持久化改用独立幂等 unit（可复用 N3 的 `warp-rules.sh`），并评估是否被客户端删规则（P1-3）。

第 1、2 项改完，建议在 N1 上按 `README.md` 的复现流程再跑一遍 A/B（上游失败 / 补丁成功），
并额外验证"WARP 不可达"场景：用 `nft`/`iptables` 临时 DROP `162.159.192.0/24` 与
`2606:4700:d0::/48`，确认脚本**在有限时间内退出**且 `warp` 命令仍在。

---

## 附录 A · 本目录文件

| 路径 | 说明 |
|---|---|
| `upstream/menu.sh` | 上游原版（gitlab main） |
| `fixed/menu.sh` | 修复版（可直接运行 / 作为一键源） |
| `patches.diff` | 上游 → 补丁版完整 diff（5 hunk） |
| `patches-fixed.diff` | 补丁版 → 修复版完整 diff |
