# Code Roaster — Full Build Prompt (Antigravity)

One single prompt that builds the entire app end-to-end. Paste it into Antigravity's agent after you've run the Stitch design prompt and have the reference screens open. The agent should scaffold the project, then create every file described below, in order, in one session.

---

```
Build "Code Roaster" from scratch: a single-page web app that roasts a user's pasted code. It returns a witty critique, a structured list of issues by severity, and a corrected version of the code. One-line pitch: "paste your code, get roasted, walk away with the fix."

Problem: generic linters and AI chat tools give dry, forgettable feedback that beginner/intermediate developers skim past.
Goal: make code review memorable by pairing real issue detection with comedic delivery (Hinglish + emojis, adjustable intensity) while still returning technically correct, usable fixes.
Audience: students, hackathon participants, devs wanting quick, entertaining feedback on a single snippet — not a team production code-review tool.

Non-goals: no auth/accounts, no persistence/history/DB, no multi-file or repo-level analysis, no collaboration features, no automated test suite/CI.

Tech stack:
- IDE/agent: Antigravity (you)
- Framework: Next.js 16, App Router
- UI: React 19 + TypeScript
- Styling: Tailwind CSS v4
- AI: Google Gemini via the official `@google/genai` SDK, model "gemini-3.5-flash-lite"
- Hosting target: Vercel (keep it deployable, no Vercel-specific work needed now)
- Secrets: the Gemini API key must live in `.env.local` and be read ONLY on the server — it must never reach the browser bundle.

Visual direction: follow the Stitch design reference I'm showing you — an "architectural blueprint / technical drafting desk" theme. Sharp corners (no border-radius), visible 1-2px borders on every panel, a faint drafting grid in the page background, monospace-heavy typography, small uppercase tracked-out labels like "01 // ROAST". Color tokens: ink/frame #111111, canvas #F5F5F3, panel #FDFDFD, subtle border #E6E6E2, accent #D94826, plus severity colors (fatal = red-orange, code smell = amber, optimization = green). Fonts: Space Grotesk (display/headings) via next/font/google as --font-space-grotesk, JetBrains Mono (everything else — body, labels, code, buttons) as --font-jetbrains-mono. Map both into Tailwind v4 via @theme / @theme inline in globals.css, along with the color tokens above as --color-frame, --color-canvas, --color-panel, --color-subtle, --color-accent. Give the page body a subtle drafting-grid background (two low-opacity 1px linear-gradients, 24px x 24px tile) and a thin black-on-light scrollbar.

Core user flow:
1. User pastes code, picks a language (8 supported: Python, JavaScript, TypeScript, Java, C, C++, Go, Rust) and a roast level (Dry / Sharp / Savage, mildest to harshest).
2. User can optionally add an error message / stack trace via a collapsible drawer.
3. User clicks "Roast" (or presses Ctrl+Enter / Cmd+Enter from anywhere on the page) — input is validated client- and server-side.
4. The server calls Gemini with a structured system prompt and a fixed JSON response schema.
5. The UI shows: the roast text, a list of issues with severity tags, the fixed code, and a short takeaway.

Features: roast levels (Dry/Sharp/Savage, each with a distinct tone), 8 languages with label+extension, severity tags (FATAL BUG, CODE SMELL, OPTIMIZATION), a "Load Sample Bug" button, copy-to-clipboard and "Apply to Editor" on fixed code, input limits (20,000 chars code / 4,000 chars error message, enforced client- and server-side), auto-retry up to 3 attempts on Gemini 503 with increasing backoff.

Architecture (stateless, no database, no auth):
CodeEditor (browser) -> Workspace (owns all state) -> lib/api.ts (fetch helper) -> POST /api/roast (validates input) -> lib/gemini.ts (calls Gemini, using lib/prompt.ts + lib/schema.ts) -> Gemini API -> RoastResult JSON -> RoastReport renders IssueCards + FixedCode.

Now build these files exactly as specified:

=== 1. Scaffold ===
Create a new Next.js 16 App Router project, React 19, TypeScript, Tailwind v4, using `@google/genai` (not the older `@google/generative-ai`). Set up fonts in app/layout.tsx per the visual direction above. Root layout metadata: title = `${APP.name} — AI Code Critique`, description = APP.tagline. <html> gets both font variables + `h-full antialiased`; <body> gets `min-h-full flex flex-col`. app/page.tsx is just a centered container rendering <Workspace />. Add .gitignore excluding node_modules, .next, .env.local (not .env.example). Create .env.example with `GEMINI_API_KEY=` and a comment linking to https://aistudio.google.com/apikey — do NOT create or write into .env.local yourself, that's mine to fill in manually.

=== 2. config/app.config.ts ===
The single source of truth, imported by both browser and server — never put secrets here. Export:
- APP = { name: "Code Roaster", version: "v3", tagline: "Your code. Our problem now." }
- AI = { model: "gemini-3.5-flash-lite", modelLabel: "Gemini 3.5 Flash-Lite", maxAttempts: 3 }
- ROAST_LEVELS (as const, mildest to harshest), each { id, label, description }: dry/"Dry"/"Mild and deadpan. Gentle jabs, mostly helpful."; sharp/"Sharp"/"Pointed and witty. Calls out every mistake directly."; savage/"Savage"/"Maximum burn. Brutally honest, but still technically accurate."
- LANGUAGES (as const), each { id, label, extension }: python/Python/py, javascript/JavaScript/js, typescript/TypeScript/ts, java/Java/java, c/C/c, cpp/C++/cpp, go/Go/go, rust/Rust/rs.
- DEFAULTS = { language: "python", roastLevel: "savage" } as const
- SEVERITIES = ["FATAL BUG", "CODE SMELL", "OPTIMIZATION"] as const
- LIMITS = { maxCodeLength: 20_000, maxErrorMessageLength: 4_000 }
- SAMPLE = { language: "python", code: <a Python function that averages a list of numbers but does `total += numbers` instead of `total += number` inside the loop, causing a TypeError> } as const

=== 3. types/roast.ts ===
Derive types FROM the config arrays (so adding a language/level later updates types automatically): LanguageId, RoastLevel, Severity as unions off LANGUAGES/ROAST_LEVELS/SEVERITIES ids. Then: RoastRequest { language, code, roastLevel, errorMessage? }; RoastIssue { line, severity, title, codeSnippet, diagnosis, expected }; RoastResult { roast, issues: RoastIssue[], correctedCode, takeaway }; ReportState = "empty" | "loading" | "results" | "error".

=== 4. lib/prompt.ts (the AI personality — get this exact) ===
Build a roastLevelGuide by mapping ROAST_LEVELS into "- {id}: {description}" lines. Export ROAST_SYSTEM_INSTRUCTION with this exact content:
- "You are a Code Roaster. Your job is to analyze user-submitted code and provide a structured critique."
- Personality: observational, concise, deadpan, technically grounded, spontaneous; understandable to college students but never condescending; funny like a senior roasting a junior in the college lab — witty, never insulting the person, only the code.
- Language and style (very important): write in Hinglish — Hindi words written in English/Roman letters, mixed naturally with simple English. Example verbatim: "Bhai, yeh loop har baar poori list add kar raha hai 😅. Python bhi soch raha hoga ki kya chal raha hai 🤦". Never use Devanagari script, only Roman letters. Short, simple sentences — students aren't fluent in English. Keep technical terms in English (loop, variable, function, list, TypeError, etc.) so students learn the real terms. Add emojis (😂 🔥 💀 🤦 😅 ✅ 🚀), roughly 1-3 per text field, don't overdo it. Use Hinglish + emojis ONLY in "roast", "title", "diagnosis", "expected", "takeaway". Do NOT use Hinglish or emojis inside "codeSnippet" or "correctedCode" — those must be valid code; comments in correctedCode may be short simple English.
- Interpolate roastLevelGuide under "Adjust the intensity of the 'roast' text to the requested roast level:".
- Analyze the code for: fatal bugs/logic errors/syntax issues, performance bottlenecks, architectural smells, best practices violations.
- Rules for the response: "line" is the 1-based line number where the issue appears; "severity" must be exactly one of the SEVERITIES values (always English, no emojis); "codeSnippet" is the exact problematic code copied from the submission; list the most serious issues first, empty issues array if none found; "correctedCode" is the complete fixed program in the same language, plain code with no markdown fences; keep technical explanations accurate even when the roast is harsh.
- Close with: "Return a JSON object conforming exactly to the requested schema."
Export buildUserPrompt({ language, code, roastLevel, errorMessage }): joins, filtering empty lines: `Language: {language}`, `Roast Level: {roastLevel}`, `Error Message:\n{errorMessage}` (only if present), and a fenced block `Code:\n\`\`\`{language}\n{code}\n\`\`\``.

