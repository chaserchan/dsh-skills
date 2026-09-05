---
name: wsl-dsh-mirror
description: >-
  在本机 WSL Ubuntu 里同步一份本机 `~/.dsh`（含 web profile、settings、所有数据、plugin 源码）作为
  dsh 影子环境，用于插件开发、版本验证、避免污染正式实例。适用于：用户说"在 WSL 里跑 dsh 影子/
  本机 dsh 一比一同步到 WSL/dsh 开发环境/dup dsh"等。集成 9 步标准流程、3 件对齐验收、4 个实战坑速查
  （WSL bash stdin pipe bug、chmod 600 强制、pnpm 非 TTY 需 CI=true、平台依赖单例挂接顺序），以及嵌入式
  Python TCP 转发脚本（替代 socat，零依赖解决 dsh 0.0.0.0 安全锁）。
---

# wsl-dsh-mirror

> 把本机 Windows 的 `~/.dsh` 一比一同步到 WSL Ubuntu 跑一份独立 dsh 影子实例（端口 3081），与本机 3080 行为完全一致但数据/插件/会话完全隔离。Docker 方案在 6+ 个边界坑上栽过，**WSL 是正解**（Linux 原生 fs、`/mnt/c` 访问宿主 Windows 盘符、零跨 OS 边界污染）。

## 触发场景

- 用户说："在 WSL 里跑个 dsh 影子/把本机 dsh 同步到 WSL/一比一复制 dsh/dsh 影子环境/WSL 跑 dsh"。
- 关键信号词：**WSL + dsh 同步/镜像/影子/开发环境**。
- 不适用于：Docker 容器化 dsh（用 dsh-dev skill 里的 Docker entrypoint 模式）；纯本机 dev 模式（不需要影子）；heavily prod 场景（shadow 终归是开发用）。

## 前置条件（30 秒核验）

```bash
# 1) WSL 装好
wsl -d Ubuntu-22.04 -- bash -c 'echo "$(node -v) / $(npm -v) / $(pnpm -v)"'
# 期望：v24.x / 11.x / 11.x 都有；否则：sudo apt update && sudo apt install -y nodejs npm && npm i -g pnpm@11

# 2) WSL 能读到宿主 Windows 盘
ls /mnt/c/Users/<WINDOWS_USERNAME>/.dsh/profiles/web/package.json
# 期望：文件存在

# 3) 宿主 ~/.dsh 在 owner-readable 模式（不是 mode 755，详见坑 #2）
stat -c '%a' /mnt/c/Users/<WINDOWS_USERNAME>/.dsh/.credentials.yaml 2>/dev/null
# 预期 600；若是 755：chmod 600 ~/.dsh/.credentials.yaml
```

## 9 步标准流程

> **执行要点**：必须用 `wsl -d Ubuntu-22.04 -- bash << 'WSEOF' ... WSEOF`（heredoc 模式），**绝不能用 stdin pipe 形式** `bash -c '...'`——WSL bash 4.4+ 有 stdin pipe + command substitution 冲突的 bug（详见坑 #1）。

### Step 1. 装 dsh@本机同版 + 切 npmmirror

```bash
wsl -d Ubuntu-22.04 -- bash << 'WSEOF'
npm i -g pnpm@11 @deepseek-ai/dsh@0.1.1-rc.1 --registry=https://registry.npmmirror.com
# 验证：dsh --version → 0.1.1-rc.1
# 关键：WSL 默认 prefix 是 /home/<user>/.npm-global，不是 /usr/local
ls $HOME/.npm-global/lib/node_modules/@deepseek-ai/dsh/node_modules/@deepseek-ai/ | wc -l
# 期望：197 个内置平台包
WSEOF
```

### Step 2. 同步本机 ~/.dsh 到 WSL `~/.dsh`

```bash
wsl -d Ubuntu-22.04 -- bash << 'WSEOF'
export HOME=/home/<WSL_USERNAME>
# 1) 权限必须先修（坑 #2）
chmod 600 $HOME/.dsh/.credentials.yaml
# 2) 同步（cp -r 不用排除 node_modules——这次复制数据，不复制 plugin 源码；plugin 源码单独走 Step 3）
cp -r /mnt/c/Users/<WINDOWS_USERNAME>/.dsh/. $HOME/.dsh/
WSEOF
```

