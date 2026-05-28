---
name: tts-skill
description: Generate natural-sounding speech audio (WAV) from text in 14 languages by calling the hosted aworld TTS API at https://tts-api.aworld.ltd. Returns a public HTTPS URL the user can play, embed, or download. Use whenever the user wants to convert text to speech, narrate text, create a voice clip, generate spoken audio, read text aloud, produce a TTS file, synthesize speech, clone a voice for some text, or mentions any of 文字转语音, 朗读, 语音合成, 配音, TTS, voiceover, text-to-speech — even if they don't explicitly name this API. Also use when the user asks "say this out loud", "make an audio of this", "turn this into a podcast clip", or attaches text with the intent to hear it. The skill handles 14 languages including Chinese, English, Japanese, Korean, German, French, Spanish, Russian and more; reach for it whenever audio output of text is desired. Requires a Bearer token (one-time setup — the skill itself walks the agent through asking the user and persisting it locally).
---

# tts-skill

Convert text to speech via a hosted Confucius4-TTS endpoint. The reference voice is fixed on the server side (a single cloned voice — you don't choose voices). Output is a **public** HTTPS URL to a WAV file that the user can click to play, paste into an `<audio>` tag, or download.

## Endpoint

```
POST https://tts-api.aworld.ltd/tts
Authorization: Bearer <token>
Content-Type: application/json
```

**Request body:**
```json
{ "text": "要朗读的内容", "lang": "zh" }
```

**Response (200):**
```json
{
  "path": "/server/path/tts_<ts>.wav",
  "url":  "https://tts-api.aworld.ltd/audio/tts_<ts>.wav"
}
```

The `url` requires **no authentication** and is the artifact to hand to the user. It stays valid as long as the server retains the file (treat the link as semi-ephemeral — re-synthesize if it 404s).

## Token configuration (first-time setup)

The API requires a Bearer token. The skill never ships a token in source. Resolve it in this order **on every call** — never hardcode:

1. **Environment variable** `AWORLD_TTS_TOKEN` (preferred for CI / one-off shells).
2. **Local config file** `~/.config/aworld-tts/token` — a single line containing the token, mode `600`.
3. **Ask the user** if neither exists, then persist to the config file (so it's a one-time prompt).

### When the token is missing

If both lookups fail, ask the user something like:

> "I need an aworld TTS API token to continue. Ask the API operator for one (it looks like `sk-...`). Paste it here and I'll save it to `~/.config/aworld-tts/token` (chmod 600) so you won't be asked again."

Once the user provides the token, save it:

```bash
mkdir -p ~/.config/aworld-tts
umask 077
printf '%s\n' "<token-from-user>" > ~/.config/aworld-tts/token
chmod 600 ~/.config/aworld-tts/token
```

Then proceed with the request. **Do not echo the token back** in any user-facing response.

### Reading the token

Always re-resolve at call time — the user may rotate. Don't cache across long-running processes.

**Bash:**
```bash
token=${AWORLD_TTS_TOKEN:-$(cat ~/.config/aworld-tts/token 2>/dev/null)}
[ -z "$token" ] && { echo "aworld TTS token not configured — see SKILL.md" >&2; exit 1; }
```

**Python:**
```python
import os, pathlib
def get_token() -> str:
    t = os.environ.get("AWORLD_TTS_TOKEN")
    if t: return t.strip()
    p = pathlib.Path.home() / ".config/aworld-tts/token"
    if p.exists(): return p.read_text().strip()
    raise RuntimeError("aworld TTS token not configured — see SKILL.md")
```

**Node:**
```js
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
function getToken() {
  if (process.env.AWORLD_TTS_TOKEN) return process.env.AWORLD_TTS_TOKEN.trim();
  const p = path.join(os.homedir(), ".config/aworld-tts/token");
  if (fs.existsSync(p)) return fs.readFileSync(p, "utf8").trim();
  throw new Error("aworld TTS token not configured — see SKILL.md");
}
```

## Picking `lang`

Pass the ISO-style code. Detect language from the text when possible; otherwise default to `zh` for Chinese characters, `en` for Latin text.

| Code | Language     | | Code | Language       |
|------|--------------|-|------|----------------|
| zh   | 中文 (Chinese) | | id   | Indonesian     |
| en   | English      | | vi   | Tiếng Việt      |
| ja   | 日本語        | | es   | Español        |
| ko   | 한국어        | | pt   | Português      |
| de   | Deutsch      | | it   | Italiano       |
| fr   | Français     | | ru   | Русский        |
| th   | ไทย         | | ms   | Bahasa Melayu  |

If the requested language isn't on this list, tell the user rather than guessing — output for unsupported languages is garbled. The model is designed for cross-lingual voice transfer, so the same reference voice speaks all 14 languages with a consistent timbre.

## Quick calls

**curl (token from env or config file):**
```bash
token=${AWORLD_TTS_TOKEN:-$(cat ~/.config/aworld-tts/token)}
curl -sS https://tts-api.aworld.ltd/tts \
  -H "Authorization: Bearer $token" \
  -H "Content-Type: application/json" \
  -d '{"text":"你好，世界。","lang":"zh"}'
# {"path":"...","url":"https://tts-api.aworld.ltd/audio/tts_xxx.wav"}
```

**Python:**
```python
import requests

def tts(text: str, lang: str = "zh") -> str:
    r = requests.post(
        "https://tts-api.aworld.ltd/tts",
        headers={"Authorization": f"Bearer {get_token()}"},  # get_token() from above
        json={"text": text, "lang": lang},
        timeout=60,
    )
    r.raise_for_status()
    return r.json()["url"]

print(tts("Hello, world.", "en"))
```

**Node / browser fetch:**
```js
async function tts(text, lang = "zh") {
  const r = await fetch("https://tts-api.aworld.ltd/tts", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${getToken()}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ text, lang }),
  });
  if (!r.ok) throw new Error(`tts ${r.status}: ${await r.text()}`);
  return (await r.json()).url;
}
// usage: const url = await tts("Bonjour", "fr"); audio.src = url;
```

## How to present results to the user

The user wants to *hear* the audio. The most useful reply is the URL itself, optionally wrapped in a clickable/embeddable form. Suggested patterns:

- **Plain text / chat**: just paste the URL on its own line so the client can render it as a link.
- **Markdown context**: `[▶ Play audio](<url>)` or an HTML `<audio src="<url>" controls></audio>` block.
- **Programmatic**: return the URL as the function output; let the caller render.

Do **not** dump the raw JSON unless the user asked for it — they wanted a link, not API plumbing.

## Important behaviors

- **Latency**: ~2-3 seconds for a short sentence; longer for longer text. Set the HTTP client timeout to ≥60s.
- **Length sweet spot**: one sentence to one short paragraph per call. For long articles, split on sentence/paragraph boundaries and call multiple times in parallel.
- **The audio URL is public** — anyone with the link can play/download it. Don't synthesize anything sensitive (passwords, private data, etc.) where link leakage matters.
- **Voice is fixed**: the cloned reference is set server-side. There's no voice/speaker parameter exposed by this endpoint.
- **Same `text` + `lang` will NOT dedupe** — each call generates a fresh WAV. If you need to reuse audio, cache the URL yourself.

## Error handling

| Status | Meaning                         | What to do |
|--------|---------------------------------|------------|
| 200    | OK                              | Use `url`. |
| 400    | `text` empty / missing          | Provide non-empty `text`. |
| 401    | Missing or wrong Bearer Token   | Token invalid or rotated — delete `~/.config/aworld-tts/token` and re-ask the user. |
| 500    | Upstream synthesis error        | Retry once after 2-3s; report if it persists. |
| timeout | Server still working           | Increase timeout or shorten text. |

On 404 from a previously valid `/audio/<file>.wav`: the server cleaned it up — re-synthesize.

## When NOT to use this skill

- The user wants a specific celebrity/character voice → this skill only has one fixed voice.
- The user wants speech-to-text (the opposite direction) → wrong skill.
- The user wants real-time streaming TTS (e.g. for a voice assistant) → this endpoint is synchronous, returns a finished WAV.
- The user wants music generation, sound effects, or non-speech audio → wrong skill.