=== 5. lib/schema.ts ===
Using `Type`/`Schema` from "@google/genai" and SEVERITIES from config, export roastResponseSchema: Schema, an OBJECT with properties: roast (STRING), issues (ARRAY of OBJECT { line: INTEGER, severity: STRING enum=SEVERITIES, title: STRING, codeSnippet: STRING, diagnosis: STRING, expected: STRING }, all six required), correctedCode (STRING), takeaway (STRING). Top-level required: ["roast", "issues", "correctedCode", "takeaway"]. No extra properties beyond this.

=== 6. lib/gemini.ts (server-only, never imported client-side) ===
Export async analyzeCode(request: RoastRequest): Promise<RoastResult>. Read process.env.GEMINI_API_KEY, throw if missing ("GEMINI_API_KEY is missing. Copy .env.example to .env.local, add your key, and restart the server."). Create `new GoogleGenAI({ apiKey })`. Call ai.models.generateContent with model: AI.model, contents: buildUserPrompt(request), config: { systemInstruction: ROAST_SYSTEM_INSTRUCTION, responseMimeType: "application/json", responseSchema: roastResponseSchema }, inside a retry loop: on ApiError with status 503 and attempt < AI.maxAttempts, wait `attempt * 1000`ms and retry; otherwise throw via a friendlyErrorMessage(status, error) helper mapping 400/403/404/429/503/default to specific actionable messages (mention .env.local, AI.model in config/app.config.ts, rate limits, overload, as appropriate). Parse response.text as JSON (throw clear errors on empty/invalid), then return with defensive fallbacks: `{ roast: result.roast ?? "", issues: Array.isArray(result.issues) ? result.issues : [], correctedCode: result.correctedCode ?? "", takeaway: result.takeaway ?? "" }` so the UI never crashes on a partial answer.

