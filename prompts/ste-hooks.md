Set up my Reply Standard hooks. Do exactly this, in order. Copy every file exactly as written between the fences, byte for byte, with no edits, no reformatting and nothing added.

1. Ask me one question: "Do you want the hooks at project level or at global level?" Stop and wait for my answer. Do not create anything before I answer.
2. My answer sets BASE and COMMAND_BASE for all of the steps that come after:
   - For project level, BASE is the folder `.claude` in this project. COMMAND_BASE is `${CLAUDE_PROJECT_DIR}/.claude`.
   - For global level, BASE is the folder `$HOME/.claude`. COMMAND_BASE is `$HOME/.claude`.
3. Run `python3 --version`. If that fails, run `python --version`. If that fails too, run `py --version`. The first one that works is PYTHON. If none of them works, stop here, create nothing, and tell me "Python is not installed on this machine".
4. Create the folder BASE/hooks if it is not there.
5. Create BASE/hooks/card.md with exactly this content:
```
REPLY STANDARD (read before you answer)

Prose:
- One instruction or one fact per sentence. At most 20 words in an instruction, 25 in an explanation.
- Active voice. Simple tenses. No "-ing" clause hanging off a comma.
- Plain word over formal: use, before, because, if. Never leverage, utilise, robust, seamless, comprehensive.
- Condition before command: "If the build fails, read the log."
- Name one thing one way for the whole reply.
- Define a technical term the first time it appears, in brackets, in plain words.
- Keep a hedge that carries real doubt. Delete a hedge that carries none.
- No noun stacks over three words. No semicolons. No phrasal verbs (start, not spin up).

Shape:
- The first line answers the question. Everything after it changes what the reader does next, or it is cut.
- Length matches the question: one line for a one-line question.
- Three or more items: a list. A comparison: a table. Never both for the same content.
- No preamble, no restating the question, no closing offer to help.
- No narration before a tool call. Run the tool, then answer.
- If you cut detail, end with one line: "If you want more on X, say so." Never save the cut detail to a file or to memory. The offer is enough.

British spelling. Commas, colons or brackets in place of dashes. Code, paths and quoted errors stay exact.
```
6. Create BASE/hooks/card.py with exactly this content:
```
"""UserPromptSubmit: print the reply standard beside every prompt. The model reads it; the member never sees it.

Routing (deterministic):
  a question or an ask for an explanation  -> the whole card
  a build instruction                      -> the Shape half only
  a one-liner with no question             -> one line back
Loop: if the meter scored the previous reply with violations, the card opens by naming them.
"""
import json
import pathlib
import re
import sys

HERE = pathlib.Path(__file__).resolve().parent
CARD = (HERE / "card.md").read_text(encoding="utf-8")
LOG = HERE / "meter.log"
QUESTION = re.compile(r"\?|^(why|what|how|should|is|are|can|could|which|do|does|explain|tell me|help me understand)\b", re.I)

prompt = (json.loads(sys.stdin.buffer.read().decode("utf-8", "replace")).get("prompt") or "").strip()
sys.stdout.reconfigure(encoding="utf-8")
words = len(prompt.split())

prefix = ""
if LOG.exists():
    last = LOG.read_text(encoding="utf-8").strip().splitlines()
    if last:
        prev = json.loads(last[-1])
        broke = [f"{k.replace('_', ' ')} x{v}" for k, v in prev.get("violations", {}).items() if v]
        if broke:
            prefix = "Your previous reply broke the standard: " + ", ".join(broke) + ". Not this time."
            print(prefix)
            print()

if QUESTION.search(prompt) or words > 12:
    mode = "full card"
    print(CARD)
elif words <= 6:
    mode = "one line"
    print("REPLY STANDARD: one line back, plain words, British spelling, no preamble.")
else:
    mode = "shape half"
    prose, shape = CARD.split("Shape:", 1)
    print("REPLY STANDARD (read before you answer)\n\nShape:" + shape)

# one line per firing, so a board (or a proof) can show what the model was handed
import datetime  # noqa: E402
with (HERE / "card.log").open("a", encoding="utf-8") as f:
    f.write(json.dumps({"at": datetime.datetime.now().isoformat(timespec="seconds"), "mode": mode, "prefix": prefix, "prompt": prompt[:80]}) + "\n")
sys.exit(0)
```
7. Create BASE/hooks/meter.py with exactly this content:
```
"""Stop hook: score the reply that just finished and write one line to meter.log, beside this file. Never blocks, prints nothing.

The counter is SimpleEnglish's ste_lint.py (AminBlg/SimpleEnglish, MIT), unchanged in what it counts, plus two house rules
reported beside the STE total (British spelling, no dashes). A regex pass, not a grammar parser: the same for every reply.
"""
import json
import pathlib
import re
import sys

BANNED_MODALS = re.compile(r"\b(should|would|may|might|could)\b", re.I)
PERFECT = re.compile(r"\b(has|have|had)\s+been\b|\b(has|have)\s+\w+ed\b", re.I)
CONTRACTION = re.compile(r"\b\w+(n't|'ll|'re|'ve|'d)\b|\bit's\b|\byou're\b", re.I)
ING_CLAUSE = re.compile(r",\s*(mak|allow|enabl|ensur|highlight|creat|provid|offer|help|reduc|improv|lead|caus|result)ing\b", re.I)
LATIN = re.compile(r"\b(e\.g\.|i\.e\.|etc\.?)(?=[\s,)]|$)", re.I)
SLOP = re.compile(
    r"\b(simply|seamlessly|effortlessly|robust|leverag\w*|utiliz\w*|"
    r"comprehensive|powerful|blazingly|streamlin\w*|facilitat\w*|"
    r"performant|plethora|myriad|delve|crucial|pivotal)\b", re.I)
TRAILING_COND = re.compile(r"\w[^.!?\n]{3,}\s\b(if|when)\b\s", re.I)
ROTATION_SETS = [
    ("check-verify", re.compile(r"\b(check|verify|confirm|validate|ensure)\w*\b", re.I)),
    ("config-settings", re.compile(r"\b(config|configuration|settings)\b", re.I)),
]
# ours
AMERICAN = re.compile(
    r"\b(\w+iz(e|es|ed|ing|ation|ations)|colors?|behaviors?|favors?|centers?|analyz(e|ed|ing)|catalog)\b", re.I)
DASH = re.compile("[—–]| -- ")
LIMITS = {"procedural": 20, "descriptive": 25}


def strip_code(text):
    text = re.sub(r"```.*?```", " ", text, flags=re.S)
    text = re.sub(r"`[^`\n]+`", " CODESPAN ", text)  # one word per Rule 8.6
    text = re.sub(r"^#+\s.*$", " ", text, flags=re.M)  # headings exempt (titles, 8.6)
    text = re.sub(r"https?://\S+", " URL ", text)
    return text


def sentences(text):
    text = re.sub(r"^\s*([-*]|\d+\.)\s+", "", text, flags=re.M)  # list markers
    parts = re.split(r"(?<=[.!?:])\s+", text)
    return [p.strip() for p in parts if len(p.strip().split()) >= 2]


def lint(text, text_type="descriptive"):
    body = strip_code(text)
    sents = sentences(body)
    limit = LIMITS[text_type]
    counts = {}
    lengths = [len(s.split()) for s in sents]
    counts["sentence_over_limit"] = sum(1 for n in lengths if n > limit)
    counts["contraction"] = len(CONTRACTION.findall(body))
    counts["banned_modal"] = len(BANNED_MODALS.findall(body))
    counts["perfect_tense"] = len([m for m in PERFECT.finditer(body)])
    counts["ing_clause"] = len(ING_CLAUSE.findall(body))
    counts["semicolon"] = body.count(";")
    counts["latin_abbrev"] = len(LATIN.findall(body))
    counts["slop_word"] = len(SLOP.findall(body))
    counts["trailing_condition"] = sum(
        1 for s in sents if TRAILING_COND.search(s) and not re.match(r"^(if|when)\b", s, re.I))
    rotation = 0
    for _, rx in ROTATION_SETS:
        stems = {m.group(1).lower().rstrip("s") for m in rx.finditer(body)}
        if len(stems) > 1:
            rotation += len(stems) - 1
    counts["synonym_rotation"] = rotation
    words = max(1, len(body.split()))
    ste_total = sum(counts.values())
    # ours, reported beside the STE total, never inside it
    house = {"american_spelling": len(AMERICAN.findall(body)), "dash": len(DASH.findall(body))}
    return {
        "type": text_type,
        "words": words,
        "sentences": len(sents),
        "mean_sentence_words": round(sum(lengths) / max(1, len(lengths)), 1),
        "longest_sentence_words": max(lengths, default=0),
        "violations": counts,
        "violations_total": ste_total,
        "violations_per_100w": round(100.0 * ste_total / words, 2),
        "house": house,
    }


HERE = pathlib.Path(__file__).resolve().parent
data = json.loads(sys.stdin.buffer.read().decode("utf-8", "replace"))
text = data.get("last_assistant_message") or ""
if text.strip():
    row = lint(text, "descriptive")
    row["session_id"] = data.get("session_id")
    with (HERE / "meter.log").open("a", encoding="utf-8") as f:
        f.write(json.dumps(row) + "\n")
sys.exit(0)
```
8. If BASE/settings.json already exists, add the "hooks" block below to it and keep everything else in that file as it is. If it does not exist, create it with exactly this content. Make these two replacements in both command lines:
   - Replace `${CLAUDE_PROJECT_DIR}/.claude` with COMMAND_BASE from step 2.
   - If PYTHON is not python, replace the word python with PYTHON (python3 or py).
```
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python \"${CLAUDE_PROJECT_DIR}/.claude/hooks/card.py\"",
            "timeout": 10
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python \"${CLAUDE_PROJECT_DIR}/.claude/hooks/meter.py\"",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```
9. Tell me the four files are in place, the level I chose, and which Python command is in the settings. Do nothing else.