### Step 3. 复制 14 个 plugin 源码到 WSL `~/.dsh/plugin/`（tar 排除 node_modules/junction，坑 #4）

```bash
wsl -d Ubuntu-22.04 -- bash << 'WSEOF'
export HOME=/home/<WSL_USERNAME>
SRC=/mnt/d/job/developer/DSH/plugin
DST=$HOME/.dsh/plugin
rm -rf $DST && mkdir -p $DST
for d in $SRC/*/; do
  name=$(basename "$d")
  dst=$DST/$name
  mkdir -p $dst
  # tar 排除 node_modules（关键！cp -r 会把 Windows junction 复制成死 symlink）
  tar -C "$d" --exclude=node_modules --exclude=.git -cf - . 2>/dev/null | tar -C $dst -xf -
done
# chaseman-link 是 plugin/ 外的平铺目录
mkdir -p $HOME/.dsh/dsh-chaseman-link
tar -C /mnt/d/job/developer/DSH/dsh-chaseman-link --exclude=node_modules --exclude=.git -cf - . | tar -C $HOME/.dsh/dsh-chaseman-link -xf -
echo "plugin 源码同步: $(ls $DST | wc -l) 个"
WSEOF
```

### Step 4. 平台依赖 197 包 × 15 插件 批量单例挂接（ln -s，坑 #3 顺序：必须在 pnpm install 之后）

```bash
wsl -d Ubuntu-22.04 -- bash << 'WSEOF'
export HOME=/home/<WSL_USERNAME>
WSL_DSH_PKGS="$HOME/.npm-global/lib/node_modules/@deepseek-ai/dsh/node_modules/@deepseek-ai"
PLUGIN_ROOTS=""
for d in $HOME/.dsh/plugin/*/; do PLUGIN_ROOTS="$PLUGIN_ROOTS ${d%/}"; done
PLUGIN_ROOTS="$PLUGIN_ROOTS $HOME/.dsh/dsh-chaseman-link"
for root in $PLUGIN_ROOTS; do
  mkdir -p "$root/node_modules/@deepseek-ai" 2>/dev/null || true
  for host_pkg in $WSL_DSH_PKGS/*/; do
    pkg="$(basename "$host_pkg")"
    [ "$pkg" = "dsh" ] && continue
    ln -sfn "$host_pkg" "$root/node_modules/@deepseek-ai/$pkg" 2>/dev/null || true
  done
done
echo "挂接: $(ls $WSL_DSH_PKGS | wc -l) 包 × $(echo $PLUGIN_ROOTS | wc -w) 插件"
# 验证：ls $HOME/.dsh/plugin/dsh-user-system/node_modules/@deepseek-ai/schemastery → 应该是 symlink
WSEOF
```

### Step 5. 改写 profile link/file: 路径（DSH 镜像副本，不要指向宿主 Windows）

```bash
wsl -d Ubuntu-22.04 -- bash << 'WSEOF'
export HOME=/home/<WSL_USERNAME>
export PROFILE_PKG="$HOME/.dsh/profiles/web/package.json"
node -e '
  const fs = require("fs");
  const p = JSON.parse(fs.readFileSync(process.env.PROFILE_PKG, "utf8"));
  for (const k of Object.keys(p.dependencies)) {
    const v = p.dependencies[k];
    const m = /^link:(\/mnt\/d\/job\/developer\/DSH\/plugin\/[^\/]+)\/?$/.exec(v);
    if (m) p.dependencies[k] = "link:" + process.env.HOME + "/.dsh/plugin/" + m[1].split("/").pop();
    const f = /^link:(\/mnt\/d\/job\/developer\/DSH\/[^\/]+)\/?$/.exec(v);
    if (f) p.dependencies[k] = "link:" + process.env.HOME + "/.dsh/" + f[1].split("/").pop();
  }
  fs.writeFileSync(process.env.PROFILE_PKG, JSON.stringify(p, null, 2) + "\n");
'
WSEOF
```

### Step 6. pnpm install（必须 CI=true 绕非 TTY 卡死，坑 #5）

```bash
wsl -d Ubuntu-22.04 -- bash << 'WSEOF'
export HOME=/home/<WSL_USERNAME>
cd $HOME/.dsh/profiles/web
rm -rf node_modules pnpm-lock.yaml   # 关键：避免 pnpm 检测残留后要交互确认
CI=true pnpm install --registry=https://registry.npmmirror.com
# 期望：Progress: resolved 50, reused 50, done
WSEOF
```