=== 7. scripts/check-api.ts ===
Standalone script (not part of the Next app): import analyzeCode, AI, SAMPLE; call analyzeCode with SAMPLE code, roastLevel "sharp", and a sample TypeError errorMessage; log the roast + issue count on success, or the error and process.exit(1) on failure. Add npm script `"check:api": "tsx --env-file=.env.local scripts/check-api.ts"`.

=== 8. app/api/roast/route.ts ===
POST handler: parse JSON body (400 on parse failure), read code (string, default "") and errorMessage (trimmed string, default ""). Validate in order, 400 + JSON {error} on first failure: empty code -> "No code provided."; code over LIMITS.maxCodeLength -> length message; errorMessage over LIMITS.maxErrorMessageLength -> length message. For language/roastLevel: if not a known id from LANGUAGES/ROAST_LEVELS, silently fall back to DEFAULTS instead of erroring. Call analyzeCode(...) in a try/catch: 200 + JSON result on success; on failure console.error and return 500 + { error: message }.

=== 9. lib/api.ts (browser-only fetch helper) ===
Export async requestRoast(payload: RoastRequest): Promise<RoastResult> — the ONLY place browser code talks to our backend. POST to /api/roast; wrap fetch itself in try/catch rethrowing "Could not reach the server. Is `npm run dev` still running?" on network failure. Parse response JSON defensively (`.catch(() => null)`). If !response.ok || !data, throw `data?.error` or a generic "Server error ({status})." message. Otherwise return data as RoastResult.

