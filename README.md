# jjka_jha_prototype
A working prototype of the forthcoming JHA feature set for Safety Management Suite, including Auth &amp; Saved Records

## Live demo

Hosted via GitHub Pages: `https://ahanseter.github.io/jjka_jha_prototype/` (enable under repo **Settings → Pages** if not live yet).

Behind a lightweight passcode gate (see `index.html`, search `GATE_CODE`) — this is a casual deterrent for a public link, not real security. Change the code any time by editing that one line.

## What this is / isn't

- Single static HTML file, no build step, no server, no shared database.
- Data persistence uses the browser's File System Access API + IndexedDB — each visitor picks their own local folder; nothing is shared between viewers or devices.
- Intended purely as a clickable design/functional reference for stakeholders.

## How this was deployed

1. Clone this repo.
2. Copy the prototype's single HTML file in as `index.html` at repo root.
3. Add the passcode gate overlay (a `<div>` + inline `<script>` block right after `<body>` — search `protoGate` / `GATE_CODE` in `index.html`).
4. Commit and push to `main`.
5. Enable GitHub Pages: repo **Settings → Pages** → Source: **Deploy from a branch** → Branch: `main`, folder `/ (root)` → Save.
6. Pages builds in a minute or two; the site is live at `https://<owner>.github.io/<repo>/`.

No build step, no CI, no secrets involved — steps 1–4 are plain `git`, step 5 is a one-time UI toggle.
