# HermesAgent 通用安装提示词（国内镜像优化版）

> **用途**：把本文档交给任意 AI Agent（或工程师），在**国内网络环境的 Linux 服务器**上从零安装最新版 **Hermes Agent**（v0.21+ PM 架构）。
> **特点**：完全自包含——不依赖任何预置脚本、不绑定特定机器；三个配套脚本已内嵌（第六节），复制保存即可用。
> **适用范围**：Ubuntu / Debian 系（glibc，x86_64 / arm64）。其他平台（macOS / Windows / Termux）见文末说明。
> **一句话结论**：GitHub Releases 与 PyPI 官方源在国内常被限速到几十 KB/s，直连安装会卡死数小时；改走 **Nous 官方镜像 + 阿里云 PyPI + 南大镜像 + npmmirror** 后，全流程约 **5～10 分钟**。

---

## 一、总纲：给 AI Agent 的执行契约

```text
你是 Hermes Agent 安装助手。目标：在目标 Linux 服务器上从零安装最新版 Hermes Agent
（v0.21+ PM 架构），全程规避 GitHub Releases / PyPI 官方源在国内的限速问题。

执行顺序：
1. 前置：安装 git/curl/build-essential/libatomic1，按需配置 2G swap（内存≤4G 建议）。
2. 测速判定（第二节「测速三连」）：确认目标机确实存在限速。
   - GitHub Releases 直连 或 PyPI 官方源 < 1MB/s → 启用本镜像方案（下文全流程）
   - 两者都正常（国际网络）→ 直接 curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash 即可，无需镜像
3. 检查旧版：~/.hermes/hermes-agent 或 /opt/hermes-agent 若存在但【无 .git、无 pm/ 目录】，
   是 v0.10 时代的"控制台快照版"，必须备份移走后重装（这类版本无法 update）。
4. 下载官方安装脚本到 ~/hermes-install.sh。
5. 依次执行，每步验证产物后再下一步：
   a. bash ~/hermes-install.sh --non-interactive --stage prerequisites   # 环境自检
   b. bash ~/hermes-install.sh --non-interactive --stage repository      # 克隆仓库（约 2 分钟）
   c. 保存并运行 stage-pm-artifacts.sh   # 预置 9 个工具包缓存 + bootstrap uv（~560MB，官方镜像）
   d. 保存并运行 reapply-cn-mirrors.sh   # 重写 uv.lock 下载 URL + npm/pip 镜像
   e. 保存并运行 run-stages.sh           # 顺序跑 venv→python-deps→config→products→complete
6. 验证：hermes --version / hermes doctor / hermes pm doctor，全绿才算完成。
7. 汇报：实际版本号、各阶段耗时、验证输出、遗留事项。

严禁：
- 使用 npm.taobao.org（已废弃）等旧源；
- 修改锁文件里 registry = "https://pypi.org/simple" 这些行（会令 uv --locked 判定锁过期）；
- 使用 v0.10 时代的社区"加速脚本"（uv pip install -e ".[all]" 老架构），会破坏 PM 管理导致 hermes update 失效；
- 用 pkill -f 粗暴杀安装进程（可能误伤同会话 SSH，把自己踢下线）——用精确 PID kill。
```

---

## 二、第 0 步：「测速三连」快速判定

在目标服务器上直接跑这三条命令（10 秒内出结果）：

```bash
# ① GitHub Releases（uv 二进制包，限速重灾区）
curl -sL -o /dev/null -w 'github: %{speed_download} B/s\n' -m 10 --range 0-3000000 \
  https://github.com/astral-sh/uv/releases/download/0.12.3/uv-x86_64-unknown-linux-gnu.tar.gz

# ② PyPI 官方文件站（Python 依赖，限速重灾区）
curl -sL -o /dev/null -w 'pypi:   %{speed_download} B/s\n' -m 10 \
  'https://files.pythonhosted.org/packages/b7/ce/149a00dd41f10bc29e5921b496af8b574d8413afcd5e30dfa0ed46c2cc5e/six-1.17.0-py2.py3-none-any.whl'

# ③ Nous 官方镜像（加速候选，按内容 sha256 寻址）
curl -sL -o /dev/null -w 'nous:   %{speed_download} B/s\n' -m 8 \
  https://hermes-assets.nousresearch.com/upstream/sha256/600cf9a742aca00d292673b16b5acffaa7b8c269a364ad0c2e79498dcb1fe101
```