=== 10. components/Workspace.tsx ("use client", the only state owner) ===
State: roastLevel, language, code, errorMessage, errorDrawerOpen, reportState ("empty"|"loading"|"results"|"error"), roastResult, roastedCode (snapshot of code actually sent), apiError. isRoasting = reportState === "loading".
handleRoast (useCallback, no-op while isRoasting): empty code -> apiError "No code provided. I can't roast the void." + reportState "error"; else reportState "loading", clear result/error, call requestRoast, on success set result + roastedCode snapshot + reportState "results", on failure set apiError + reportState "error".
handleLoadSample: sets language + code from SAMPLE.
useEffect: window keydown listener for Ctrl/Cmd+Enter -> preventDefault + handleRoast, from anywhere on the page, cleaned up on unmount.
errorLine: only defined when reportState === "results" && code === roastedCode (i.e. editor still holds exactly what was roasted) -> roastResult?.issues[0]?.line; must go undefined the instant the user edits the code even though reportState stays "results".
Render: a centered "workstation" <main> card (max-w-[1360px], near-full-viewport height, 2px border, subtle shadow, flex column, overflow hidden) containing <TopBar> wrapping <RoastControls>, a conditionally-rendered <ErrorMessageInput> (only when errorDrawerOpen), a responsive flex row (column on mobile) with <CodeEditor> (~54-55% width desktop) and <RoastReport> (~46-45% width desktop), and <StatusBar> at the bottom.

=== 11. components/TopBar.tsx ===
Wrapper accepting { children }. <header> with bottom border, light background, flex (column mobile / row desktop, space-between). Left: app name (bold uppercase tracked-out sans) + small bordered version tag from APP config. Right: render children (RoastControls) in a wrapping flex row, small mono text.

=== 12. components/RoastControls.tsx ===
Props: roastLevel, onRoastLevelChange, language, onLanguageChange, onRoast, isRoasting, errorDrawerOpen, onToggleErrorDrawer. Render: (1) "Roast Level:" + a radio group from ROAST_LEVELS as custom circular radios with labels and description tooltips; (2) "Language:" + a native <select> from LANGUAGES, custom-styled (no default appearance, bordered, mono, small ▾ indicator); (3) an error-drawer toggle button (dashed border, "+ ERROR MESSAGE" / inverted "− ERROR MESSAGE"); (4) the primary action button — solid dark bg, white bold uppercase text, "ROAST MY CODE" + small "Ctrl ⏎" hint badge, replaced by a pulsing disabled "ANALYZING..." while isRoasting.

=== 13. components/ErrorMessageInput.tsx ===
Props { value, onChange, onClose }. A bordered drawer strip: small uppercase label "Attach Terminal Traceback / Compiler Error (Optional)" + "Dismiss ✕" button calling onClose, then a 2-row <textarea> (bordered, mono, placeholder like a real TypeError example) controlled by value/onChange.

=== 14. components/CodeEditor.tsx ("use client") ===
Props { code, onChange, language, errorLine?, onLoadSample }. A plain <textarea> + synced line-number gutter (no external editor library). Track cursor (line, col) via onSelect counting newlines before selectionStart. Sync gutter scrollTop to textarea scrollTop via a ref on the gutter's onScroll. Tab key (not Shift+Tab): preventDefault, insert 4 spaces via setRangeText, call onChange — Tab must never blur the textarea. Line count = max(code.split("\n").length, 10). Layout: 40px header strip ("INPUT // SRC" muted left, "Your Code" bold center, "SAMPLE BUG" / "CLEAR" text links right), gutter + textarea row (highlight the errorLine row in accent color + soft accent background, bold; textarea: monospace, no wrap, spellcheck off, placeholder "// Paste your code here..."), bottom status strip overlay ("Ln {line}, Col {col} · {languageLabel}" left, "UTF-8 · Tab Size: 4" right, hidden on mobile).

=== 15. components/SectionHeader.tsx ===
Props { number, title, children? }. Flex row with bottom border: left = bold uppercase tracked mono text `{number zero-padded to 2 digits} // {title}` (e.g. "01 // ROAST"); right = children if provided.

=== 16. components/IssueCard.tsx ===
Props { index, issue }. Severity->style map: FATAL BUG = red/orange tint, CODE SMELL = amber tint, OPTIMIZATION = green tint. Render a bordered white card with small shadow: top row = black index chip (zero-padded 2-digit number), "LINE {issue.line}" bold, severity badge (colored per map), issue.title right-aligned muted small text (wraps, never clips); a <pre><code> code block with a left accent border stripe showing issue.codeSnippet; two lines "✕ Diagnosis: {issue.diagnosis}" (accent, bold ✕) and "✓ Expected: {issue.expected}" (green, bold ✓).

