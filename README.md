# TKC_TRANSCRIPT

TKC show transcripts — auto-generated from YouTube via Supadata API.

## Files

Transcripts are named `<MMDDYY>_TS.md` (e.g. `071026_TS.md` for July 10, 2026).

Two timestamp formats exist:
- `(MM:SS)  text` — parenthesized timestamps
- `MM:SS\ntext` — bare timestamps on separate lines

## Searching Transcripts

Download the **Manage Transcripts** skill to give your AI agent (opencode, Claude Code, Aider, Hermes, etc.) the ability to search, summarize, list, and analyze transcripts without cloning the repo:

```bash
# Linux / macOS
mkdir -p .opencode/skills/manage-transcripts
curl -sL "https://raw.githubusercontent.com/ProducerJenn/TKC_TRANSCRIPT/main/.opencode/skills/manage-transcripts/SKILL.md" \
  -o .opencode/skills/manage-transcripts/SKILL.md
```

```powershell
# Windows (PowerShell)
New-Item -ItemType Directory -Force -Path ".opencode/skills/manage-transcripts"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/ProducerJenn/TKC_TRANSCRIPT/main/.opencode/skills/manage-transcripts/SKILL.md" `
  -OutFile ".opencode/skills/manage-transcripts/SKILL.md"
```

Once installed, ask your AI agent to search, summarize, or list transcripts. (For non-opencode agents, point them to the `.opencode/skills/manage-transcripts/SKILL.md` file or copy the relevant workflows.)

### Quick search (no skill needed)

```bash
curl -s "https://api.github.com/repos/ProducerJenn/TKC_TRANSCRIPT/contents" | \
  jq -r '.[].name' | grep '_TS.md' | sort | while read f; do
  content=$(curl -s "https://raw.githubusercontent.com/ProducerJenn/TKC_TRANSCRIPT/main/$f")
  echo "$content" | while IFS= read -r line; do
    if echo "$line" | grep -qi "search term"; then
      ts=$(echo "$line" | grep -oP '^\(?\d+:\d+' || echo "??:??")
      echo "[$f @ $ts]  $line"
    fi
  done
done
```

Requires `curl` and `jq`.