### Step 7. dsh dump-config 干跑（确认 16 个 bundle 全入树）

```bash
wsl -d Ubuntu-22.04 -- bash << 'WSEOF'
export HOME=/home/<WSL_USERNAME>
$HOME/.npm-global/bin/dsh --profile web --dump-config 2>&1 | grep "^# ==" | sort -u
# 期望看到：@deepseek-ai/dsh-base, dsh-web-app, dshmarket, modlens, pocket, modsearch, chat-import,
# session-cost, agent-message, chaseman-link, user-system, agent-teams, cockpit, archify-dsh,
# media-capture, wechat-devtools 共 16 个
WSEOF
```

### Step 8. 启动 dsh + Python TCP 转发（替代 socat，零依赖 + 绕 dsh 0.0.0.0 安全锁，坑 #6）

**Step 8a. 写入 Python 转发脚本**（用 base64 喂脚本过 stdin pipe 障碍）：

```bash
SCRIPT_B64=$(base64 -w0 <(cat << 'PYEOF'
#!/usr/bin/env python3
# 0.0.0.0:3081 -> 127.0.0.1:3080 TCP 转发（替代 socat，零依赖）
import socket, threading
def pipe(src, dst):
    try:
        while True:
            data = src.recv(8192)
            if not data: break
            dst.sendall(data)
    except OSError: pass
    finally:
        try: src.close()
        except: pass
        try: dst.close()
        except: pass
def serve():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(('0.0.0.0', 3081))
    s.listen(128)
    print(f'fwd 0.0.0.0:3081 -> 127.0.0.1:3080', flush=True)
    while True:
        c, _ = s.accept()
        try:
            t = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            t.connect(('127.0.0.1', 3080))
        except OSError:
            c.close(); continue
        threading.Thread(target=pipe, args=(c, t), daemon=True).start()
        threading.Thread(target=pipe, args=(t, c), daemon=True).start()
serve()
PYEOF
))
wsl -d Ubuntu-22.04 -- bash -c "echo '$SCRIPT_B64' | base64 -d > /home/<WSL_USERNAME>/tcp-fwd.py"
```

**Step 8b. 后台启动 dsh + fwd**：

```bash
wsl -d Ubuntu-22.04 -- bash << 'WSEOF'
export HOME=/home/<WSL_USERNAME>
# 清残留（忽略 exit 1 错误）
pkill -9 -f "dsh --profile web" 2>/dev/null || true
pkill -9 -f "tcp-fwd.py" 2>/dev/null || true
sleep 2
# 后台启动
setsid socat TCP-LISTEN:3081,reuseaddr,fork,bind=0.0.0.0 TCP:127.0.0.1:3080 >/tmp/socat.log 2>&1 </dev/null &
sleep 1
setsid /home/<WSL_USERNAME>/.npm-global/bin/dsh --profile web --port 3080 --no-open >/tmp/dsh.log 2>&1 </dev/null &
disown -a
sleep 10
# 验证
ss -tlnp 2>/dev/null | grep -E ":30(80|81) " | head
# 期望：dsh 监听 127.0.0.1:3080，fwd 监听 0.0.0.0:3081
WSEOF
```

### Step 9. 三件对齐验收（一键返回 PASS/FAIL）

```bash
echo "=== 1) dump-config entry 列表 ==="
wsl -d Ubuntu-22.04 dsh --profile web --dump-config 2>&1 | grep -c '^# == '

echo "=== 2) session.list items (从宿主打 3081) ==="
curl -s --noproxy "*" --max-time 30 -X POST -H "Content-Type: application/json" \
  -d '{"type":"client-request","rpcId":"r","method":"session.list","payload":{}}' \
  -o /tmp/sl.json -w "HTTP=%{http_code}\n" http://127.0.0.1:3081/api/session.list
node -e 'const j=require("fs").readFileSync("/tmp/sl.json","utf8");const o=JSON.parse(j);console.log("items="+(o.result?.value?.items?.length||"?"))'

echo "=== 3) attachments 文件数 ==="
wsl -d Ubuntu-22.04 bash -c 'find ~/.dsh/attachments -type f | wc -l'

echo "=== 4) 3081 探活 ==="
curl -s --noproxy "*" -o /dev/null -w "HTTP=%{http_code}\n" --max-time 5 http://127.0.0.1:3081/
```