**判定标准**：
- ① 或 ② 明显慢（< 1MB/s，常见 5～100KB/s）、③ 明显快（MB/s 级）→ **本镜像方案适用**
- ①②③ 都正常（> 5MB/s）→ 国际网络，直接用官方一行安装即可，本文后续步骤不需要
- ③ 也慢 → 检查出口网络/DNS；本方案仍可尝试 gh-proxy 兜底（内置降级）

---

## 三、镜像源清单（选型与实测速度）

| 源 | 地址 | 用途 | 实测速度参考 |
|---|---|---|---|
| **Nous 官方 artifacts 镜像** | `https://hermes-assets.nousresearch.com/upstream/sha256/<sha256>` | PM 工具包、uv（内容寻址+哈希校验，官方出品） | **4.6 MB/s** ✅ |
| **阿里云 PyPI（公网 HTTPS）** | `https://mirrors.aliyun.com/pypi/simple/` | Python 依赖 | **2.7～20 MB/s** ✅ |
| 阿里云 PyPI（ECS 内网） | `http://mirrors.cloud.aliyuncs.com/pypi/simple/` | 仅阿里云 ECS 内部 | 4.2 MB/s |
| **南大 python-build-standalone 镜像** | `https://mirror.nju.edu.cn/github-release/astral-sh/python-build-standalone` | Python 解释器（uv 镜像参数） | **13 MB/s** ✅ |
| **npmmirror** | `https://registry.npmmirror.com` | npm 包 / Node 二进制 | **7.5～10.7 MB/s** ✅ |
| 阿里云 Node 镜像 | `https://mirrors.aliyun.com/nodejs-release/` | Node 二进制（备用） | 3.5 MB/s |
| gh-proxy | `https://gh-proxy.com/<github原始url>` | GitHub 兜底代理 | 0.4～0.9 MB/s（降级备用） |
| ~~GitHub Releases 直连~~ | — | ❌ 常见限速 | **30～100 KB/s** |
| ~~files.pythonhosted.org~~ | — | ❌ 常见限速 | **5～47 KB/s** |
| ~~npm.taobao.org~~ | — | ❌ 已废弃，勿用 | 不可用 |

> **选型原则**：工具包优先走 Nous 官方镜像（安全等价官方源——同样的 sha256、同样的字节）；Python 依赖走阿里云（公网 HTTPS 通用性好，不限云厂商）；解释器走南大镜像；npm 走 npmmirror。

---

## 四、原理：为什么直连会卡死——PM 三层下载模型

Hermes v0.21+ 由 **PM 包管理器**负责下载，共三层，每层都可能卡在限速上。**关键认知：所有"镜像 fallback"只在网络错误时触发；限速不算失败，兜底永远不触发**——这就是"看起来卡死"的根因。

### 第 1 层：bootstrap uv（安装器 `ensure_uv()`）

- 从 GitHub Releases 下载 uv → 存到 `~/.hermes/tools/uv-<版本>-<target>/`。
- **破解**：提前用官方镜像下载 uv（sha256 校验），解包放到该目录；安装器检测到已存在直接使用（零下载）。

### 第 2 层：PM 工具包（node/python/ffmpeg/chromium/agent-browser/cua-driver 等 9 个，约 560MB）

- 下载器逻辑：`pm/store.py::fetch_many` → 缓存目录 `tools/fetch-<sha256>/`；
  **目录里已有文件且 sha256 匹配 → 直接跳过下载**（先 rehash 再决定）。
