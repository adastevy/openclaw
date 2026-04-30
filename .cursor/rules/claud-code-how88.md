# Claude Code + how88 代理配置指南（可迁移）

> 目标：在一台全新 Linux 机器上，快速完成 `Claude Code` 安装，并通过 `how88` 代理稳定调用模型（`claude-opus-4-7`）。

---

## 1. 前置条件

- 系统：Linux（本文按 Ubuntu）
- 已安装 Node.js（建议 `v22+`）和 npm
- 拥有可用代理凭据：
  - `OPENROUTER_API_KEY`（示例 key 是 `sk-...`）
  - 代理地址：`https://how88.top`

快速检查：

```bash
node -v
npm -v
```

---

## 2. 安装 Claude Code（用户级，无 sudo）

> 避免全局目录权限问题（`/usr/lib/node_modules` EACCES）。

```bash
mkdir -p ~/.local ~/.local/bin ~/.npm-global
npm config set prefix ~/.npm-global
npm install -g @anthropic-ai/claude-code
ln -sf ~/.npm-global/bin/claude ~/.local/bin/claude
```

把路径加入 `~/.bashrc`：

```bash
grep -q 'export PATH="$HOME/.local/bin:$HOME/.npm-global/bin:$PATH"' ~/.bashrc \
  || echo 'export PATH="$HOME/.local/bin:$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
```

让当前 shell 生效：

```bash
source ~/.bashrc
claude --version
```

---

## 3. 配置 Claude Code 使用 how88 代理

创建 `~/.claude/env`：

```bash
mkdir -p ~/.claude
cat > ~/.claude/env <<'EOF'
# Claude Code via how88 proxy (Anthropic-compatible)
ANTHROPIC_API_KEY=替换成你的key
ANTHROPIC_BASE_URL=https://how88.top

# 可选：本地网络代理（按需启用）
# HTTPS_PROXY=http://127.0.0.1:7890
# HTTP_PROXY=http://127.0.0.1:7890
# ALL_PROXY=socks5://127.0.0.1:7890
EOF
chmod 600 ~/.claude/env
```

把 env 自动加载加入 `~/.bashrc`：

```bash
grep -q 'source "$HOME/.claude/env"' ~/.bashrc || cat >> ~/.bashrc <<'EOF'

# Claude Code env
if [ -f "$HOME/.claude/env" ]; then
  set -a
  source "$HOME/.claude/env"
  set +a
fi
EOF
```

重新打开终端（或执行 `source ~/.bashrc`）。

---

## 4. 验证是否配置成功

### 4.1 鉴权状态

```bash
claude auth status
```

预期关键字段：

- `"loggedIn": true`
- `"authMethod": "api_key"`
- `"apiKeySource": "ANTHROPIC_API_KEY"`

### 4.2 实际调用测试

```bash
claude -p "Reply with exactly: pong" --output-format text --model claude-opus-4-7

claude --model claude-opus-4-7
```

预期输出：

```text
pong
```

---

## 5. 常见坑（这次排错结论）

## 5.1 Claude Code 的 base URL 不要写 `/v1`

对于 Claude Code（Anthropic 协议）请使用：

- ✅ `ANTHROPIC_BASE_URL=https://how88.top`
- ❌ `ANTHROPIC_BASE_URL=https://how88.top/v1`

原因：Claude Code 走的是 Anthropic Messages 协议路径（`/v1/messages`）。

## 5.2 Web 应用走 OpenAI 兼容时，反而要带 `/v1`

如果你在项目代码里用 OpenAI SDK + chat/completions（如 Next.js 项目）：

- ✅ `OPENROUTER_BASE_URL=https://how88.top/v1`
- ❌ `OPENROUTER_BASE_URL=https://how88.top`

原因：不带 `/v1` 会命中网页 HTML，导致解析失败（常见报错：读取 `choices[0]` 报错）。

## 5.3 模型不可用报错

如果出现：

```text
There's an issue with the selected model ...
```

通常是：

- 当前 key 对该模型无权限，或
- 代理路由对该模型不可用

可先手动测接口：

```bash
curl -sS -D - https://how88.top/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-7","max_tokens":16,"messages":[{"role":"user","content":"reply pong"}]}'
```

---

## 6. 一键复用（最短流程）

```bash
# 1) 安装
mkdir -p ~/.local ~/.local/bin ~/.npm-global ~/.claude
npm config set prefix ~/.npm-global
npm i -g @anthropic-ai/claude-code
ln -sf ~/.npm-global/bin/claude ~/.local/bin/claude

# 2) 环境
cat > ~/.claude/env <<'EOF'
ANTHROPIC_API_KEY=替换成你的key
ANTHROPIC_BASE_URL=https://how88.top
EOF
chmod 600 ~/.claude/env

# 3) PATH + 自动加载
grep -q 'export PATH="$HOME/.local/bin:$HOME/.npm-global/bin:$PATH"' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
grep -q 'source "$HOME/.claude/env"' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
if [ -f "$HOME/.claude/env" ]; then
  set -a
  source "$HOME/.claude/env"
  set +a
fi
EOF

source ~/.bashrc
claude auth status
claude -p "Reply with exactly: pong" --model claude-opus-4-7
```

---

## 7. 建议

- 建议把本文件随项目一起保留，作为跨机器标准化配置 SOP。
- 若切换代理服务商，优先先验证 `/v1/messages` 是否可用，再接入 Claude Code。
