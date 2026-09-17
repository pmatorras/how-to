# Blocking em dashes in Claude Code (hook setup)

## Why this exists

One can set `CLAUDE.md` / global preferences saying "never use em dashes", but is not fully reliable on its own, it takes it as a strong recommendation, not a hard rule.
A hook fixes this because it runs deterministically, outside the model's control, every time a file is written or edited, blocking any message that has em dashes, and not relying on Claude "remembering."

## What it does

A `PostToolUse` hook fires after every `Edit`/`Write`. It greps the file that was just touched for `—`. If found, it returns a `block` decision with the
offending line(s) quoted back to Claude, telling it to rewrite that line naturally (comma, period, split sentence) rather than mechanically deleting
the character. This avoids the "blind find-and-replace" problem, where a regex swap produces stitched-together, unnatural sentences.

## One-time setup on a new machine

**1. Install `jq`** (the script needs it to parse the hook's JSON input) if it isn't there:
```bash
sudo apt install jq
```

**2. Create the hook script** at `~/.claude/hooks/em-dash-check.sh`:
```bash
#!/usr/bin/env bash
# ~/.claude/hooks/em-dash-check.sh
input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

[ -z "$file_path" ] || [ ! -f "$file_path" ] && exit 0

hits=$(grep -n '—' "$file_path")
if [ -n "$hits" ]; then
  jq -n --arg reason "Em dash found in $file_path:
$hits
Rewrite these lines naturally (comma, period, or split sentence). Don't just delete the character." \
    '{"decision": "block", "reason": $reason}'
else
  exit 0
fi
```

**3. Make it executable:**
```bash
chmod +x ~/.claude/hooks/em-dash-check.sh
```

**4. Add the hook to `~/.claude/settings.json`** (global, applies to every
project). Keep your existing keys, just add `hooks`:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "~/.claude/hooks/em-dash-check.sh" }
        ]
      }
    ]
  }
}
```

**5. Restart your Claude Code session** so the new settings load.

## Testing it works

```bash
echo "this is a test — with an em dash" > /tmp/test.md
echo '{"tool_input":{"file_path":"/tmp/test.md"}}' | ~/.claude/hooks/em-dash-check.sh
```
Expect JSON output containing `"decision": "block"`. No output means either
no em dash was found in the test file, or the file path didn't exist, check
both before assuming the hook is broken.

## Notes

- This only catches em dashes in files Claude edits or writes (code
  comments, `.md` docs, etc). It does not touch plain chat responses, hooks
  can't intercept those.
- Also worth doing in `CLAUDE.md`: replace the abstract "never use em dashes"
  rule with 2-3 concrete before/after examples in the style where it
  actually shows up (short code comments), concrete examples in-context
  stick better than a general prose rule.

---
# Make it sound more natural
Following reddit advice, i tried too add this to `~/.claude/output-styles/eli5.md`, to make it sound more natural
```
---
name: ELI5
description: keep it simple pls
keep-coding-instructions: true
---
Talk to me in small words. Not because I don't know the domain. I do. The problem is burying the answer under qualifiers, shorthand, and names only you have open.

Small words, short sentences, short paragraphs. One idea per sentence. If you have to use a big word, explain it right after. Only return what's actually
necessary. Lead with the answer. First line, every time. Then the detail, if there is any.

Name things in words, not in numbers. Never send me to another document to decode a sentence. 
- Not "section 14b": say "the rule that Japanese has to sound like me." 
- Not "AGENTS.md section 5": say "the business rules: Grade 5 minimum, pricing, no employer names." 
The file name can go at the end for whoever needs to open it.

Just tell me what you did, did it work, what do I do now.

Tell me when I'm wrong. Say it first and say it plainly, in the same small words. Don't soften it, don't bury it in options, don't wait until the end. If my plan won't work, the first line is "that won't work" and then why. Being easy to read never means being agreeable.

Same for your own mistakes. If you shipped something wrong, say so straight and fix it. No long apology.

Keep paths, commands, filenames, and numbers exact. I have no brain cells left for the rest.
```
