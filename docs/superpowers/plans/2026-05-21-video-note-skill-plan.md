# Video Note Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建技能，支持通过 Telegram 将 B 站视频链接转换为带完整转写、总结的 Obsidian 笔记

**Architecture:** 技能基于 OpenClaw/Hermes 平台，使用 yt-dlp 下载音频，支持 Whisper/飞书妙记/MiniMax 三种转写引擎，按模板生成 Obsidian 文档后通过 obsidian-cli 写入

**Tech Stack:** Shell scripts, yt-dlp, whisper-cpp/lark-cli/MiniMax API, obsidian-cli

---

## 文件结构

```
~/.openclaw/skills/video-note/
├── SKILL.md                    # 技能主入口
├── transcribe/
│   ├── download.sh            # 音频下载
│   └── engines/
│       ├── whisper.sh          # Whisper 本地转写
│       ├── lark.sh             # 飞书妙记
│       └── minimax.sh          # MiniMax API
├── templates/
│   └── video-note.md           # Obsidian 文档模板
└── scripts/
    └── write-to-obsidian.sh    # obsidian-cli 调用

~/.hermes/skills/video-note/    # 软链接到上述目录
```

---

## Task 1: 技能基础结构

**Files:**
- Create: `~/.openclaw/skills/video-note/SKILL.md`
- Create: `~/.openclaw/skills/video-note/skill.yaml`
- Create: `~/.openclaw/skills/video-note/transcribe/download.sh`
- Create: `~/.openclaw/skills/video-note/transcribe/engines/whisper.sh`
- Create: `~/.openclaw/skills/video-note/transcribe/engines/lark.sh`
- Create: `~/.openclaw/skills/video-note/transcribe/engines/minimax.sh`
- Create: `~/.openclaw/skills/video-note/templates/video-note.md`
- Create: `~/.openclaw/skills/video-note/scripts/write-to-obsidian.sh`
- Create: `~/.hermes/skills/video-note/` → `~/.openclaw/skills/video-note/`

- [ ] **Step 1: Create skill directory structure**

```bash
mkdir -p ~/.openclaw/skills/video-note/transcribe/engines
mkdir -p ~/.openclaw/skills/video-note/templates
mkdir -p ~/.openclaw/skills/video-note/scripts
ln -sf ~/.openclaw/skills/video-note ~/.hermes/skills/video-note
```

- [ ] **Step 2: Create skill.yaml**

```yaml
name: video-note
version: 1.0.0
description: "B站视频转笔记：下载视频音频 → 转写 → AI总结 → 写入Obsidian"
default_engine: whisper
obsidian_vault: "chank knowledge"
output_folder: "01 - 知识库/Learning"
supported_engines:
  - whisper
  - lark
  - minimax
```

- [ ] **Step 3: Create SKILL.md**

```markdown
---
name: video-note
version: 1.0.0
description: "B站视频转笔记：下载视频音频 → 转写 → AI总结 → 写入Obsidian"
---

# Video Note (v1)

将 B 站视频链接转换为带完整转写和总结的 Obsidian 笔记。

## 使用方式

发送视频链接即可：
```
/video-note https://b23.tv/xxx
```

指定转写引擎：
```
/video-note https://b23.tv/xxx --engine minimax
/video-note https://b23.tv/xxx --engine lark
```

## 处理流程

1. 解析 B 站链接，提取视频元信息
2. yt-dlp 下载音频
3. 调用转写引擎（默认 Whisper）
4. LLM 生成总结
5. 按模板生成文档
6. obsidian-cli 写入 Obsidian

## 配置

- default_engine: whisper（默认）
- obsidian_vault: chank knowledge
- output_folder: 01 - 知识库/Learning
```

- [ ] **Step 4: Create download.sh**

```bash
#!/bin/bash
# download.sh - 下载 B 站视频音频
# 用法: download.sh <url> <output_dir>

URL="$1"
OUTPUT_DIR="$2"

if [ -z "$URL" ] || [ -z "$OUTPUT_DIR" ]; then
    echo "用法: download.sh <url> <output_dir>"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"

# 下载音频并转换为 mp3
yt-dlp \
    --extract-audio \
    --audio-format mp3 \
    --audio-quality 0 \
    --output "$OUTPUT_DIR/%(id)s.%(ext)s" \
    "$URL"

AUDIO_FILE=$(find "$OUTPUT_DIR" -name "*.mp3" | head -1)

if [ -f "$AUDIO_FILE" ]; then
    echo "$AUDIO_FILE"
    exit 0
else
    echo "下载失败" >&2
    exit 1
fi
```

- [ ] **Step 5: Create whisper.sh**

