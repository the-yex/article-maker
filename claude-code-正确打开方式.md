# 程序员使用 Claude Code 的正确打开方式

今天和大家分享一下我日常 Claude Code 配置多模型深度使用的方案，感觉挺实用的。

市面上有各种可视化配置工具，有些做得挺花哨，但我用不来——总感觉多了层中间件，反而不如命令行来得直接。程序员嘛，简单粗暴的终端操作才是真爱。

## 每次换模型都要改配置，这谁受得了

官方用法是这样：要换模型就去 `~/.config/claude/claude_desktop_config.json`，找到 `anthropic_model` 字段，改成你要的模型 ID，保存，重启终端。

一次还好，但开发过程中想法多变。写复杂架构想用 Opus，写业务逻辑想用国内便宜模型，写代码调试想用火山买的coding plan。一天下来，配置文件改了七八次。

更烦的是每次还要记住不同模型的 ID，手滑输错一个字符就得重来。

关键问题是：Claude Code 支持环境变量覆盖配置，但没人教你用。

## 三行代码解决所有问题

核心思路很简单：用 Shell 函数包裹环境变量 + Claude Code 命令，每次调用临时设置三个环境变量：

- `ANTHROPIC_AUTH_TOKEN`：API Key
- `ANTHROPIC_BASE_URL`：接口地址
- `ANTHROPIC_MODEL`：模型名称

环境变量只在该次调用生效，互不干扰。

先建个 `~/.ai_env` 存放所有 Key：

```bash
# ~/.ai_env
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC="1"
export VOLCES_KEY="你的火山key"
export CLAUDE_KEY="你的claude key"
```

再建 `~/.ai_model.sh` 定义命令：

```bash
source ~/.ai_env

# 豆包编程模型
doubao() {
  ANTHROPIC_AUTH_TOKEN="$VOLCES_KEY" \
  ANTHROPIC_BASE_URL="https://ark.cn-beijing.volces.com/api/compatible" \
  ANTHROPIC_MODEL="doubao-seed-code-preview-251028" \
  claude "$@"
}

# 能力超强的claude opus 模型，做架构设计或者解决一些无法解决的问题
opus() {
  ANTHROPIC_AUTH_TOKEN="$CLAUDE_KEY" \
  ANTHROPIC_BASE_URL="https://claude-code.club/api" \
  ANTHROPIC_MODEL="claude-opus" \
  claude "$@"
}

# 大量任务使用的plan 套餐包
plan() {
  ANTHROPIC_AUTH_TOKEN="$VOLCES_KEY" \
  ANTHROPIC_BASE_URL="https://ark.cn-beijing.volces.com/api/coding" \
  ANTHROPIC_MODEL="ark-code-latest" \
  claude "$@"
}
# ..... 后面要加新的模型比如GLM5 GLM4  都直接加，很简单方便
```

然后在 `~/.zshrc` 或 `~/.bashrc` 加一句：

```bash
source ~/.ai_model.sh
```

最后执行

``````shell
source ~/.zshrc
``````

搞定。

## 一天下来，这样用

早上来公司遇到个诡异 bug，`plan 分析这个错误日志`，Doubao-Coding 上手就是排错。

午饭前想写个复杂的服务架构设计，`opus 帮我设计一个高可用的订单服务`，Opus 考虑得周全。

下午要写段业务逻辑，`glm5 写个用户注册接口`，GLM5 又快又准。

每个模型各司其职，不用再记配置文件路径，不用再重启终端。

---

## Windows 用户看这里

如果你用的是 Windows，PowerShell 也能搞定，思路完全一样。

在 `$PROFILE` 文件里添加这些函数（通常在 `Documents\PowerShell\Microsoft.PowerShell_profile.ps1`）：

```powershell
# 设置环境变量
$env:VOLCES_KEY = "你的火山key"
$env:CLAUDE_KEY = "你的claude key"

# 豆包编程模型
function doubao {
  param([string]$prompt)
  $env:ANTHROPIC_AUTH_TOKEN = $env:VOLCES_KEY
  $env:ANTHROPIC_BASE_URL = "https://ark.cn-beijing.volces.com/api/compatible"
  $env:ANTHROPIC_MODEL = "doubao-seed-code-preview-preview-251028"
  claude $prompt
}

# Claude Opus 模型，做架构设计或解决复杂问题
function opus {
  param([string]$prompt)
  $env:ANTHROPIC_AUTH_TOKEN = $env:CLAUDE_KEY
  $env:ANTHROPIC_BASE_URL = "https://claude-code.club/api"
  $env:ANTHROPIC_MODEL = "claude-opus"
  claude $prompt
}

# Plan 套餐包，大量任务专用
function plan {
  param([string]$prompt)
  $env:ANTHROPIC_AUTH_TOKEN = $env:VOLCES_KEY
  $env:ANTHROPIC_BASE_URL = "https://ark.cn-beijing.volces.com/api/coding"
  $env:ANTHROPIC_MODEL = "ark-code-latest"
  claude $prompt
}
# 后面要加新的模型比如 GLM5、GLM4，直接按这个格式加即可
```

PowerShell 的语法稍微啰嗦点，但核心逻辑一模一样：设置环境变量 → 调用 `claude` 命令。

## 避坑指南

1. **API Key 别直接写死在函数里**，统一放 `~/.ai_env`，管理起来方便，也不怕误提交到 git

2. **环境变量用反斜杠换行**，看着舒服不影响执行，单行写太长容易错

3. **`$@` 别漏了**，这是把所有参数传给 Claude Code 命令，漏了就传不进去你的提问

4. **确认配置生效**，新开终端执行 `echo $DOUBAO_KEY`，如果输出你的 Key 说明没问题

## 最后一句

工具的价值不在于功能多，而在于顺手。改配置文件用起来当然没问题，但每天重复十次，就是浪费生命。

把繁琐交给 Shell 函数，把时间留给思考。

---

说个真话，这套方案我用了几个月，确实省心不少。但我也清楚，肯定还有我没玩明白的骚操作。

你有没有更好的 Claude Code 使用方案？评论区分享出来，咱们互相抄一下作业。