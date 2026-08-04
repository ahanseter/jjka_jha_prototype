# jjka_jha_prototype
A working prototype of the forthcoming JHA feature set for Safety Management Suite, including Auth &amp; Saved Records

## Live demo

Hosted via GitHub Pages: `https://ahanseter.github.io/jjka_jha_prototype/` (enable under repo **Settings → Pages** if not live yet).

Behind a lightweight passcode gate (see `index.html`, search `GATE_CODE`) — this is a casual deterrent for a public link, not real security. Change the code any time by editing that one line.

## What this is / isn't

- Single static HTML file, no build step, no server, no shared database.
- Data persistence uses the browser's File System Access API + IndexedDB — each visitor picks their own local folder; nothing is shared between viewers or devices.
- Intended purely as a clickable design/functional reference for stakeholders.