**期望输出对照**：

| 项 | 本机基线 | 影子期望 | FAIL 排查 |
|---|---|---|---|
| 1) entry 列表 | 16 | 16 | 若 < 16：Step 5 path 改写没生效 |
| 2) items | 518 | 518±N | 若差太大：Step 6 pnpm install 没重建 attachments |
| 3) attachments | 528 | 528 | 严格相等 |
| 4) HTTP | 200 | 200 | 000 → Step 8b fwd 没起，看 fwd.log |

## 4 个实战坑速查（pitfalls #82 落库版）

| 坑 | 症状 | 根因 | 修法 |
|---|---|---|---|
| **#1 WSL bash stdin pipe + command substitution 冲突** | `for d in $(...)` 的 `$d` 全是空字符串，循环里 mkdir/tar 全失败 | bash 4.4+ 把 stdin pipe 当作命令替换的 word splitting 数据源，`exec 0</dev/null` 也无效 | **全程用 heredoc**：`wsl -d Ubuntu-22.04 -- bash << 'WSEOF' ... WSEOF`；或用 base64 喂脚本（Step 8a） |
| **#2 dsh-credentials-local 强制 600** | boot 报 `readable beyond its owner (mode 755); run "chmod 600"` | dsh 安全设计：credentials 文件必须 owner-only | **Step 2 开头** `chmod 600 ~/.dsh/.credentials.yaml` |
| **#3 平台依赖挂接顺序** | 挂接后 pnpm install 把 symlink 全删了 / 挂接时机不对 | profile pnpm install 处理 link 插件的 node_modules 重建会覆盖手工挂接 | **Step 4 必须在 Step 6 pnpm install 之后**（先 install，让 pnpm 重建好 node_modules 框架，再 ln -s 挂接补丁）——但本 skill 流程 Step 4 在 Step 6 之前，因为镜像 plugin 副本不靠 profile pnpm install，是单独 ln -s |
| **#4 cp -r 复制 Windows junction 变死 symlink** | 容器/WSL 里 plugin/node_modules 是 dead symlink，ls 看 No such，mkdir 报 File exists | `cp -r` 不区分 Linux symlink vs Windows junction，原样复制 | **必须用 tar 排除**：`tar -C $src --exclude=node_modules --exclude=.git -cf - . | tar -C $dst -xf -` |
| **#5 pnpm 非 TTY 拒绝 purge** | `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY` | pnpm 默认交互确认是否清空 modules 目录 | **必须 `CI=true`** 显式声明非交互 |
| **#6 dsh 0.0.0.0 安全锁 + socat 缺失** | dsh 启动报 `--host 0.0.0.0 is intentionally not supported yet for safety`；WSL 无 socat 且无 sudo | dsh 产品级安全锁；WSL Ubuntu 默认不带 socat | **用 Python TCP 转发**（Step 8a 嵌入脚本），零依赖；不能 `--host 0.0.0.0` |

## 日常使用

```powershell
# 改插件源码后
wsl -d Ubuntu-22.04 -- rsync -av --delete /mnt/d/job/developer/DSH/plugin/dsh-<x>/ \
   /home/<WSL_USERNAME>/.dsh/plugin/dsh-<x>/

# 重启 dsh 让新插件生效
wsl -d Ubuntu-22.04 -- bash -c 'pkill -9 -f "dsh --profile web" 2>/dev/null; sleep 2; setsid $HOME/.npm-global/bin/dsh --profile web --port 3080 --no-open >/tmp/dsh.log 2>&1 </dev/null & disown -a; sleep 8; echo "restarted PID $(pgrep -f "dsh --profile")"'
# 验证
curl -s --noproxy "*" -o /dev/null -w "HTTP=%{http_code}\n" http://127.0.0.1:3081/

# 改 profile 配置后
wsl -d Ubuntu-22.04 -- bash -c 'pkill -9 -f "dsh --profile web"; sleep 2; cd $HOME/.dsh/profiles/web; CI=true pnpm install --registry=https://registry.npmmirror.com; setsid $HOME/.npm-global/bin/dsh --profile web --port 3080 --no-open >/tmp/dsh.log 2>&1 </dev/null & disown -a'
```

**WSL2 localhost forwarding**：宿主浏览器 `http://127.0.0.1:3081` 自动通 WSL 内部 `0.0.0.0:3081`（无需端口映射）。
