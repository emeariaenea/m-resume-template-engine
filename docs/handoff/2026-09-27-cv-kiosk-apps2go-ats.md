# Handoff: CV kiosk + Apps2Go + ATS fixes (2026-09-27, session 2)

Previous handoff (render script, CSS notes): `docs/handoff/2026-09-27-cv-update-linkedin-c1.md`.

## Result
- `Currículo\Mariana-Pacheco-CV-2026-09-ESP.pdf` and `-ENG.pdf` regenerated from
  `docs/superpowers/plans/trazo-source.html` / `trazo-source-en.html`. Both: 1 page, Letter (612×792 pt).
- Previous versions of both PDFs backed up in that session's scratchpad only (temp folder, may be gone).
- Source changes are **uncommitted** (Mariana didn't ask for a commit).

## Decisions (confirmed by Mariana)
- **Kiosk (Xetux):** Mariana is technical lead AND developer; the kiosk is **in production**. Multi-brand, Kotlin.
  - Clients (Beijing, Plan B) are NOT named in public docs (CV/LinkedIn): "cadenas de restaurantes" / "restaurant chains". Named only in the handwritten interview autobiography.
- **Feminine forms (ESP only):** Ingeniera en Informática (headline + Perfil), Ingeniera de Software, Arquitecta de Soluciones TI, Ingeniera de Software Júnior.
- **Merged microservices bullets** (design + implementation) and removed "ágiles"/"agile" from "arquitecturas".
- **Apps2Go TECH re-added** (Nov 2022 – Aug 2023, Miami, Florida, per LinkedIn), 1 bullet. Overrides the earlier "drop Apps2Go" decision, because it fits now.
- **Education ESP:** "Grado en Ingeniería, Computer Science" → "Ingeniería en Informática". No honors mentioned (Mariana's choice for the CV).
- **Perfil:** filler sentence replaced. ESP (final wording from Mariana, plus a comma added by Claude): "Ingeniera en Informática con experiencia en el desarrollo de soluciones de software innovadoras para mejorar la productividad empresarial. Líder técnica de equipos multidisciplinarios, especializada en arquitecturas de microservicios con Java y Spring Boot." It fits in 2 lines. ENG: "Technical lead of multidisciplinary teams under agile methodologies." (fits, so it was kept longer). ESP/ENG wording differs slightly here.
- **ATS (Mariana: "POR COMPLETO ATS friendly"):**
  - Skills: `<dl class="rows">` grid → `<div class="skills">` with one `<p>` per category (`<span class="skill-label">Backend:</span> …`). The grid made extractors read all labels first, then all values.
  - Education: date moved inline into the `<h3>` (`<span class="entry-when">· date</span>`), so each date stays attached to its degree. Before, the dates came out as a separate block.
  - Idiomas/Languages kept as its own section (a brief merge into Educación was reverted for ATS).
  - Verified with `pdftotext -enc UTF-8`: sections in order, dates attached, no garbled chars.
  - `.rows` CSS was left in place (other variants ficha/margen still reference it).

## Gaps / doubts
- ESP kiosk bullet says "para restaurantes, ya en producción" while ENG says "for restaurant chains, now in production" (the ESP was shortened to fit).
- The master's date wraps to its own line ("· mar. 2025 – nov. 2026 (en curso)"). ATS reading order is still correct.
- No metrics in the bullets: Mariana hasn't provided any numbers. Don't invent them.
- LinkedIn still has pending fixes (Cyrillic "Каракас" location, "gestión de recursos humanos", top skills, headline, kiosk as Project). These were given as text in chat; Mariana applies them herself.
- ENG wording hasn't been reviewed by Mariana yet (carried over from the previous handoff).

## QA guide (Mariana)
1. Open both PDFs: exactly 1 page each, 8.50 × 11.00 in.
2. Xetux: the kiosk bullet is first; the microservices bullet is merged; ESP titles are feminine.
3. Apps2Go TECH appears last in Experiencia, with 1 bullet.
4. Skills: 4 lines, each with its label inline (BACKEND: …). It looks fine to you visually.
5. Education: date follows each degree after "·". Idiomas is its own section at the bottom.
6. Ctrl+A → copy → Notepad: order top-to-bottom, no odd characters. Optional: run it through Jobscan.

## Resume prompt
```
Repo m-resume-template-engine. Read docs/handoff/2026-09-27-cv-kiosk-apps2go-ats.md
(and the render script in docs/handoff/2026-09-27-cv-update-linkedin-c1.md).
Pending: apply Mariana's QA feedback on Currículo\Mariana-Pacheco-CV-2026-09-ESP.pdf / -ENG.pdf,
keep 1 Letter page and ATS-clean text (verify with pdftotext -enc UTF-8). Commit only if she asks.
```
