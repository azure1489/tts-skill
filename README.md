# tts-skill

一个可被任意 AI agent 安装使用的**文本转语音 (TTS) skill**，包装了 [aworld TTS API](https://tts-api.aworld.ltd)（基于 Confucius4-TTS，14 语种零样本声音克隆）。

调用后返回一个**公开可访问的** WAV 链接，可直接在浏览器播放、嵌入 `<audio>` 标签或下载。

```
User → "把这段话念出来"
   ↓
Agent (with this skill) → POST /tts
   ↓
Returns: https://tts-api.aworld.ltd/audio/tts_xxx.wav
```

> **📘 想直接写客户端 / 集成进自己的系统？** 看 [`docs/CLIENT.md`](docs/CLIENT.md) — 完整 API 参考 + cURL / Python / Node / Go / Java / Rust 可粘贴示例 + 超时重试并发等工程细节。

## 特性

- 14 语种：中、英、日、韩、德、法、西、葡、意、俄、泰、印尼、越南、马来
- 单次合成 ~2-3 秒
- 返回的音频 URL 公开可访问，无需鉴权即可播放/下载
- Token **不打包在仓库里**，首次使用时由 agent 询问用户并保存到本地

## 你需要什么

- 一个 **Bearer Token**（向 API 运营方索取，形如 `sk-...`）
- 一个会读 SKILL.md 的 AI agent（Claude Code / Claude Desktop / 自建 agent 均可）

## 安装

### 方式 1：Claude Code（推荐）

```bash
git clone https://github.com/azure1489/tts-skill.git
mkdir -p ~/.claude/skills
cp -r tts-skill/tts-skill ~/.claude/skills/
```

下次启动 Claude Code 时，该 skill 会自动出现在可用列表中。

### 方式 2：打包成 `.skill` 文件

```bash
git clone https://github.com/azure1489/tts-skill.git
cd tts-skill
zip -r tts-skill.skill tts-skill
```

然后把 `tts-skill.skill` 拖进 Claude Code 安装。

### 方式 3：其它 agent

`tts-skill/SKILL.md` 就是一份纯 Markdown 文档。直接把它读进 agent 的 system prompt / RAG 索引即可。

## 首次配置 Token

第一次让 agent 用这个 skill 时，它会要你提供 token。**你不需要手动配置任何东西**，按 agent 提示输入即可。

agent 内部会按下面的顺序找 token：

1. 环境变量 `AWORLD_TTS_TOKEN`
2. 本地文件 `~/.config/aworld-tts/token`（一行纯文本，权限 600）
3. 都没找到 → 询问你 → 写入上面那个文件 → 以后再不打扰

### 想提前手动配置？

```bash
mkdir -p ~/.config/aworld-tts
echo "sk-你的token" > ~/.config/aworld-tts/token
chmod 600 ~/.config/aworld-tts/token
```

或临时用环境变量：

```bash
export AWORLD_TTS_TOKEN="sk-你的token"
```

### 更换 token

```bash
rm ~/.config/aworld-tts/token
# 下次调用时 agent 会重新询问
```

## 使用示例

让 agent 做什么都行，自然语言：

- "把这段话念出来：床前明月光，疑是地上霜"
- "Generate audio for: One voice, any language."
- "做个有声版的，日语：こんにちは、世界"
- "TTS 一下这段话发我"

agent 应该返回类似：

> ▶ [Play audio](https://tts-api.aworld.ltd/audio/tts_1779964409672809233.wav)

## 直接调用 API（不用 agent）

如果你只想用 curl 直接调：

```bash
TOKEN=$(cat ~/.config/aworld-tts/token)
curl -sS https://tts-api.aworld.ltd/tts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"text":"你好，世界","lang":"zh"}'
# → {"path":"...","url":"https://tts-api.aworld.ltd/audio/tts_xxx.wav"}
```

详细的 API 文档见 [tts-skill/SKILL.md](tts-skill/SKILL.md)。

## 安全说明

- Token 应当像密码一样保管。仓库本身不含 token，但你自己的 `~/.config/aworld-tts/token` 是明文，请勿提交到任何版本控制。
- 生成的音频 URL 是**公开的**，任何拿到链接的人都能下载。请不要合成密码、私密信息等敏感内容。

## 反馈 / 问题

提 issue：https://github.com/azure1489/tts-skill/issues

## License

MIT — 见 [LICENSE](LICENSE)