- 主下载 URL 是 GitHub/nodejs.org 等慢源；官方镜像只是 fallback（限速不触发）。
- **破解**：把 9 个文件按 sha256 从官方镜像预置进 `tools/fetch-<sha256>/`，安装时全部命中缓存、零下载。

### 第 3 层：Python 依赖 venv（`uv sync --frozen / --locked`）

- frozen/locked 模式下 uv **直接按 `uv.lock` 里记录的 URL 下载**（官方锁里全部是 `files.pythonhosted.org`）。
- 即使设了 `UV_INDEX_URL`，frozen 模式也不会改用镜像索引——它认锁文件里的 URL。
- **破解（关键）**：把 `uv.lock` 与 `pm/uv.lock` 中所有 `https://files.pythonhosted.org`
  批量替换为 `https://mirrors.aliyun.com/pypi`（阿里云同构托管同一批文件）。
  - **哈希不变 → uv 照常做 sha256 校验，安全性无损**；
  - **不要动 `registry = "https://pypi.org/simple"` 行**（动了会让 `--locked` 报 "lockfile needs to be updated"）。

### 附：Python 解释器

- 引导阶段 `uv python install` 从 python-build-standalone 的 GitHub Releases 下载（慢源）。
- uv 支持镜像参数 `UV_PYTHON_INSTALL_MIRROR`；用南大镜像预装后，安装器 `uv python find` 直接命中，不再回源。

### 三条规则总结（背下来）

| # | 规则 | 一句话 |
|---|---|---|
| 1 | **预置即跳过** | PM 对 `tools/fetch-<sha256>/` 先 rehash，命中零下载——提前放好大文件 |
| 2 | **锁文件可换 URL、不可换 registry** | 只替换 `files.pythonhosted.org` → 阿里云前缀；registry 行保持 pypi.org |
| 3 | **限速≠失败，兜底不触发** | 对限速必须主动预置/换 URL，不能等它自己回退 |

### 耗时对比（参考）

| 阶段 | 直连官方源 | 本镜像方案 |
|---|---|---|
| 克隆仓库（blobless） | ~2 分钟 | ~2 分钟 |
| bootstrap uv（21MB） | 卡几十分钟+ | 预置命中，0 秒 |
| 工具包 9 个（~560MB） | 预计数小时 | 预置下载 ~2 分钟，安装时解包即过 |
| Python 依赖 venv（300+ 包） | 预计数小时 | **46 秒** |
| 全流程合计 | 数小时～数十小时 | **约 5～10 分钟** ✅ |

---

## 五、完整安装流程（命令流）

> 下列命令 root 直用；普通用户加 `sudo`（宿主机层面操作）或直接以该用户运行（用户层操作）。`$HOME` 即用户主目录。

```bash
# ========== Step 0：前置 ==========
apt-get update && apt-get install -y build-essential libatomic1 git curl
# 2G swap（内存 ≤ 4G 建议，作为编译/峰值保险丝）
[ -f /swapfile ] || { fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile && echo '/swapfile none swap sw 0 0' >> /etc/fstab; }

# ========== Step 1：下载官方安装脚本 ==========
curl -fsSL https://hermes-agent.nousresearch.com/install.sh -o ~/hermes-install.sh

# ========== Step 2：环境自检 + 克隆仓库 ==========
bash ~/hermes-install.sh --non-interactive --stage prerequisites
bash ~/hermes-install.sh --non-interactive --stage repository
# 看到 "✓ Hermes Agent cloned" 即完成（约 2 分钟）

# ========== Step 3：预置 PM 工具包 + bootstrap uv（脚本见第六节，先保存） ==========
bash ~/stage-pm-artifacts.sh          # ~560MB 从官方镜像下载，全部 sha256 校验

# ========== Step 4：锁文件 URL 重写 + npm/pip 镜像 ==========
bash ~/reapply-cn-mirrors.sh

# ========== Step 5：跑完剩余阶段（venv→python-deps→config→products→complete） ==========
nohup bash ~/run-stages.sh >/dev/null 2>&1 &   # 建议后台跑，日志在 ~/stages.log
tail -f ~/stages.log                            # 观察进度；看到 ALL DONE 即完成

# ========== Step 6：验证 ==========
source ~/.bashrc 2>/dev/null    # 让 PATH 包含 ~/.local/bin
~/.local/bin/hermes --version   # 显示版本 + commit
~/.local/bin/hermes doctor      # 全绿（Python/SSL/依赖/目录结构）
~/.local/bin/hermes pm doctor   # 工具链清单（uv/python/node/ffmpeg/chromium…）
```

