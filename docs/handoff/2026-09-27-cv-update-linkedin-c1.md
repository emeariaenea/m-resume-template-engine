# Handoff: CV update from LinkedIn + English C1 (2026-09-27)

## Result
- Folder `C:\Users\Mariana\OneDrive\Documents\Recursos\Personal (MGPM)\Currículo\` (each: 1 **Letter** page 8.5×11in, selectable text):
  - `Mariana-Pacheco-CV-2026-09-ESP.pdf`: Spanish. It's Mariana's manually edited PDF, renamed from `-2026-09.pdf` (not regenerated).
  - `Mariana-Pacheco-CV-2026-09-ENG.pdf`: English, rendered from `docs/superpowers/plans/trazo-source-en.html`.
- Original `Mariana-Pacheco-CV.pdf`: untouched.
- Sources (Trazo variant): `docs/superpowers/plans/trazo-source.html` (ES, edited in place) and `trazo-source-en.html` (EN, new file, same CSS). Committed. **CSS is duplicated**: any style change must go into both files.

## Decisions (confirmed by Mariana)
- LinkedIn deltas included: Maestría TECH Universidad (`mar. 2025 – nov. 2026 (en curso)`, placed first in Educación) and `Caracas, Venezuela` location under Xetux.
- LinkedIn deltas excluded on purpose: Google Cloud / .NET Framework skills, extra LinkedIn bullets.
- New section **Idiomas** after Certificaciones: `Español — Nativo · Inglés — C1 (Avanzado)`, rendered with the existing `.certs` style (single line) to fit one page.
- Page size **Carta/Letter** (round 2). CSS: `@page{size:letter}`, `--sheet-w:215.9mm; --sheet-h:279.4mm`.
- Strict 1-page limit. Mariana's rule: if space is short, drop Apps2Go. On Letter it overflowed → **Apps2Go TECH entry removed** (skills list still mentions Python/Node.js/Flutter/Figma).
- Earlier wording trims were reverted after the Apps2Go removal; all bullets keep Mariana's original wording.
- Output named with date suffix; old file kept.
- Trazo timeline aesthetics (round 3, replicating Mariana's manual PDF edit):
  - Node circles centered on the rail: `left:calc(2.5pt + .5px - 4.4pt)` (rail center minus radius incl. border). Was `left:0` → 1.5pt off to the right.
  - No rail stub below the last node: `.entry:last-child::before{display:none;}` (was `height:10pt`).
  - Verified: render vs her edited PDF differs only in the node area by ≤1px (sub-pixel centering); rest pixel-identical.
  - Her edited PDF in `Currículo` was left as-is (not overwritten).
- English version (round 4), `-ENG` / `-ESP` suffixes:
  - Job titles translated literally: Software Engineer, IT Solutions Architect, Junior Software Engineer.
  - Degrees as US equivalents: Bachelor's Degree in Computer Engineering; Master's Degree in Software Engineering and Information Systems (in progress).
  - Languages: Spanish — Native · English — C1 (Advanced). French A1 (old EN CV) NOT added.
  - Headline "Computer Engineer — Backend Development & Software Architecture"; dates `Feb 2025 – Present`; bullets use past-tense action verbs.
  - Certifications fully in English (round 5, Mariana flagged "Spanglish"): "DevOps: Deploying Microservices with Docker", "Programming Foundations: Web Security". The rest were already in English.
  - Text-extraction note: `pypdf` drops "fi"/"ff" ligatures (shows `ef cient`); `pdftotext` extracts them correctly → use `pdftotext` for ATS checks.

## How to re-render
Chrome `--print-to-pdf` CLI **ignores CSS `@page size`** and outputs A4 -> do not use it. Use CDP `Page.printToPDF` with explicit 8.5x11in (Node 24, no deps). Needs internet (IBM Plex from Google Fonts). Save the script below as `print-letter.mjs` outside the repo, then:
```bash
node print-letter.mjs "file:///<repo>/docs/superpowers/plans/trazo-source.html" "<out.pdf>"
```
Verify: pypdf mediabox must be `[0, 0, 612, 792]` and pages == 1. Close the PDF in any viewer first (EBUSY otherwise).

```js
// Usage: node print-letter.mjs <file-url> <out.pdf>
import { spawn } from 'node:child_process';
import { writeFileSync, mkdtempSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

const [src, out] = process.argv.slice(2);
const chrome = 'C:/Program Files/Google/Chrome/Application/chrome.exe';
const port = 9333;
const proc = spawn(chrome, ['--headless=new', '--disable-gpu', `--remote-debugging-port=${port}`,
  `--user-data-dir=${mkdtempSync(join(tmpdir(), 'cvchrome-'))}`, 'about:blank'], { stdio: 'ignore' });

const sleep = ms => new Promise(r => setTimeout(r, ms));
let target;
for (let i = 0; i < 50 && !target; i++) {
  try { target = (await (await fetch(`http://127.0.0.1:${port}/json`)).json()).find(t => t.type === 'page'); }
  catch { await sleep(200); }
}
const ws = new WebSocket(target.webSocketDebuggerUrl);
await new Promise(r => ws.addEventListener('open', r));
let id = 0; const pending = new Map(); const events = [];
ws.addEventListener('message', e => {
  const m = JSON.parse(e.data);
  if (m.id && pending.has(m.id)) { pending.get(m.id)(m); pending.delete(m.id); } else events.push(m.method);
});
const send = (method, params = {}) => new Promise(r => { const i = ++id; pending.set(i, r); ws.send(JSON.stringify({ id: i, method, params })); });

await send('Page.enable');
await send('Page.navigate', { url: src });
for (let i = 0; i < 100 && !events.includes('Page.loadEventFired'); i++) await sleep(100);
await send('Runtime.evaluate', { expression: 'document.fonts.ready.then(()=>1)', awaitPromise: true });
const res = await send('Page.printToPDF', {
  paperWidth: 8.5, paperHeight: 11, marginTop: 0, marginBottom: 0, marginLeft: 0, marginRight: 0,
  printBackground: true, displayHeaderFooter: false, preferCSSPageSize: false,
});
if (res.error) { console.error(res.error); process.exitCode = 1; }
else writeFileSync(out, Buffer.from(res.result.data, 'base64'));
ws.close(); proc.kill();
```

## Gaps / doubts
- Line wraps differ slightly from the original PDF; how the original was rendered is unknown.
- Master's title wraps: "— TECH / Universidad" goes to 2 lines. ATS text extraction reads "TECH\nUniversidad"; minor.
- ~1 line of free space left at the bottom on Letter.
- Maestría placed before Grado (reverse-chronological) by Claude; not explicitly confirmed.
- "Español — Nativo" was proposed by Claude and accepted within the option; not stated separately by Mariana.
- The EN wording (summary, bullets) was written by Claude and hasn't been reviewed by Mariana yet.

## Environment fix (same session)
- `jq: command not found` came from `~/.claude/hooks/always-on-rules.sh` and `~/.claude/statusline-command.sh`. Installed `jqlang.jq` 1.8.2 via winget (`%LOCALAPPDATA%\Microsoft\WinGet\Links` is on user PATH). Needs a Claude Code restart to take effect. Both scripts tested OK with jq on PATH.

## Pending
- Restart Claude Code / terminal so hooks pick up jq.
- Visual QA of ENG wording by Mariana (guide below). ESP + ENG approved on 2026-09-27; changes committed.

## QA guide (ENG)
1. `-ENG.pdf`: 1 page, 8.50 × 11.00 in; nodes centered on the rail, no stub below Keep Typing.
2. Read Summary + each bullet: tone and meaning OK?
3. Education shows Bachelor's/Master's wording; Languages shows Spanish Native · English C1.
4. Copy all → Notepad: no garbled chars.

## QA guide (ESP, earlier rounds)
1. Open the new PDF → exactly 1 page; File > Properties shows 8.50 × 11.00 in. Apps2Go TECH no longer appears; Keep Typing is the last job.
2. Check Xetux shows "Caracas, Venezuela" under the title.
3. Educación: Maestría first, dates `mar. 2025 – nov. 2026 (en curso)`; Grado second.
4. Last section IDIOMAS: `Español — Nativo · Inglés — C1 (Avanzado)`.
5. Ctrl+A / copy all text → paste in Notepad: reading order top-to-bottom, no garbled chars.
6. Optional ATS check: upload to an ATS scanner (e.g. Jobscan) and confirm sections Experiencia/Educación/Idiomas are detected.

## Resume prompt
```
Repo m-resume-template-engine. Read docs/handoff/2026-09-27-cv-update-linkedin-c1.md.
Pending: apply any review feedback on Mariana-Pacheco-CV-2026-09-ENG.pdf (source
docs/superpowers/plans/trazo-source-en.html). CSS is duplicated in trazo-source.html and trazo-source-en.html: style changes go in both. Render with the print-letter.mjs script in the handoff (Letter, 1 page).
```
