# 视频笔记技能设计规格

## 概述

用户可以在 Telegram 中将 B 站视频链接发送到 OpenClaw 或 Hermes，AI 自动完成视频音频下载、转写、AI 总结，并按模板写入用户 Obsidian 知识库。

## 技能范围

- **输入**：Telegram 消息中的 B 站视频链接
- **输出**：符合模板格式的 Obsidian Markdown 文件
- **平台支持**：OpenClaw、Hermes

## 流程

```
Telegram 消息（含 B 站视频链接）
    ↓
AI 解析链接，提取视频标题
    ↓
yt-dlp 下载视频音频
    ↓
调用转写引擎（默认或指定）
    ↓
LLM 按模板生成文档
    ↓
obsidian-cli 写入 Obsidian
    ↓
返回写入路径给用户
```

## 转写引擎

支持三种引擎，默认为 Whisper，调用时可临时指定：

| 引擎 | 类型 | 配置方式 |
|------|------|----------|
| Whisper | 本地 | `whisper-cpp + ggml-tiny.bin` |
| 飞书妙记 | 云端 | lark-cli minutes |
| MiniMax | 云端 | MiniMax API |

### 引擎选择

- **默认**：Whisper（本地运行，免费）
- **临时指定**：`--engine minimax` 或 `--engine lark`

## Obsidian 写入

### 路径结构

```
<obsidian-vault>/
└── 01 - 知识库/
    └── Learning/
        └── <年份>/
            └── <月份>/
                └── <视频标题>.md
```

### 文件命名

- 提取视频标题作为文件名
- 特殊字符去除，空格转为连字符
- 长度限制 50 字符

### 模板格式

```markdown
# <视频标题>

> 来源：B站视频（<BV号>）
> UP主：<UP主名>（<粉丝数>）
> 发布时间：<视频日期>
> 视频长度：<时长>
> 转写时间：<执行时间>
> 转写工具：<使用的引擎>

---

## 完整文案转录

<AI 转写全文>

---

## <分段主题1>

<该段内容>

---

## <分段主题2>

...

---

## 核心要点总结

### <主题1>

- 要点1
- 要点2

### <主题2>

...

---

## 相关资源

- **视频**：<原始链接>
- **合集**：<合集链接，如有>

---

## 标签

#标签1 #标签2 #标签3
```

## 技能结构

```
video-note-skill/
├── skill.yaml              # 技能元数据
├── system-prompt.md        # 技能指令
├── transcribe/
│   ├── download.sh         # yt-dlp 下载
│   ├── engines/
│   │   ├── whisper.sh      # Whisper 转写
│   │   ├── lark-minutes.sh # 飞书妙记
│   │   └── minimax.sh      # MiniMax API
│   └── transcribe.sh       # 统一入口
├── templates/
│   └── video-note.md       # 文档模板
└── scripts/
    └── write-to-obsidian.sh # obsidian-cli 调用
```

## 配置项（skill.yaml）

```yaml
name: video-note
default_engine: whisper
obsidian_vault: <vault-name>
output_folder: "01 - 知识库/Learning"
```

## 错误处理

- **下载失败**：提示用户检查链接有效性
- **转写失败**：回退到其他引擎尝试
- **写入失败**：返回错误信息，保留临时文件

## 调用示例

```
# 默认引擎（Whisper）
/video-note https://b23.tv/xxx

# 指定引擎
/video-note https://b23.tv/xxx --engine minimax
/video-note https://b23.tv/xxx --engine lark
```

## 待确认

- [ ] B 站视频元数据（UP主、时长等）如何获取
- [ ] 标签自动提取策略（从内容提取 vs 固定标签）