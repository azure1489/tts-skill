# Client Integration Guide

> 这份文档面向**开发者**。如果你只是想把这个 skill 装给 AI agent 用，请看 [README](../README.md)。
> 这里的内容用来手写自己的客户端、SDK、或集成进现有系统。

## 目录

1. [概览](#概览)
2. [Base URL & 鉴权](#base-url--鉴权)
3. [接口 1：POST /tts — 合成](#接口-1post-tts--合成)
4. [接口 2：GET /audio/&lt;file&gt; — 下载/播放](#接口-2get-audiofile--下载播放)
5. [错误码](#错误码)
6. [客户端实现示例](#客户端实现示例)
   - [cURL](#curl)
   - [Bash 完整脚本](#bash-完整脚本)
   - [Python (requests)](#python-requests)
   - [Python (async, aiohttp)](#python-async-aiohttp)
   - [Node.js (fetch)](#nodejs-fetch)
   - [Go](#go)
   - [Java (HttpClient)](#java-httpclient)
   - [Rust (reqwest)](#rust-reqwest)
7. [客户端最佳实践](#客户端最佳实践)
8. [常见问题](#常见问题)

---

## 概览

这是一个 HTTPS JSON 服务，把文字转成 WAV 音频，由 [Confucius4-TTS](https://github.com/netease-youdao/Confucius4-TTS) 驱动：

- **同步**：POST 一句话，2-3 秒后拿到永久 URL。无流式、无 WebSocket。
- **零样本声音克隆**：参考音色在服务端固定，所有调用共享同一种音色。
- **跨语种**：14 种语言用同一种音色朗读，音色一致、口音不漂移。

适合的场景：
- 离线生成有声内容（播客片段、配音、提示音）
- 给 web/移动端做朗读功能（直接把返回的 URL 喂给 `<audio>`）
- 多语种通知/播报（一份语料、14 个语种版本）

不适合的场景：
- 实时语音助手（无流式，整句返回）
- 自选音色 / 多角色对话（只有一个固定音色）
- 长视频解说（请按句切分后并发调用）

---

## Base URL & 鉴权

```
Base URL: https://tts-api.aworld.ltd
```

| 路径前缀 | 是否需要 Token |
|---|---|
| `POST /tts`  | ✅ 必须 |
| `GET /audio/*.wav` | ❌ 公开 |
| 其他路径 | ✅ 必须 |

**鉴权方式**：HTTP Header

```
Authorization: Bearer <YOUR_TOKEN>
```

Token 形如 `sk-...`，向 API 运营方索取。

**Token 存放推荐**：
- 环境变量 `AWORLD_TTS_TOKEN`
- 或本地配置 `~/.config/aworld-tts/token`（文本一行，`chmod 600`）
- ❌ 不要硬编码、不要提交到 git、不要打进前端代码（前端调用请走自己的后端代理）

---

## 接口 1：`POST /tts` — 合成

合成一段文本，返回音频 URL。

### Request

```http
POST /tts HTTP/1.1
Host: tts-api.aworld.ltd
Authorization: Bearer <YOUR_TOKEN>
Content-Type: application/json

{
  "text": "要朗读的内容",
  "lang": "zh"
}
```

**字段**

| 字段 | 类型 | 必填 | 默认 | 说明 |
|---|---|---|---|---|
| `text` | string | ✅ | — | 待合成文本。一句到一短段为佳，过长会增加延迟或被截断。 |
| `lang` | string | 否 | `"zh"` | 语种代码，见下表。 |

**支持的 `lang` 取值**

| Code | Language | Code | Language |
|---|---|---|---|
| `zh` | 中文 | `id` | Indonesian |
| `en` | English | `vi` | Tiếng Việt |
| `ja` | 日本語 | `es` | Español |
| `ko` | 한국어 | `pt` | Português |
| `de` | Deutsch | `it` | Italiano |
| `fr` | Français | `ru` | Русский |
| `th` | ไทย | `ms` | Bahasa Melayu |

不在表上的语种**不要传**，否则发音会错乱。

### Response — 200 OK

```json
{
  "path": "/opt/confucius4-tts/outputs/tts_1779963837856237656.wav",
  "url":  "https://tts-api.aworld.ltd/audio/tts_1779963837856237656.wav"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `path` | string | 服务端文件路径（调试/日志用，外部访问没意义） |
| `url`  | string | **公开 HTTPS URL**，直接可下载/嵌入 `<audio>` |

文件名包含纳秒级时间戳，不会重复；同样 `text+lang` 也会产生不同 URL（无去重）。

### 时延

| 文本长度 | 大致耗时 |
|---|---|
| 一句话 (10-30 字) | 2-3 s |
| 一段 (50-100 字) | 4-6 s |
| 一段 (>200 字) | 不保证，建议切句 |

客户端 HTTP 超时建议 ≥ **60 秒**。

---

## 接口 2：`GET /audio/<file>` — 下载/播放

```http
GET /audio/tts_1779963837856237656.wav HTTP/1.1
Host: tts-api.aworld.ltd
```

**无需鉴权**。直接拿到 WAV 二进制。

### Response 头部

```
HTTP/1.1 200 OK
Content-Type: audio/x-wav
Content-Length: 132014
Cache-Control: public, max-age=3600
```

格式固定为 **PCM 16-bit mono 22050 Hz** WAV。

### 用法

- HTML：`<audio src="https://tts-api.aworld.ltd/audio/tts_xxx.wav" controls></audio>`
- 直链分享给用户播放
- 客户端 `wget` / `curl -O` 下载

### 文件保留期

服务端保留窗口有限（视磁盘容量动态清理），曾经可用的链接可能 404。**需要长期持有就立刻下载到自己的存储**。

---

## 错误码

| HTTP | `error` body | 含义 | 客户端处理 |
|---|---|---|---|
| 200 | — | 成功 | 用 `url` |
| 400 | `text is required` | `text` 为空或缺字段 | 校验后重提 |
| 401 | (nginx 401 页) | 缺 / 错 Token | 不要重试，检查 Token |
| 404 | — | `/audio/<file>` 已被清理 | 重新合成 |
| 405 | `method not allowed` | 用了 GET 调 `/tts` 等 | 改用 POST |
| 500 | `{"error":"..."}` | 上游 TTS 模型异常 | 退避 2-5s 后重试一次 |
| 502/504 | (nginx 网关) | 后端短暂不可用 | 退避重试 |
| timeout | — | 文本过长或服务端过载 | 切短文本或重试 |

错误响应 body 形如：
```json
{ "error": "text is required" }
```

> ⚠️ 401 永远不要无脑重试 —— 是 Token 问题，把告警告诉用户去更新 Token。

---

## 客户端实现示例

所有示例都假设环境变量 `AWORLD_TTS_TOKEN` 已设置；如果用配置文件，自己改一下取 Token 的部分。

### cURL

```bash
curl -sS https://tts-api.aworld.ltd/tts \
  -H "Authorization: Bearer $AWORLD_TTS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"text":"你好，世界","lang":"zh"}'
```

直接下载到文件：
```bash
URL=$(curl -sS https://tts-api.aworld.ltd/tts \
  -H "Authorization: Bearer $AWORLD_TTS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"text":"你好","lang":"zh"}' | jq -r .url)
curl -sS -o hello.wav "$URL"
```

### Bash 完整脚本

```bash
#!/usr/bin/env bash
set -euo pipefail

# 取 token：env > 配置文件
token=${AWORLD_TTS_TOKEN:-$(cat ~/.config/aworld-tts/token 2>/dev/null || true)}
if [ -z "$token" ]; then
  echo "ERROR: AWORLD_TTS_TOKEN 未设置，也找不到 ~/.config/aworld-tts/token" >&2
  exit 1
fi

text=${1:-"你好世界"}
lang=${2:-zh}
out=${3:-out.wav}

resp=$(curl -sS -w "\n%{http_code}" https://tts-api.aworld.ltd/tts \
  -H "Authorization: Bearer $token" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg t "$text" --arg l "$lang" '{text:$t, lang:$l}')")

code=$(echo "$resp" | tail -n1)
body=$(echo "$resp" | sed '$d')

if [ "$code" != "200" ]; then
  echo "HTTP $code: $body" >&2
  exit 2
fi

url=$(echo "$body" | jq -r .url)
echo "Synth URL: $url"
curl -sS -o "$out" "$url"
echo "Saved: $out"
```

### Python (requests)

```python
import os
import requests

class AworldTTS:
    BASE = "https://tts-api.aworld.ltd"

    def __init__(self, token: str | None = None, timeout: float = 60.0):
        self.token = token or os.environ.get("AWORLD_TTS_TOKEN")
        if not self.token:
            raise RuntimeError("missing AWORLD_TTS_TOKEN")
        self.timeout = timeout
        self._sess = requests.Session()
        self._sess.headers["Authorization"] = f"Bearer {self.token}"

    def synthesize(self, text: str, lang: str = "zh") -> str:
        r = self._sess.post(
            f"{self.BASE}/tts",
            json={"text": text, "lang": lang},
            timeout=self.timeout,
        )
        if r.status_code == 401:
            raise PermissionError("invalid AWORLD_TTS_TOKEN")
        r.raise_for_status()
        return r.json()["url"]

    def synthesize_to_file(self, text: str, path: str, lang: str = "zh") -> str:
        url = self.synthesize(text, lang)
        with self._sess.get(url, stream=True, timeout=self.timeout) as r:
            r.raise_for_status()
            with open(path, "wb") as f:
                for chunk in r.iter_content(65536):
                    f.write(chunk)
        return path


if __name__ == "__main__":
    tts = AworldTTS()
    print(tts.synthesize("Hello, world.", "en"))
    tts.synthesize_to_file("こんにちは", "hello_ja.wav", "ja")
```

### Python (async, aiohttp)

适合**长文章并发合成**。

```python
import asyncio, os, aiohttp

async def tts_one(session, text, lang="zh"):
    async with session.post(
        "https://tts-api.aworld.ltd/tts",
        json={"text": text, "lang": lang},
    ) as r:
        r.raise_for_status()
        return (await r.json())["url"]

async def batch_synth(sentences, lang="zh", concurrency=4):
    token = os.environ["AWORLD_TTS_TOKEN"]
    headers = {"Authorization": f"Bearer {token}"}
    sem = asyncio.Semaphore(concurrency)
    async with aiohttp.ClientSession(headers=headers,
                                     timeout=aiohttp.ClientTimeout(total=120)) as s:
        async def bound(t):
            async with sem:
                return await tts_one(s, t, lang)
        return await asyncio.gather(*(bound(t) for t in sentences))

# usage:
# urls = asyncio.run(batch_synth(["句子一", "句子二", "句子三"], "zh"))
```

### Node.js (fetch)

Node 18+ 内置 `fetch`。

```js
const BASE = "https://tts-api.aworld.ltd";

export class AworldTTS {
  constructor(token = process.env.AWORLD_TTS_TOKEN, timeout = 60_000) {
    if (!token) throw new Error("missing AWORLD_TTS_TOKEN");
    this.token = token;
    this.timeout = timeout;
  }

  async synthesize(text, lang = "zh") {
    const ac = new AbortController();
    const tm = setTimeout(() => ac.abort(), this.timeout);
    try {
      const r = await fetch(`${BASE}/tts`, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${this.token}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ text, lang }),
        signal: ac.signal,
      });
      if (r.status === 401) throw new Error("invalid AWORLD_TTS_TOKEN");
      if (!r.ok) throw new Error(`tts ${r.status}: ${await r.text()}`);
      return (await r.json()).url;
    } finally {
      clearTimeout(tm);
    }
  }

  async synthesizeToFile(text, path, lang = "zh") {
    const fs = await import("node:fs/promises");
    const { Readable } = await import("node:stream");
    const url = await this.synthesize(text, lang);
    const r = await fetch(url);
    if (!r.ok) throw new Error(`download ${r.status}`);
    await fs.writeFile(path, Buffer.from(await r.arrayBuffer()));
    return path;
  }
}

// usage:
// const tts = new AworldTTS();
// console.log(await tts.synthesize("Hello", "en"));
```

### Go

```go
package aworldtts

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

const Base = "https://tts-api.aworld.ltd"

type Client struct {
	Token string
	HTTP  *http.Client
}

func New(token string) *Client {
	if token == "" {
		token = os.Getenv("AWORLD_TTS_TOKEN")
	}
	return &Client{
		Token: token,
		HTTP:  &http.Client{Timeout: 60 * time.Second},
	}
}

type Result struct {
	Path string `json:"path"`
	URL  string `json:"url"`
}

func (c *Client) Synthesize(ctx context.Context, text, lang string) (string, error) {
	if c.Token == "" {
		return "", errors.New("missing AWORLD_TTS_TOKEN")
	}
	body, _ := json.Marshal(map[string]string{"text": text, "lang": lang})
	req, _ := http.NewRequestWithContext(ctx, "POST", Base+"/tts", bytes.NewReader(body))
	req.Header.Set("Authorization", "Bearer "+c.Token)
	req.Header.Set("Content-Type", "application/json")

	resp, err := c.HTTP.Do(req)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()

	if resp.StatusCode == 401 {
		return "", errors.New("invalid AWORLD_TTS_TOKEN")
	}
	if resp.StatusCode/100 != 2 {
		raw, _ := io.ReadAll(resp.Body)
		return "", fmt.Errorf("tts %d: %s", resp.StatusCode, raw)
	}
	var r Result
	if err := json.NewDecoder(resp.Body).Decode(&r); err != nil {
		return "", err
	}
	return r.URL, nil
}
```

### Java (HttpClient)

需要 Java 11+。

```java
import java.net.URI;
import java.net.http.*;
import java.time.Duration;
import com.google.gson.*;

public class AworldTTS {
    private static final String BASE = "https://tts-api.aworld.ltd";
    private final String token;
    private final HttpClient http = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10)).build();
    private final Gson gson = new Gson();

    public AworldTTS(String token) {
        if (token == null || token.isEmpty())
            token = System.getenv("AWORLD_TTS_TOKEN");
        if (token == null || token.isEmpty())
            throw new IllegalStateException("missing AWORLD_TTS_TOKEN");
        this.token = token;
    }

    public String synthesize(String text, String lang) throws Exception {
        var body = gson.toJson(java.util.Map.of("text", text, "lang", lang));
        var req = HttpRequest.newBuilder(URI.create(BASE + "/tts"))
                .timeout(Duration.ofSeconds(60))
                .header("Authorization", "Bearer " + token)
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(body))
                .build();
        var r = http.send(req, HttpResponse.BodyHandlers.ofString());
        if (r.statusCode() == 401) throw new SecurityException("invalid token");
        if (r.statusCode() / 100 != 2)
            throw new RuntimeException("tts " + r.statusCode() + ": " + r.body());
        return gson.fromJson(r.body(), JsonObject.class).get("url").getAsString();
    }
}
```

### Rust (reqwest)

```toml
# Cargo.toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

```rust
use serde::Deserialize;
use std::time::Duration;

const BASE: &str = "https://tts-api.aworld.ltd";

#[derive(Deserialize)]
struct TtsResp { url: String }

pub struct Client {
    token: String,
    http: reqwest::Client,
}

impl Client {
    pub fn new() -> anyhow::Result<Self> {
        let token = std::env::var("AWORLD_TTS_TOKEN")?;
        Ok(Self {
            token,
            http: reqwest::Client::builder()
                .timeout(Duration::from_secs(60))
                .build()?,
        })
    }

    pub async fn synthesize(&self, text: &str, lang: &str) -> anyhow::Result<String> {
        let r = self.http.post(format!("{BASE}/tts"))
            .bearer_auth(&self.token)
            .json(&serde_json::json!({"text": text, "lang": lang}))
            .send().await?;
        if r.status() == 401 {
            anyhow::bail!("invalid AWORLD_TTS_TOKEN");
        }
        let r = r.error_for_status()?;
        Ok(r.json::<TtsResp>().await?.url)
    }
}
```

---

## 客户端最佳实践

### 超时

- HTTP 总超时 **≥ 60s**，连接超时 5-10s 足够
- 如果设得太短（如 10s），稍长文本就会超时失败

### 重试

只对**幂等且可恢复**的失败重试，且要退避：

| 状态 | 是否重试 | 策略 |
|---|---|---|
| 200 | — | — |
| 4xx (尤其 401) | ❌ 不重试 | 直接报错 |
| 500 | ✅ 最多 1 次 | 退避 3s |
| 502/503/504 | ✅ 最多 2 次 | 指数退避 1s, 4s |
| 超时 | ✅ 最多 1 次 | 检查文本是否过长 |

### 并发

服务端没显式限流，但 GPU 资源有限。建议：

- 单进程同时 in-flight 不超过 **4-8 个请求**
- 长文章用 "切句 + 并发" 而不是 "一次性提交"
- 如果客户端要爆量调用，先 sleep 一下用 1-2 个慢慢发，观察服务端是否会 502

### 长文本切句

中文按 `。！？` 切，英文按 `. ! ?` 切；保留标点。一次请求最多 ~150 字（中文）或 100 词（英文）较稳。

```python
import re
def split_sentences(text, lang):
    if lang == "zh":
        parts = re.split(r"(?<=[。！？])", text)
    else:
        parts = re.split(r"(?<=[.!?])\s+", text)
    return [p.strip() for p in parts if p.strip()]
```

把每句返回的 wav 用 `ffmpeg` 或客户端音频库拼起来。

### 缓存

同样的 `(text, lang)` 服务端不去重，每次都会新生成 wav。客户端如果要省钱省资源，自己做 K-V 缓存：

```
key = sha256(f"{lang}|{text}").hexdigest()
value = url
```

### 前端调用？

**别**直接从浏览器 JS 调，那样 Token 会暴露。流程应该是：

```
浏览器 → 你自己的后端 → tts-api.aworld.ltd
              ↑ Token 保管在这里
```

后端把返回的 `url` 透传给前端即可（URL 公开，没问题）。

---

## 常见问题

**Q: 能选别的音色吗？**
A: 不能。服务端固定了一个克隆好的音色，所有调用共享。

**Q: 能传 SSML / 控制语速、停顿吗？**
A: 当前不支持。

**Q: 音频格式是？**
A: PCM 16-bit mono 22050 Hz WAV。要 mp3/ogg 自己用 ffmpeg 转。

**Q: 同步调用太慢，有 WebSocket 流式吗？**
A: 没有。设计用途是离线生成，不适合实时对话。

**Q: 长文章会被截断吗？**
A: 极长输入可能被模型截断或导致超时。请客户端切句。

**Q: `/audio/<url>` 突然 404 了？**
A: 服务端清理了。重新调一次 `/tts`。需要长期持有就立刻下载到你自己的 OSS。

**Q: 我的 Token 不工作？**
A: 检查 `Authorization: Bearer <token>` 拼写、Token 是否被运营方撤销/轮换。401 不要重试。

**Q: 不同语种音色一样吗？**
A: 一样。Confucius4-TTS 的卖点之一就是跨语种音色一致性。

**Q: 在不支持语种里调用会怎样？**
A: 服务端不会拒绝，但发音会错乱（按近似规则拼读）。客户端先校验。

---

## 反馈

API 行为问题 / 申请 Token / 速率配额商谈：→ https://github.com/azure1489/tts-skill/issues