### 断点续跑（重要）

官方安装脚本支持**单阶段执行**，任何时候中断都不必从头来：

```bash
bash ~/hermes-install.sh --non-interactive --stage venv
bash ~/hermes-install.sh --non-interactive --stage python-deps
bash ~/hermes-install.sh --non-interactive --stage config
bash ~/hermes-install.sh --non-interactive --stage products
bash ~/hermes-install.sh --non-interactive --stage complete
```

---

## 六、三个基础脚本（自包含，复制保存到 ~/ 即可）

### 6.1 `~/stage-pm-artifacts.sh` — 预置工具包缓存 + bootstrap uv

```bash
#!/bin/bash
# Hermes Agent 国内加速：从官方镜像预置 PM 工具包缓存 + bootstrap uv
# 原理：PM 对 tools/fetch-<sha256>/ 缓存先 rehash，命中即跳过慢速源。
# 用法：克隆完成后运行：bash ~/stage-pm-artifacts.sh
set -u
HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"
REPO="$HERMES_HOME/hermes-agent"
STORE="$HERMES_HOME/tools"
MIRROR="https://hermes-assets.nousresearch.com/upstream/sha256"
mkdir -p "$STORE"

# 目标平台（本脚本支持 linux glibc x86_64 / arm64）
case "$(uname -m)" in
  x86_64|amd64) TARGET="linux-x64" ;;
  aarch64|arm64) TARGET="linux-arm64" ;;
  *) echo "不支持的架构: $(uname -m)（musl 系统请手动调整 target）"; exit 1 ;;
esac

# —— 1. 从 pm/lock.json 生成需要预置的 artifacts 清单（版本动态读取，不写死） ——
python3 - "$REPO" "$TARGET" <<'PYEOF' > /tmp/pm-artifacts.tsv
import json, sys
repo, target = sys.argv[1], sys.argv[2]
# 默认工具闭包：source_install_packages(必装) + default_packages(可选默认)
names = ["uv", "python", "node", "npm", "ripgrep", "ffmpeg",
         "agent-browser", "chromium", "cua-driver"]
d = json.load(open(f"{repo}/pm/lock.json"))
rows = set()
for name in names:
    p = d["packages"].get(name)
    if not p:
        continue
    arts = p.get("artifacts", {})
    pick = arts.get(target) or arts.get("any")
    items = pick if isinstance(pick, list) else ([pick] if pick else [])
    for a in items:
        if a and a.get("sha256") and a.get("url"):
            rows.add((name, a["sha256"], a["url"], p.get("version", "")))
for name, sha, url, ver in sorted(rows):
    print(f"{name}\t{sha}\t{url}\t{ver}")
PYEOF

# —— 2. 下载到 fetch-<sha> 缓存（官方镜像优先，gh-proxy 兜底，全部哈希校验） ——
while IFS=$'\t' read -r name sha url ver; do
  [ -z "$sha" ] && continue
  dir="$STORE/fetch-$sha"; base=$(basename "$url")
  if [ -f "$dir/$base" ] && [ "$(sha256sum "$dir/$base" | cut -d' ' -f1)" = "$sha" ]; then
    echo "OK      $name $ver"; continue
  fi
  rm -rf "$dir"; mkdir -p "$dir"
  ok=""
  if curl -sL --retry 3 --retry-delay 2 -o "$dir/$base" "$MIRROR/$sha" \
     && [ "$(sha256sum "$dir/$base" | cut -d' ' -f1)" = "$sha" ]; then ok=1; fi
  if [ -z "$ok" ] && curl -sL --retry 2 -o "$dir/$base" "https://gh-proxy.com/$url" \
     && [ "$(sha256sum "$dir/$base" | cut -d' ' -f1)" = "$sha" ]; then ok=1; fi
  if [ -n "$ok" ]; then
    echo "STAGED  $name $ver ($(du -h "$dir/$base" | cut -f1))"
  else
    echo "FAILED  $name $ver ($url)"; rm -rf "$dir"
  fi
done < /tmp/pm-artifacts.tsv

# —— 3. 解包 bootstrap uv 到 tools/uv-<版本>-<target>/（安装器直接使用，跳过 GitHub 下载） ——
UVSHA=$(awk -F'\t' '$1=="uv"{print $2}' /tmp/pm-artifacts.tsv)
UVVER=$(awk -F'\t' '$1=="uv"{print $4}' /tmp/pm-artifacts.tsv)
if [ -n "$UVSHA" ] && [ ! -x "$STORE/uv-$UVVER-$TARGET/uv" ]; then
  tmp=$(mktemp -d)
  f=$(ls "$STORE/fetch-$UVSHA"/*.tar.gz 2>/dev/null | head -1)
  [ -n "$f" ] && tar -xzf "$f" -C "$tmp" 2>/dev/null
  found=$(find "$tmp" -name uv -type f | head -1)
  if [ -n "$found" ]; then
    mkdir -p "$STORE/uv-$UVVER-$TARGET"
    mv "$found" "$STORE/uv-$UVVER-$TARGET/uv"
    fx=$(find "$tmp" -name uvx -type f | head -1)
    [ -n "$fx" ] && mv "$fx" "$STORE/uv-$UVVER-$TARGET/uvx"
    chmod +x "$STORE/uv-$UVVER-$TARGET/uv" "$STORE/uv-$UVVER-$TARGET/uvx" 2>/dev/null || true
    echo "STAGED  bootstrap uv $UVVER -> $STORE/uv-$UVVER-$TARGET/"
  fi
  rm -rf "$tmp"
fi

# —— 4. 用南大镜像预装 Python（若尚未安装） ——
pypin=$(python3 -c "import json;print(json.load(open('$REPO/pm/lock.json'))['packages']['python']['version'].split('+')[0].rsplit('.',1)[0])" 2>/dev/null || echo "3.14")
UVPY="$STORE/uv-$UVVER-$TARGET/uv"
if [ -x "$UVPY" ] && ! "$UVPY" python find --managed-python "$pypin" >/dev/null 2>&1; then
  UV_PYTHON_INSTALL_MIRROR="https://mirror.nju.edu.cn/github-release/astral-sh/python-build-standalone" \
    "$UVPY" python install --no-bin --no-registry "$pypin" \
    && echo "STAGED  Python $pypin（南大镜像）"
fi
echo "== 预置完成 =="
```