```bash
#!/bin/bash
# whisper.sh - 本地 Whisper 转写
# 用法: whisper.sh <audio_file>

AUDIO_FILE="$1"

if [ -z "$AUDIO_FILE" ] || [ ! -f "$AUDIO_FILE" ]; then
    echo "用法: whisper.sh <audio_file>" >&2
    exit 1
fi

# 检查 whisper-cli 是否可用
if ! command -v whisper-cli &> /dev/null; then
    echo "whisper-cli 未安装，跳过本地转写" >&2
    exit 1
fi

# 执行转写，输出 JSON
whisper-cli "$AUDIO_FILE" --model ggml-tiny.bin --output-json 2>/dev/null
```

- [ ] **Step 6: Create lark.sh**

```bash
#!/bin/bash
# lark.sh - 飞书妙记转写
# 用法: lark.sh <audio_file>

AUDIO_FILE="$1"

if [ -z "$AUDIO_FILE" ] || [ ! -f "$AUDIO_FILE" ]; then
    echo "用法: lark.sh <audio_file>" >&2
    exit 1
fi

# 上传音频到飞书云空间
FILE_TOKEN=$(lark-cli drive +upload --file "$AUDIO_FILE" 2>/dev/null | jq -r '.file_token')

if [ -z "$FILE_TOKEN" ] || [ "$FILE_TOKEN" = "null" ]; then
    echo "飞书上传失败" >&2
    exit 1
fi

# 生成妙记
MINUTE_URL=$(lark-cli minutes +upload --file-token "$FILE_TOKEN" 2>/dev/null | jq -r '.minute_url')

if [ -z "$MINUTE_URL" ] || [ "$MINUTE_URL" = "null" ]; then
    echo "妙记生成失败" >&2
    exit 1
fi

# 提取 minute_token
MINUTE_TOKEN=$(echo "$MINUTE_URL" | grep -oE '[^/]+$')

# 获取逐字稿
lark-cli vc +notes --minute-tokens "$MINUTE_TOKEN" 2>/dev/null | jq -r '.transcript'
```

- [ ] **Step 7: Create minimax.sh**

```bash
#!/bin/bash
# minimax.sh - MiniMax API 转写
# 用法: minimax.sh <audio_file>

AUDIO_FILE="$1"

if [ -z "$AUDIO_FILE" ] || [ ! -f "$AUDIO_FILE" ]; then
    echo "用法: minimax.sh <audio_file>" >&2
    exit 1
fi

# 检查 MiniMax API 配置
if [ -z "$MINIMAX_API_KEY" ]; then
    echo "MINIMAX_API_KEY 未设置" >&2
    exit 1
fi

# 调用 MiniMax 语音转写 API
curl -X POST "https://api.minimax.chat/v1/speech/transcribe" \
    -H "Authorization: Bearer $MINIMAX_API_KEY" \
    -F "file=@$AUDIO_FILE" \
    -F "model=whisper-1" \
    2>/dev/null | jq -r '.text'
```

- [ ] **Step 8: Create video-note.md template**

```markdown
# <title>

> 来源：B站视频（<bv_id>）
> UP主：<uploader>（<followers>）
> 发布时间：<publish_date>
> 视频长度：<duration>
> 转写时间：<transcribe_date>
> 转写工具：<engine>

---

## 完整文案转录

<transcript>

---

<sections>

---

## 核心要点总结

<summary>

---

## 相关资源

- **视频**：<url>
- **合集**：<playlist>

---

## 标签

<tags>
```

- [ ] **Step 9: Create write-to-obsidian.sh**

```bash
#!/bin/bash
# write-to-obsidian.sh - 将笔记写入 Obsidian
# 用法: write-to-obsidian.sh <title> <content> <vault> <folder>

TITLE="$1"
CONTENT="$2"
VAULT="${3:-chank knowledge}"
FOLDER="${4:-01 - 知识库/Learning}"

# 清理文件名
FILENAME=$(echo "$TITLE" | sed 's/[^a-zA-Z0-9一-龥]/-/g' | cut -c1-50)

# 构建路径
YEAR=$(date "+%Y")
MONTH=$(date "+%m")
VAULT_PATH=$(obsidian-cli print-default)
NOTE_PATH="$YEAR/$MONTH/$FILENAME"

# 使用 obsidian-cli 创建笔记
obsidian-cli create \
    --vault "$VAULT" \
    --content "$CONTENT" \
    --path "$FOLDER/$NOTE_PATH" \
    2>/dev/null

echo "$FOLDER/$NOTE_PATH.md"
```

- [ ] **Step 10: Set executable permissions**

```bash
chmod +x ~/.openclaw/skills/video-note/transcribe/download.sh
chmod +x ~/.openclaw/skills/video-note/transcribe/engines/*.sh
chmod +x ~/.openclaw/skills/video-note/scripts/write-to-obsidian.sh
```

