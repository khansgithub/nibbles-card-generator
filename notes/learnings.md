# Development Notes

Learnings from working on this project (agent-browser quirks, tooling gotchas, and app architecture decisions).

## agent-browser (CLI browser automation)

This tool is flaky on this Windows machine. Known issues and workarounds:

- **Daemon drops the CDP link after `upload`.** The `upload` command itself reports `✓ Done`, but every subsequent command (`eval`, `snapshot`, etc.) fails with `os error 10060` (TCP connect timeout). Retrying does not help.
- **`open` + `eval` work reliably** in a fresh session. The breakage is specifically triggered by file upload.
- **Workaround cycle:** `agent-browser close --all` then `agent-browser open <url>` re-establishes the connection. Use a distinct session to avoid conflicts: `agent-browser --session test open <url>`.
- **Daemon restart:** if commands keep failing, kill the daemon process (`Get-Process | Where ProcessName -match 'agent-browser'`, then `Stop-Process`) and it auto-restarts on next command. `agent-browser session list` is a quick health check; `--version` works even when the daemon is down.
- **`--session-name <name>` persists cookies + localStorage across sessions.** Workaround for the upload problem: open with `--session-name`, upload, then `close --all` + reopen with the same name to restore state without re-uploading.
- **Do not pass large payloads as eval args.** On Windows a 34 KB base64 JSON blob caused `The filename or extension is too long`. Set state via small evals or chunked appends instead.
- The tool writes a `.ps1` shim at `C:\nvm4w\nodejs\agent-browser.ps1`; `Start-Process agent-browser` fails with "not a valid Win32 application" — invoke via the CLI directly.

## Dev server on Windows

- Start it hidden with logging: `Start-Process cmd "/c npm run dev > dev-server.log 2>&1" -WindowStyle Hidden`.
- Vite picks 5174 when 5173 is busy; use `npm run dev -- --port 5173 --strictPort` to pin the port.
- Verify with curl: `curl.exe -s -o NUL -w "HTTP %{http_code} time %{time_total}s" http://localhost:5173/`. Note: a dead port gives a slow `HTTP 000` timeout, not an instant refusal.

## Build tooling

- **`?raw` imports cannot resolve from dot-directories** (`.prompts/`). The image-gen template lives at `src/prompts/image-gen-template.txt`; the generated output goes to `.prompts/image-gen-prompt.txt`.
- Commands: `npm run build` = `tsc -b && vite build`; `npm run lint` = oxlint; `npm run build:image-prompts` = `tsx scripts/build-image-gen-prompt.ts`. Only pre-existing lint warning: `src/components/Slide1Content.tsx:38:27` jsx-key.

## gh-pages deploy

- `master` is local-only; only `gh-pages` is pushed. The orphan `gh-pages` branch tracks `.gitignore`, so `git rm -rf .` removes it and `node_modules` would then be committed by `git add -A`. The README flow restores it with `git checkout HEAD -- .gitignore` after copying `dist\*`.

## App architecture decisions

- **Text editing:** edits are keyed `${keyword}|${slide}|${cls}|${idx}` and applied via a `useLayoutEffect` (no deps, runs every render) to the visible card and the hidden download cards (`allCardsRef`). Newlines render as `<br>`; `**…**` = yellow highlight span, `~~…~~` = blue.
- **Drag-to-reposition:** replaces X/Y sliders. A `pointerdown` on the card (delegated from the `.preview` wrapper via `closest(TEXT_SELECTOR)`) starts a drag; the preview is scaled with CSS `zoom`, so deltas are divided by `card.getBoundingClientRect().width / 1080`. `pointermove`/`pointerup` listen on `window` and mutate a ref imperatively (no React re-render during drag); the final offset is committed to `textOffsets` state on pointer-up so the same transform applies to the hidden download cards and exports.
- **Download:** captures a clone of the card (zoom reset to 1, off-screen) with html2canvas at 1080x1350.

## Recurring workflow

- After each feature: `npm run lint` then `npm run build`, then commit (user asks to commit per feature).
- For data upload during testing, `data/data.json` (10 keywords) is injected into `localStorage["cardgen:data"]`.