# Checkpoint

A local tool with four tabs — About, Information, Asked so far, and Common questions —
that lets you paste in text extracted from chat messages or slides, ask a question, and
get an answer from Gemma 4 running locally via Ollama. URLs and numbered field codes are
automatically flagged for manual verification against the original source, every question
asked is logged, and repeated questions are grouped together to show what's worth
answering once instead of over and over. WhatsApp and Telegram chat exports can both be
imported directly.

This was built based on real testing failures (see the project report's Model Limits Log):
Gemma was found to sometimes truncate URLs (e.g. `howGet` → `how`) and fuse form field
numbers into fabricated values (e.g. "2.3B" + "98 days" → "2.3B days"). Rather than trust
the model's own sense of confidence — which testing showed is unreliable — this tool applies
fixed, rule-based flags to the categories of content that failed in testing, and always
shows the full original source text so the flag can be checked at a glance.

## The four tabs

- **About** — what the tool does and how it works, for anyone opening it for the first time.
- **Information** — where you add sources (pasted text, an image read via OCR, or an
  imported WhatsApp export) and ask questions. Every source you add earns a small stamp
  in your collection here, so what you've put in is visible at a glance, not hidden in a
  plain list.
- **Asked so far** — a running log of every question asked this session, which source
  answered it, and how many parts of the answer got flagged.
- **Common questions** — groups repeated or near-identical questions together with a count,
  directly showing the pattern from the project's Problem Statement: the same question
  being asked, and answered, more than once because nobody could see it had already come up.

Questions and sources reset when the page is closed or refreshed — nothing is saved to
disk or sent anywhere, in keeping with everything else in this tool staying local-only.

## Requirements

- [Ollama](https://ollama.com) installed, with `gemma4:e2b` (or another Gemma 4 variant) pulled:
  ```
  ollama pull gemma4:e2b
  ```
- A modern web browser (Chrome, Edge, Firefox)

No other installs, no internet connection needed once Ollama and the model are set up.
No cloud API is used anywhere — every request goes to `http://localhost:11434`, Ollama's
local API, which only ever talks to the model on your own machine.

## Running it

1. Start Ollama's server, allowing local browser requests (see CORS note below):

   **Windows (PowerShell):**
   ```powershell
   $env:OLLAMA_ORIGINS="*"
   ollama serve
   ```

   **Mac/Linux (Terminal):**
   ```bash
   OLLAMA_ORIGINS=* ollama serve
   ```

   Leave this window open and running.

   If Ollama is already running as a background service (e.g. it auto-started with your
   computer), close it first — check the system tray / Activity Monitor for an existing
   Ollama process — then run the command above so the `OLLAMA_ORIGINS` setting takes effect.

2. Open `index.html` in your browser — just double-click the file, no server needed.
   It opens on the **About** tab; click **Information** to get to the working part
   of the tool.

3. Check the status bar at the top of the Information tab. It should say "Connected
   to local Ollama" with a green dot. If it's red, see Troubleshooting below.

4. Paste in your extracted text under "Add what you've got", give it a short name
   (e.g. "Visa steps"), and click "Add this". Add as many sources as you like — chat
   messages, slide text, form instructions, etc.

   **To use an image, PDF, or Word file instead of typing/pasting:** click the file
   upload field and select it.
   - **Images** (a screenshot, a slide, a map) are read using OCR (optical character
     recognition) directly in your browser, via Tesseract.js.
   - **PDFs** have their real text extracted directly via pdf.js — fast and accurate,
     since it reads the actual text layer rather than "looking" at the page. If a
     PDF is scanned images rather than real text (no text layer), it will say so;
     save that page as an image instead and upload it for OCR.
   - **Word documents (.docx)** have their text extracted via mammoth.js.

   **Always review the result before adding it as a source** — OCR in particular is
   not perfect, especially on stylised fonts, small text, or busy backgrounds like
   maps, and can misread characters. This is why the text box stays editable after
   any of these run, rather than adding the source automatically.

   **To import from WhatsApp or Telegram:**

   *WhatsApp* — open the chat, tap the menu, then More, then Export chat, and choose
   without media. This saves a plain `.txt` file straight from WhatsApp itself.

   *Telegram* — open the chat, tap the menu, then Export chat history, and choose
   JSON as the format (not HTML). This saves a `.json` file straight from Telegram
   itself.

   Either way, nothing about this touches the app's servers — it's the app writing
   your own chat history to a file on your device. Upload that file using the
   "Import a WhatsApp or Telegram chat export" field; the tool detects which one it
   is automatically from the file type. It reads every message out and shows them as
   a checklist, so you can filter by sender or keyword and tick only the messages
   worth keeping, rather than dumping an entire chat history in at once — the same
   narrowing principle used everywhere else in this tool. Selected messages get added
   to the text box below for you to review, name, and add as a source as normal.

   There's no way to connect this tool live to a WhatsApp or Telegram account, and
   that's intentional — neither has a local, personal API for that, and anything that
   scraped either app directly would violate its terms of service. The export file is
   the legitimate, fully offline way to bring chat content in.

5. Type a question and click "Ask". The tool picks the single best-matching
   source based on keyword overlap (this mirrors a finding from testing: feeding Gemma
   the whole document at once caused it to lose track of specific details and answer
   with a vague summary instead — narrowing to one relevant source first gave much
   better results).

6. Any URL or numbered field code (like `6.2E` or `2.3B`) in the answer is
   highlighted. Click it to reveal the full original source text underneath, so you
   can check the flagged detail against what was actually said before trusting it.

7. Every question you ask gets logged automatically — check the **Asked so far** tab
   for a running history, and the **Common questions** tab to see which questions have
   come up more than once.

## Why the CORS setting is needed

Browsers block a webpage from calling `localhost` APIs on a different port by default,
for security. `OLLAMA_ORIGINS=*` tells Ollama to accept requests from any local page,
which is what lets `index.html` talk to it directly. This only affects requests reaching
your own machine — it does not expose anything to the internet.

## Troubleshooting

- **Status dot stays red / "Cannot reach Ollama":** confirm `ollama serve` is running in
  a terminal window with `OLLAMA_ORIGINS=*` set (step 1), and that nothing else is using
  port 11434.
- **"Could not reach Gemma" after clicking Ask:** double-check the model name field
  matches exactly what `ollama list` shows (e.g. `gemma4:e2b`).
- **Answer looks empty or unrelated:** the keyword-matching source picker is intentionally
  simple (non-AI) — if your question shares very few words with the right source, it may
  pick the wrong one. Try adding a distinctive keyword from the source into your question,
  or add fewer, more specific sources.

## The grounding score

Every answer shows a percentage — "grounding" — under the answer text. This is
**not** a correctness score, and deliberately so: testing showed Gemma cannot
reliably judge its own accuracy, so asking it "how confident are you?" would just
produce another fabricated-sounding number, no different in kind from the wrong
answers already documented in the Model Limits Log.

Instead, the grounding score is calculated directly, without asking the model
anything: it's the percentage of the answer's significant words that literally
appear in the source text. Low grounding is a genuinely useful signal — it means
Gemma introduced wording that isn't in what you gave it, which is worth a closer
look. High grounding means the answer's wording is traceable to the source; it
does not guarantee the answer is correct, only that it isn't inventing vocabulary
wholesale. This is stated in the tool itself, right under the score.

## Exporting what you've collected

At the bottom of the **Common questions** tab, "Take it with you" lets you save
everything from the current session — every source you added, every question asked
with its answer, and the common-questions grouping — as a single file:

- **Download as PDF** — built entirely in the browser using a small local library
  (jsPDF), loaded from a CDN the first time the page opens.
- **Download as Word** — technically an HTML file saved with a `.doc` extension,
  which Word opens as a normal document. This is a standard lightweight technique
  for producing a Word-compatible file without a heavy library; it is not a true
  binary `.docx`, so very complex formatting won't carry over, but headings,
  paragraphs, and lists all come through correctly.

Both exports happen entirely on your machine — nothing is uploaded anywhere. Like
everything else in this tool, the export is only as good as what you've added and
asked; it's a snapshot of your current session, not a live sync.

## Known limitations (documented, not hidden)

- Source picking is plain keyword overlap, not semantic search — it will occasionally
  pick the wrong source for a vaguely worded question.
- The Common questions tab groups questions by normalized exact wording (lowercase,
  punctuation stripped) — it will not catch two differently-worded questions that mean
  the same thing (e.g. "when do I land" vs. "what's my arrival date"). This was a
  deliberate simplicity tradeoff, consistent with source picking and flagging both
  being rule-based rather than AI-judged throughout this tool.
- Flagging is rule-based (regex for URLs and `\d+\.\d+[A-Za-z]?` field codes), not
  AI-judged — it will sometimes flag correct content and occasionally miss an error
  that doesn't match either pattern. This tradeoff was intentional: testing showed
  Gemma's own confidence is not trustworthy, so a fixed rule is used instead.
- Image input is now supported via in-browser OCR (Tesseract.js), used instead of
  Gemma's own vision, since testing found Gemma 4's vision capability unreliable on
  real map/slide images (see Model Limits Log — it described a real slide as "a
  repeating grid of placeholder cells"). OCR itself is also imperfect, especially on
  stylised fonts, dense annotated maps, or low-contrast text — always review OCR
  output before adding it as a source, which is why it's left editable rather than
  auto-added.
- The OCR library (Tesseract.js) loads from a CDN the first time the page is opened,
  so an internet connection is needed once per session to fetch it. The actual text
  recognition then runs fully locally in the browser — no image or text is sent
  anywhere during OCR or during the Gemma request.
- The grounding score is a word-overlap calculation, not a fact-check — an answer
  can score high while still being wrong (e.g. reusing source vocabulary in an
  incorrect combination, which is exactly the kind of fusion error seen in testing),
  and can score low for a correct answer that happens to paraphrase heavily. It
  should be read as "how much of this is traceable to the source," not "how correct
  this is."