### 6.2 `~/reapply-cn-mirrors.sh` — 锁文件重写 + npm/pip 镜像

```bash
#!/bin/bash
# 重写锁文件下载地址为国内镜像 + 配置 npm/pip 镜像
# 用法（预置后、跑 venv 阶段前）：bash ~/reapply-cn-mirrors.sh
set -u
HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"
REPO="$HERMES_HOME/hermes-agent"
cd "$REPO" || { echo "找不到 $REPO"; exit 1; }

changed=0
for f in uv.lock pm/uv.lock; do
  [ -f "$f" ] || continue
  if grep -q 'files.pythonhosted.org' "$f"; then
    [ -f "$f.orig" ] || cp "$f" "$f.orig"
    # 只替换文件下载前缀；registry 行保持 pypi.org 不动（动了会破坏 --locked 校验）
    sed -i 's|https://files.pythonhosted.org|https://mirrors.aliyun.com/pypi|g' "$f"
    echo "已重写 $f（原版备份 $f.orig）"
    changed=1
  else
    echo "已是镜像版 $f"
  fi
done

# npm 镜像（npmmirror）
echo 'registry=https://registry.npmmirror.com' > "$HOME/.npmrc"
# pip 镜像（PM 会把 pip.conf 的 index-url 桥接给 uv）
mkdir -p "$HOME/.pip"
if ! grep -q 'mirrors.aliyun.com/pypi/simple' "$HOME/.pip/pip.conf" 2>/dev/null; then
  [ -f "$HOME/.pip/pip.conf" ] && [ ! -f "$HOME/.pip/pip.conf.orig" ] && cp "$HOME/.pip/pip.conf" "$HOME/.pip/pip.conf.orig"
  printf '[global]\nindex-url = https://mirrors.aliyun.com/pypi/simple/\n' > "$HOME/.pip/pip.conf"
fi
echo "npm/PyPI 镜像就绪"
[ "$changed" = 1 ] && echo "⚠ 提醒：以后 hermes update 之后需重跑本脚本（新版会 stash/覆盖锁文件修改）"
```

