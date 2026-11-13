# Dvojček — Complete Project & Chat Reference Manual

> **Purpose:** This document preserves the complete historical context, architectural design, bug-fix records, and codebase state of the **Dvojček** web app. Any AI agent or developer starting a new chat or working in this repository should read this document first for full project context.

---

## 1. Project Overview & URLs

- **App Name:** Dvojček — Slovenska dvojina & prevajalnik (formerly *SloDvojina*)
- **Tagline:** Slovenian Dual & Grammar Explorer / Prevajalnik in generator dvojine
- **Mascot:** Cute dragon mascot (`/mascot.jpg`), featured in top navbar, search hero, practice card, and quiz reactions.
- **Repository:** `https://github.com/wgn9f8b2w4-svg/SLOtranslator.git` (branch: `main`)
- **Live Production URL:** `https://sl-otranslator.vercel.app`
- **Local Working Directory:** `E:\antigravity\Translator app` (moved from `C:\Users\Uros\.gemini\antigravity\scratch\slodvojina`)
- **Local Server Command:** `node server.js` (runs on `http://localhost:3000`)
- **Automated Test Suite:** `npm test` (`node --test test/e2e.test.js`) — **25/25 tests passing**

---

## 2. Core Architecture & 0-Token Runtime Guarantee

> [!IMPORTANT]
> **Zero Runtime AI Token Cost:**
> The live web app does **not** call any LLM or AI model when users translate words or generate grammar tables. Everything is powered by deterministic Node.js algorithms, regex parsers, phonological rule sets, live dictionary scrapers, and external translation/TTS APIs. It costs **$0 and burns 0 AI tokens** when users browse it.

