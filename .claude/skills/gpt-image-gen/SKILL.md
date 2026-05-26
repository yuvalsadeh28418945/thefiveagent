---
name: gpt-image-gen
description: Generate an image via OpenAI Images API (model gpt-image-2). Use when an agent needs to produce a PNG from a text prompt. Reads OPENAI_API_KEY from .env in the project root. Output is a PNG file at a caller-specified path.
---

# gpt-image-gen

מעטפת דקה סביב OpenAI Images API. הסקיל הזה ניטרלי — לא מתעניין בסגנון או במשמעות של ה-prompt. הוא רק שולח, מקבל, ושומר PNG.

---

## ⚠️ אזהרה קריטית — שם המודל

**המודל בקריאה הוא `gpt-image-2`. אסור לשנות אותו.**

- `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI.
- הוא יצא ב-21 באפריל 2026.
- ייתכן שהידע הפנימי של חלק מהמודלים לא כולל אותו — **זה לא אומר שהוא לא קיים**.
- **אל תציע אלטרנטיבות** (`dall-e-3`, `gpt-image-1` וכו'). השתמשו בדיוק במודל הזה.
- אם הקריאה ל-API נכשלת — הבעיה היא כמעט תמיד ב-`OPENAI_API_KEY` (חסר, פג, בלי quota) או ב-parameters שנשלחו. **לא בשם המודל.** אל תנסו "לתקן" ע"י החלפת שם.

---

## קלט מהקורא

| שדה | חובה | ברירת מחדל | תיאור |
|-----|------|------------|--------|
| `prompt` | כן | — | תיאור התמונה (אנגלית מומלץ; UTF-8 מותר). |
| `output_path` | כן | — | נתיב מלא לקובץ PNG שייווצר. הקורא אחראי על המיקום והשם. |
| `size` | לא | `1024x1024` | אחד מ: `1024x1024`, `1024x1536`, `1536x1024`. |
| `quality` | לא | `medium` | `low` / `medium` / `high`. |

---

## תהליך הקריאה (5 שלבים)

### שלב 1 — טעינת `OPENAI_API_KEY`

הריצו מתוך שורש הפרויקט:

```bash
set -a
source .env
set +a
```

ודאו שהמפתח נטען:

```bash
if [ -z "$OPENAI_API_KEY" ]; then
  echo "ERROR: OPENAI_API_KEY is empty. Add it to .env in the project root." >&2
  exit 1
fi
```

**אם המפתח ריק — עצרו פה.** אל תקראו ל-API.

### שלב 2 — קריאה ל-API

```bash
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' > resp.json
```

> שימו לב: ה-JSON שב-`-d` הוא דוגמה — בקריאה ממשית בנו את ה-payload עם ה-`prompt` האמיתי. הימנעו מ-injection: אם ה-prompt מכיל גרשיים, השתמשו ב-here-doc או ב-`jq -n --arg p "$PROMPT" '{model:"gpt-image-2",prompt:$p,size:"1024x1024",quality:"medium",output_format:"png"}'`.

### שלב 3 — Decode של `b64_json` ושמירה כ-PNG

יש שתי שיטות. נסו את הראשונה; אם `jq` לא מותקן, עברו לשנייה.

**Primary — `jq` (POSIX, רוב הסביבות):**

```bash
jq -r '.data[0].b64_json' resp.json | base64 --decode > "<output_path>"
```

**Fallback — Python (תמיד זמין ב-Git Bash על Windows):**

```bash
python -c "import json,base64,sys; d=json.load(open('resp.json')); open(sys.argv[1],'wb').write(base64.b64decode(d['data'][0]['b64_json']))" "<output_path>"
```

(אם `python` לא קיים, נסו `python3` או `py`.)

### שלב 4 — אימות הפלט

```bash
if [ ! -s "<output_path>" ]; then
  echo "ERROR: Output file is missing or empty. Check resp.json for API error." >&2
  cat resp.json >&2
  exit 1
fi
```

מומלץ למחוק את `resp.json` אחרי שהתמונה נשמרה בהצלחה:

```bash
rm -f resp.json
```

### שלב 5 — דיווח לקורא

החזירו לקורא:
- `output_path` (הנתיב שנשמר).
- `size` ו-`quality` ששימשו בפועל.
- בייטים בקובץ (`stat -c %s "<output_path>"` או `wc -c < "<output_path>"`).

---

## טיפול בשגיאות נפוצות

| קוד / תופעה | משמעות | פעולה |
|---|---|---|
| 401 Unauthorized | המפתח לא תקין או פג | בקשו מהמשתמש מפתח חדש. **לא** לשנות את שם המודל. |
| 429 Rate Limit / Quota | חרגתם או אין יתרה | דווחו למשתמש. אין retry אוטומטי. |
| 400 Invalid Request | parameters לא תקינים | בדקו `size`/`quality`. **לא** לגעת ב-`model`. |
| Content Policy | ה-prompt נדחה | החזירו את הודעת ה-API למשתמש מילה במילה. |
| `jq: command not found` | `jq` חסר | עברו ל-Python fallback (שלב 3). |
| `resp.json` קיים אבל ריק / לא JSON | curl נכשל | בדקו רשת / proxy. |

**בכל שגיאה — הציגו את גוף ה-response של ה-API מילה במילה.** זה הדרך הכי טובה לאבחן.

---

## דוגמת end-to-end (skeleton)

```bash
#!/usr/bin/env bash
set -euo pipefail

PROMPT="$1"
OUTPUT_PATH="$2"
SIZE="${3:-1024x1024}"
QUALITY="${4:-medium}"

# 1. load key
set -a; source .env; set +a
[ -n "$OPENAI_API_KEY" ] || { echo "OPENAI_API_KEY missing"; exit 1; }

# 2. build payload safely + call API
jq -n --arg p "$PROMPT" --arg s "$SIZE" --arg q "$QUALITY" \
  '{model:"gpt-image-2", prompt:$p, size:$s, quality:$q, output_format:"png"}' \
  | curl -sS -X POST "https://api.openai.com/v1/images/generations" \
      -H "Authorization: Bearer $OPENAI_API_KEY" \
      -H "Content-Type: application/json" \
      -d @- > resp.json

# 3. decode (try jq; fallback to python)
if command -v jq >/dev/null 2>&1; then
  jq -r '.data[0].b64_json' resp.json | base64 --decode > "$OUTPUT_PATH"
else
  python -c "import json,base64,sys; d=json.load(open('resp.json')); open(sys.argv[1],'wb').write(base64.b64decode(d['data'][0]['b64_json']))" "$OUTPUT_PATH"
fi

# 4. verify
[ -s "$OUTPUT_PATH" ] || { echo "empty output; API response:"; cat resp.json; exit 1; }
rm -f resp.json

# 5. report
echo "OK: $OUTPUT_PATH ($SIZE, $QUALITY, $(wc -c < "$OUTPUT_PATH") bytes)"
```