### 6.3 `~/run-stages.sh` — 顺序跑完剩余阶段

```bash
#!/bin/bash
# 顺序执行官方安装脚本的剩余阶段（预置 + 锁重写完成后使用）
# 用法: bash ~/run-stages.sh
set -u
INSTALL_SH="${1:-$HOME/hermes-install.sh}"
LOG="$HOME/stages.log"
: > "$LOG"
for s in venv python-deps config products complete; do
  echo "=== STAGE $s START $(date -u +%H:%M:%S) ===" >> "$LOG"
  bash "$INSTALL_SH" --non-interactive --stage "$s" >> "$LOG" 2>&1
  rc=$?
  echo "=== STAGE $s EXIT $rc $(date -u +%H:%M:%S) ===" >> "$LOG"
  if [ $rc -ne 0 ]; then
    echo "=== ABORTED at $s ===" >> "$LOG"
    exit $rc
  fi
done
echo "=== ALL STAGES DONE $(date -u +%H:%M:%S) ===" >> "$LOG"
```

---

## 七、安装之后（模型与网关，简要）

### 7.1 配置模型（两种方式）

```bash
# 方式 A：交互式向导（有终端时最省事）
hermes model            # 选择 provider / 模型 / 填 API key

# 方式 B：非交互 + 自定义 OpenAI 兼容端点（脚本/远程场景）
#   以命名 provider 为例（示例名 my-endpoint，可任意取；命令用 --force 写自定义键）
hermes config set --force providers.my-endpoint.base_url https://your-endpoint/v1
hermes config set --force providers.my-endpoint.key_env MY_PROVIDER_KEY
hermes config set --force providers.my-endpoint.api_mode chat_completions
hermes config set model.provider my-endpoint
hermes config set model.default your-model-name
echo 'MY_PROVIDER_KEY=sk-xxxx' >> ~/.hermes/.env     # 密钥放 .env（变量名与 key_env 一致）
# 验证
hermes status                  # 应显示你的模型与 provider
hermes -z "你好"               # 一次性调用，直接输出回复即通
```

### 7.2 接入消息平台（网关）

```bash
hermes gateway setup           # 交互式选择平台并按提示配置（Telegram/微信/飞书/Discord…）
hermes gateway install         # 安装为后台服务（Linux 用户级 systemd）
# root 场景：gateway install 默认拒绝以 root 安装系统服务，需显式确认：
sudo hermes gateway install --system --run-as-user root
hermes gateway start --system  # 启动 + 开机自启
hermes gateway status --system # 检查
```

### 7.3 常用聊天内命令（记给用户）

`/new` 开新会话 · `/compress` 手动压缩上下文 · `/model` 切换模型 · `/stop` 中断任务 · `/help` 全部命令

---

## 八、升级与维护（重要固定动作）

1. **每次 `hermes update` 之后**：安装器/更新器会把 `uv.lock`、`pm/uv.lock` 的本地修改
   `git stash`（防止冲突），镜像重写会丢失 → **重跑**：
   ```bash
   bash ~/reapply-cn-mirrors.sh
   ```