- [ ] **Step 11: Verify directory structure**

```bash
find ~/.openclaw/skills/video-note -type f
find ~/.hermes/skills -name "video-note" -type l
```

---

## Task 2: 集成测试

**Files:**
- Test: `~/.openclaw/skills/video-note/` (manual test)

- [ ] **Step 1: Test download.sh with a test video**

```bash
# 创建测试目录
mkdir -p /tmp/video-note-test

# 测试下载（使用一个短测试视频）
~/.openclaw/skills/video-note/transcribe/download.sh \
    "https://b23.tv/BV1TZ5Y6CEH1" \
    "/tmp/video-note-test"
```

Expected: 在 /tmp/video-note-test 目录下生成 .mp3 文件

- [ ] **Step 2: Verify audio file exists**

```bash
ls -la /tmp/video-note-test/*.mp3
```

Expected: 列出 mp3 文件，大小 > 0

- [ ] **Step 3: Cleanup test files**

```bash
rm -rf /tmp/video-note-test
```

---

## Task 3: 系统提示词优化

**Files:**
- Modify: `~/.openclaw/skills/video-note/SKILL.md`

- [ ] **Step 1: Update SKILL.md with detailed instructions**

```markdown
---
name: video-note
version: 1.0.0
description: "B站视频转笔记：下载视频音频 → 转写 → AI总结 → 写入Obsidian"
---

# Video Note (v1)

将 B 站视频链接转换为带完整转写和总结的 Obsidian 笔记。

## 触发条件

用户发送 B 站视频链接（b23.tv, bilibili.com, www.bilibili.com）

## 处理流程

### 1. 解析链接
使用 yt-dlp --dump-json 提取视频元信息：
- 标题 (title)
- UP主 (uploader)
- 发布时间 (upload_date)
- 时长 (duration)
- BV号 (id)

### 2. 下载音频
调用 transcribe/download.sh

### 3. 转写
默认引擎: Whisper (本地)
备用引擎: --engine lark 或 --engine minimax

### 4. 生成内容
使用 LLM 处理转写文本：
- 生成完整转录
- 识别分段主题
- 提取核心要点

### 5. 写入 Obsidian
调用 scripts/write-to-obsidian.sh
路径: 01 - 知识库/Learning/YYYY/MM/

## 调用示例

```
/video-note https://b23.tv/xxx
/video-note https://b23.tv/xxx --engine minimax
/video-note https://b23.tv/xxx --engine lark
```

## 注意事项

- Whisper 需本地安装 whisper-cli 和 ggml-tiny.bin
- 飞书妙记需要 lark-cli 已登录
- MiniMax 需要 MINIMAX_API_KEY 环境变量
```

- [ ] **Step 2: Verify SKILL.md syntax**

```bash
head -30 ~/.openclaw/skills/video-note/SKILL.md
```

---

## Task 4: 文档

**Files:**
- Create: `~/.openclaw/skills/video-note/README.md`

- [ ] **Step 1: Create README.md**

```markdown
# Video Note Skill

将 B 站视频转换为 Obsidian 笔记。

## 安装

技能已安装在 `~/.openclaw/skills/video-note/`

## 使用

```
/video-note <B站链接>
```

### 指定转写引擎

```
/video-note <链接> --engine whisper  # 本地（默认）
/video-note <链接> --engine lark      # 飞书妙记
/video-note <链接> --engine minimax   # MiniMax API
```

## 前置依赖

- yt-dlp
- whisper-cli + ggml-tiny.bin (可选)
- lark-cli (可选)
- obsidian-cli

## 配置文件

编辑 `skill.yaml` 修改默认引擎和 Obsidian 路径。
```

- [ ] **Step 2: Verify README.md**

```bash
cat ~/.openclaw/skills/video-note/README.md
```

---

## Task 5: 提交

- [ ] **Step 1: Initialize git (if needed) and commit**

```bash
cd /Users/chank/Documents/trae
git init 2>/dev/null || true
git add .
git commit -m "feat: add video-note-skill for B站视频转Obsidian笔记

- 支持 Whisper/飞书妙记/MiniMax 三种转写引擎
- 按模板生成带转写和总结的 Obsidian 文档
- 通过 obsidian-cli 写入 01 - 知识库/Learning 目录"
```

- [ ] **Step 2: Verify git status**

```bash
git status
```

---

## 验证清单

- [ ] 技能目录结构正确
- [ ] 所有脚本有执行权限
- [ ] SKILL.md 包含完整指令
- [ ] 模板文件存在
- [ ] Hermens 软链接有效
- [ ] 下载脚本可正常工作