### Tech Stack
- **Backend:** Node.js (ES Modules), Express.js (`server.js`)
- **Frontend:** Single-page application in [`public/index.html`](file:///E:/antigravity/Translator%20app/public/index.html) using Tailwind CSS (CDN), FontAwesome 6, and Vanilla JavaScript.
- **Data Store:** Local verified database at [`data/dictionary.json`](file:///E:/antigravity/Translator%20app/data/dictionary.json).
- **TTS Engine:** Microsoft Edge Neural TTS (`msedge-tts`, voice: `sl-SI-PetraNeural`) + native human recordings from PONS.com.
- **Translation Services:** Google Translate GTX API (`translate.googleapis.com`) + MyMemory fallback (`api.mymemory.translated.net`), with strict Cyrillic-to-Latin transliteration for Serbian (`toLatin`).

---

## 3. Directory Structure & Key Files

```text
E:\antigravity\Translator app\
├── ARCHITECTURE.md          # Visual architecture & Mermaid sequence diagrams
├── CHAT_REFERENCE.md        # This comprehensive project & historical reference
├── data/
│   └── dictionary.json      # Verified Slovenian headwords, SSKJ accents, genders, POS tags, translations
├── franService.js           # Local DB indexing, Fran.si & Franja.si live scrapers, headword validation
├── package.json             # Express, cors, msedge-tts, test script ("node --test")
├── public/
│   ├── index.html           # Main SPA frontend (10 languages, search, word cards, verb UI, practice quiz)
│   └── mascot.jpg           # Official dragon mascot image
├── qa-report.md             # Formal QA and bug-fix verification report
├── server.js                # Express app, /api/analyze, /api/tts, /api/quiz/check, noun declension generator
├── shared/
│   ├── phonemic.js          # Slovenian syllabification, stress-marks, IPA transcription
│   └── textUtils.js         # Diacritic normalization (č, š, ž, ć, đ), Levenshtein fuzzy search, diffing
├── test/
│   └── e2e.test.js          # Comprehensive E2E test suite (25 tests covering R1–R4 & AC1–AC5)
└── verbConjugator.js        # Verb conjugator engine (Sedanjik, Preteklik, Prihodnjik, dual paradigms)
```

---

## 4. Key Subsystems & How They Work

### A. Word Resolution (`franService.js` & `server.js`)
1. **Local Database First:** `findInLocalDictionary(word, userLang)` checks in-memory `Map` indexes (`slIndex` and `foreignIndex`). If found, returns in 0ms without hitting the network.
2. **Live Scrape Fallback:**
   - If input is foreign: Queries **Franja.si** (`https://franja.si/iskanje?sl=<lang>&tl=sl&q=<word>`) with **strict headword equality verification** to prevent false matches.
   - If Slovenian lemma found: Queries **Fran.si (SSKJ)** (`https://fran.si/iskanje?View=1&Query=<slWord>`) for official accentuation, gender, and part-of-speech.
3. **Auto-Growing Database:** `persistNewEntry()` automatically writes newly scraped words back to [`data/dictionary.json`](file:///E:/antigravity/Translator%20app/data/dictionary.json).
4. **Non-Inflecting POS Tagging:** Words classified as `zaimek` (pronoun), `prislov` (adverb), `predlog` (preposition), `veznik` (conjunction), or `členek` (particle) are flagged `nonInflecting: true`, suppressing artificial noun declensions.

### B. Grammar Engines
1. **Noun Declensions (`server.js`):**
   - Handles irregular nouns (`pes`, `otrok`, `človek`, `dan`, `mati`, `hči`, `oko`, `uho`) with custom tables.
   - `generateRegularDeclension()` handles standard 1st feminine (`-a`), neuter (`-o`/`-e`), and masculine (consonant) declensions across all 6 cases (*Imenovalnik*, *Rodilnik*, *Dajalnik*, *Tožilnik*, *Mestnik*, *Orodnik*) for *Ednina*, *Dvojina*, and *Množina*.
2. **Verb Conjugation (`verbConjugator.js`):**
   - Supports `-ati`, `-iti`, `-eti`, and `-ovati`/`-evati` verbs.
   - Supports `-či` verbs (`reči`, `peči`, `teči`, `leči`, `vreči`, and prefixed variants like `preteči`, `odreči`, `speči`).
   - Supports irregular verbs (`biti`, `iti`, `imeti`, `hoteti`, `vedeti`, `moči`, `jesti`, `dati`, `priti`, `vzeti`).
   - Generates dual highlights for 1st, 2nd, and 3rd person dual with localized pronoun explanations (`verbDescriptions`) across all 10 supported UI languages.

### C. Phonemic IPA & Pronunciation (`shared/phonemic.js`)
- Parses SSKJ diacritic stress marks (á, é, ê, è, í, ó, ô, ò, ú, ŕ, ə).
- Splits words into syllables, handles syllabic `r` (e.g. *vrt*, *prst*), regressive voicing assimilation, and final consonant devoicing.
- Outputs syllabic respelling (e.g. `/de-lo-vá-ti/`) and broad IPA (e.g. `[dɛlɔˈʋaːti]`).

### D. Audio Pipeline (`server.js` & PONS / Edge TTS)
- **Primary:** Scrapes PONS.com for native human audio recordings. If an exact headword match exists, returns the PONS URL with a green indicator dot (`source: "pons"`).
- **Secondary:** Synthesizes high-fidelity neural audio via Microsoft Edge Neural TTS (`sl-SI-PetraNeural`) through `/api/tts?text=...&rate=slow|default`.
- **Label Standard:** All audio buttons use the localized string for `"Natural voice"` (`t.naturalVoice`, e.g. `"Prirodni glas"` in SR/HR/BS, `"Naravni glas"` in SL, `"Natural Voice"` in EN).

### E. Interactive Practice / Quiz Mode (`public/index.html` & `server.js`)
- At the bottom of each noun card, users can practice typing the dual form (*dvojina*).
- **Slash-Variant Acceptance:** Words with slash variants like `dan` (`"dva dni / dneva"`) accept any valid form (`"dva dni"`, `"dva dneva"`).
- **Soft Diacritic Tolerance:** Using `diffDiacritics()`, if only carons differ (e.g., typing `hisi` instead of `hiši`), the app gives an amber "Almost correct" badge with a character-by-character visual diff showing where carons were missed, rather than marking it completely wrong.

### F. 10-Language Localization (`i18nData` in `public/index.html`)
Fully translated UI for 10 languages:
- `sr` — Serbian (Latinica)
- `en` — English
- `hr` — Croatian
- `bs` — Bosnian
- `de` — German
- `it` — Italian
- `es` — Spanish
- `fr` — French
- `ru` — Russian
- `sl` — Slovenian

---

## 5. Critical Past Bug Fixes & Lessons Learned

| Bug | Root Cause | Implemented Solution |
| :--- | :--- | :--- |
| **`nekaj` → `Amerikanec`** | Franja.si scraper lacked headword equality checking; when searching `"nekaj"`, it pulled the first example entry on the page (`Amerikanec`), then saved it to `dictionary.json`. | Added strict headword normalization and equality check in `parseFranjaEntry`. Only matches whose primary headword matches the search query are accepted. Removed bad entry from database. |
| **Language-Switch Crash on Verbs** | `selectLanguage(lang)` had a duplicate render block calling `renderWordUI(currentAnalysis)` on verbs, which threw a `TypeError` because verbs have `verbData`, not `wordData`. | Removed duplicate call, added async re-fetching of `/api/analyze?q=...&ui=<lang>` with sequence counter to prevent race conditions, and preserved active UI mode. |
| **Sentence Token Clicks Misclassified Verbs** | Clicking a conjugated verb in a sentence (e.g. `želim` or `potujem`) fell through to noun declension because `resolveBaseLemma` lacked verb mappings. | Expanded `resolveBaseLemma` with 200+ conjugated forms and algorithmic `-ujem -> -ovati` stem reversal; routed single-word inflected verb queries directly to verb conjugation. |
| **Hardcoded "work" Descriptions** | `verbConjugator.js` hardcoded English strings `"The two of us work"` for every single verb in present and future dual summaries. | Replaced with localized `verbDescriptions` dictionary in 10 languages and forwarded `userLang` from `server.js`. |
| **Non-Inflecting Words Declining as Nouns** | Adverbs, pronouns, and prepositions (`danes`, `zelo`, `nekaj`) defaulted to `samostalnik` in `franService.js` and generated fake noun tables (`"en nekaj, dva nekaja"`). | Added `KNOWN_NON_INFLECTING` lookup and POS parser for Fran.si tags (`zaimek`, `prislov`, `predlog`, `veznik`, `členek`); set `nonInflecting: true` to suppress fake declensions. |
| **Quiz Rejection of Valid Slash Variants** | Dual forms like `"dva dni / dneva"` required typing the full slash string literally. | Added automatic splitting of slash strings so either `"dva dni"` or `"dva dneva"` returns `status: "correct"`. |

---

## 6. Verification & Test Commands

Run the full end-to-end test suite at any time from `E:\antigravity\Translator app`:
```bash
npm test
```
The test suite in [`test/e2e.test.js`](file:///E:/antigravity/Translator%20app/test/e2e.test.js) binds dynamically to an ephemeral port (`PORT=0`) to prevent port conflicts and tests:
1. **Cluster 1:** Language switching without crashes, async meaning re-fetch across 10 languages, UI translations, `naturalVoice` buttons, and branding.
2. **Cluster 2:** Verb conjugation accuracy for regular, `-či`, and irregular verbs without hardcoded "work" text.
3. **Cluster 3:** Sentence token lemmatization (`želim` → `želeti`, `potujem` → `potovati`), non-inflecting POS suppression, and quiz slash-variant acceptance.
4. **Cluster 4:** Zero references to dead legacy files.
5. **Acceptance Criteria 1–5:** End-to-end verification of all core user requirements.

---

---

## 8. Front-End Redesign (Hallmark Dusk + Sadō Waza Theme & Dedicated Quiz Hub)

In November 2026, the front-end was upgraded using the **Hallmark** design standard, adopting the **Dusk** structure with the **Sadō Waza** (茶道) Japanese tea-ceremony palette:
- **Design Tokens & Fonts:** `Wix Madefor Display` (500) and `Wix Madefor Text` (400/500). Matcha forest green (`#2f855a` / `#1c5236`), dark slate tea-house base (`#121316` / `#121215`), washi/bone wells (`#fafafa`), and roasted hojicha apricot accents (`#d97706`).
- **Tactile 3D Buttons:** Duolingo / Feather-inspired tactile pressable buttons (`btn-3d-matcha`, `btn-3d-neutral`) with physical click translation and drop shadows.
- **Animated Dragon Mascot (`/mascot.jpg`):**
  - Organic 3-second breathing float (`.mascot-idle`)
  - Celebratory hop/sparkle bounce on streaks and correct answers (`.mascot-celebrate`)
  - Supportive head wobble on mistakes (`.mascot-wobble`)
  - Full `prefers-reduced-motion: reduce` fallback support.
- **Three-Tab Architecture:**
  1. **🔍 Slovar & Dvojina (Explorer):** The full dual generator, noun declensions, verb conjugations, phonemic IPA, and natural voice TTS audio.
  2. **🎮 Vaje & Kviz (Practice & Minigames):**
     - **Adaptive Learner Progress:** Stored in `localStorage` (`dvojcek_quiz_state_v1`), tracking mastery levels (0 to 3 stars), error queues, streaks (`🔥`), and XP (`⚡`).
     - **Na pari (Matching Pairs):** 10 interactive tiles (5 Slovenian words/duals ↔ 5 translations). Tapping matches clears tiles with checkmarks; mismatches trigger haptic shakes. Strictly zero unwanted audio.
     - **Dvojina izziv (Dual Challenge):** Fill-in-the-blank dual prompt with diacritic caron tolerance.
     - **Hitri izbor (Multiple Choice):** 4 tactile 3D cards with 1 correct form and 3 distractors.
     - **Duolingo-Style Fixed Bottom Tray:** Fixed slide-up action bar (`#duo-feedback-tray`) with green celebratory banner for correct answers, amber banner for diacritics, and red banner with corrections for errors. Keyboard support (`Enter` advances).
  3. **🗺️ Učna pot (Learning Path):** Serpentine stepping-stone path previewing Units 1–4, prepared for future Slovenian grammar book integration.
- **Backend API:** Added `GET /api/quiz/pool?lang=...` in `server.js` to serve structured dual pairs and translations across all verified dictionary entries.
- **Linter & Verification:** Passed `sloplint.mjs` with 0 FAILs, and all 25 automated tests in `npm test` (`test/e2e.test.js`) pass with 100% success.

---

## 9. Instructions for Any Future AI Agent

1. **Do Not Introduce AI API Calls into the Runtime:** All word lookups, declensions, and conjugations must remain local/deterministic or use standard HTTP scraping/translation APIs.
2. **Preserve Latin Script for Serbian:** Serbian translations must always run through `toLatin()` (never raw Cyrillic).
3. **Always Run `npm test` After Modifying Code:** Ensure all 25 tests pass before committing or deploying.
4. **Vercel Deployments:** Any commit pushed to `origin/main` automatically triggers a production deployment to `https://sl-otranslator.vercel.app`.