2. **版本升级后**（PM 工具版本号会变）：重新预置新版本工具包缓存：
   ```bash
   bash ~/stage-pm-artifacts.sh
   ```
3. **了解即可**：PM 在工具"发布"后会主动清理 `fetch-<sha256>` 缓存（正常行为）；下次重装/升级前重跑预置脚本即可。
4. 查看当前版本：`hermes --version`；查看更新通道：`hermes update --check`（只检查不安装）。

---

## 九、故障排查 / 反面清单

### 9.1 常见现象 → 对号入座

| 现象 | 根因 | 解法 |
|---|---|---|
| 卡在 `Downloading uv` 几十分钟 | GitHub 限速，fallback 不触发 | 预置 bootstrap uv（6.1 脚本第 3 步） |
| 工具包下载龟速 | fetch 缓存未预置 | 跑 6.1 脚本 |
| `Installing Python dependencies…` 极慢 | 锁文件 URL 未重写（pythonhosted） | 跑 6.2 脚本 |
| `uv sync --locked` 报 `lockfile needs to be updated` | registry 行被误改，或镜像与锁不一致 | 恢复 registry 行为 `https://pypi.org/simple`；只替换文件 URL |
| `hermes: command not found` | PATH 未刷新 | `source ~/.bashrc` 或用全路径 `~/.local/bin/hermes` |
| 安装器拒绝以 root 安装系统服务 | 默认安全策略 | 加 `--run-as-user root` 显式确认 |
| 微信/Telegram 平台配好了但 gateway 起不来 | 缺依赖或 token 占位符 | 看 `~/.hermes/logs/gateway.log` 具体报错 |

### 9.2 操作纪律（踩过的坑）

| 坑 | 正确做法 |
|---|---|
| 用 `pkill -f` 杀安装进程 | 可能误伤同会话 SSH 把自己踢下线——按精确 PID kill |
| 使用 v0.10 时代社区脚本（`uv pip install -e .[all]`） | 会脱离 PM 架构、`hermes update` 失效；只借鉴镜像思路，走官方流程 |
| 等慢速下载"自己完成" | 限速不会触发 fallback，主动预置/换 URL 才是正解 |
| 双击跑完全部阶段 | 先预置再阶段化执行，每步可验证、可续跑 |

### 9.3 关键路径速查

| 路径 | 说明 |
|---|---|
| `~/.hermes/hermes-agent/` | 源码仓库（git 安装形态） |
| `~/.hermes/hermes-agent/uv.lock`、`pm/uv.lock` | 依赖锁（已重写为镜像 URL，原版备份 `*.orig`） |
| `~/.hermes/tools/` | PM 字节仓库（已装工具 + `fetch-<sha256>/` 缓存 + `uv-<版本>-<target>/`） |
| `~/.hermes/installs/<install_key>/environments/` | venv 生成代（每次同步/修复产生新代） |
| `~/.local/share/uv/python/` | uv 托管的 Python 解释器 |
| `~/.local/bin/hermes` | hermes 启动器 |
| `~/.hermes/config.yaml`、`~/.hermes/.env` | 主配置与凭证 |
| `~/.hermes/logs/` | agent / gateway / install 日志 |

---

## 十、其他平台说明

| 平台 | 安装方式 | 与本文差异 |
|---|---|---|
| Linux glibc（本文场景） | `install.sh` | — |
| macOS（Apple Silicon） | `install.sh` | 同为 POSIX 脚本；本文学到的镜像/锁重写思路同样适用 |
| Windows | `install.ps1`（PowerShell） | 逻辑相同；预置思路不变（fetch-<sha256> 缓存、锁重写） |
| Android / Termux | 仅 APT 包（`pkg install hermes-agent`） | **不要用 install.sh**（安卓无 glibc 构建） |
| Docker | 官方镜像 | 依赖已内置，无需本方案 |

---

*文档版本：通用版 v1.0 · 基于 Hermes Agent v0.22.x（PM 架构）实测 · 适用于国内网络环境从零安装场景*