=== 17. components/FixedCode.tsx ("use client") ===
Props { sectionNumber, language, code, onApply }. State copyStatus: "idle"|"copied"|"failed". handleCopy: try navigator.clipboard.writeText, set "copied", catch sets "failed", reset to "idle" after 2s either way. Copy button label by status: "📋 COPY FIXED CODE" / "✓ COPIED TO CLIPBOARD" / "✕ COPY BLOCKED, SELECT MANUALLY". Render <SectionHeader number={sectionNumber} title="Fix"> with a bold green "CORRECTED CODE" label; a bordered code block with a header strip showing fake filename `solution.{extension}` (from LANGUAGES, default "txt") left and bold green "READY TO APPLY" right, then a <pre><code> of the code; two buttons below — the copy button and "APPLY TO EDITOR ↵" (calls onApply(code)), both bordered with hover-invert-to-dark style.

=== 18. components/RoastReport.tsx ===
Props { state, roastLevel, language, result, errorMsg, onRetry, onApplyFix }. A bordered panel (~46-45% width desktop) with a 40px header strip ("AUDIT // REPORT" left, "Roast Report" bold centered, empty spacer right). Below: render exactly one of <EmptyState/>, <LoadingState/>, <ErrorState error={errorMsg} onRetry={onRetry}/>, or (state "results" && result) a scrollable padded results view with:
- Section 1 "Roast": SectionHeader + "STYLE: {roastLevel}" meta, then the roast text as a styled quote `"{result.roast}"`.
- Section 2 "What's Wrong": SectionHeader + issue count meta (singular/plural "N ISSUE"/"N ISSUES"); if 0 issues show green "No issues found. Suspiciously clean."; else map to <IssueCard>.
- Section 3 "Fix": <FixedCode sectionNumber={3} .../> ONLY when result.correctedCode is non-empty.
- Takeaway section: ONLY when result.takeaway is non-empty, with section number computed as `result.correctedCode ? 4 : 3` (must shift dynamically so numbering is never skipped or gapped) — render result.takeaway as body text.

=== 19. components/EmptyState.tsx, LoadingState.tsx, ErrorState.tsx ===
Each a centered flex-1 column with generous padding, consistent with the blueprint theme:
- EmptyState (no props): 64x64 dashed-border square with "{}" large muted, bold uppercase heading "Awaiting Code Submission", muted body: `Paste your code on the left, then click "ROAST MY CODE" or press Ctrl+Enter (⌘+Enter on Mac).`
- LoadingState (no props): 64x64 solid-border square with a spinning "/" , pulsing bold uppercase heading "Analyzing Code", muted body: "Evaluating computational complexity and architectural purity..."
- ErrorState ({ error, onRetry }): 64x64 accent-bordered square with accent "!", bold accent uppercase heading "Analysis Failed", the error message (or fallback "An unknown error occurred during code evaluation.") in muted text, and a "RETRY ANALYSIS ↵" button calling onRetry.

=== 20. components/StatusBar.tsx ===
Props { isRoasting }. Footer bar (top border, light background, small muted mono): left = "STATUS: {amber PROCESSING... if isRoasting, else green ONLINE ({AI.modelLabel uppercased})}" then (hidden on mobile) "| ENGINE: GOOGLE GEMINI"; right = "{APP.name} {APP.version} // LIVE" uppercase tracked-out. Below the main status row, add a second, smaller centered line, muted and understated (not competing visually with the status row above it), reading exactly: "Made at GDG Nashik Pre-DevFest Workshop" — this must render on every screen, in every state (empty/loading/results/error), since StatusBar is always mounted.

=== Final checks ===
Wire everything into Workspace.tsx with the exact props described above. Confirm: Ctrl+Enter/Cmd+Enter roasts from anywhere on the page; the issue line-highlight disappears the instant code is edited away from roastedCode; "Apply to Editor" loads corrected code without auto-re-roasting; the app builds and runs with `npm run dev` with no TypeScript or console errors; the "Made at GDG Nashik Pre-DevFest Workshop" line is visible in the footer at all times. Do not add any feature not described above.
```

---

**After this finishes:** create `.env.local` yourself (don't let the agent do it), paste your real Gemini key, run `npm run check:api`, then `npm run dev` and test at http://localhost:3000.
