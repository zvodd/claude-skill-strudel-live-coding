---
name: "strudel-live-coding"
description: "Compose music in Strudel section by section and hand the user a playable \"Strudel Section Composer\" artifact. Use when the user wants a beat, loop, track or song in Strudel, or wants to iterate on one by listening and giving notes."
---

# Strudel section composer

Claude cannot hear. The loop is: **write, check headless (seconds), publish or update the composer artifact, the user listens and replies in chat, change only what they said.** The user's ears are the audio test. Do not add browser-based audio checks: they are slow and the feedback loop already covers them.

## The song model
Everything lives in `song.json`:
```json
{ "shared": "setcps(100/60/4)\nconst bass = ...\nconst arp = g => ...",
  "sections": [
    {"name": "Intro", "bars": 4, "code": "return stack(bass, arp(.4))"},
    {"name": "Verse B (repeat)", "bars": 8, "ref": "Verse B"} ] }
```
- Section order is the arrangement. Play all runs them in order and loops back to the first. Play from here starts at that section and wraps round.
- `shared` holds `setcps(bpm/60/4)` and parts reused across sections. Make reusable parts **functions with a gain argument** (`const arp = g => note(...).gain(g)`), because calling `.gain()` a second time on an already-gained pattern is unreliable.
- A section's `code` is a function body that ends with `return <pattern>`. Define section-local parts as `const`s above the return.
- `ref` makes a section repeat an earlier one: edit it once and both change. Use it for repeated verses.
- Keep the user's own lines intact and build around them.

## Verified environment facts
- strudel.cc is NOT reachable from the workspace shell or WebFetch (403). npm works.
- Artifacts can run Web Audio (playback must start from a click) and load `https://unpkg.com/@strudel/web@1.3.0/dist/index.js` (global `strudel`). Iframing strudel.cc is refused. Fetches to other hosts are blocked, so **sample banks (`bd`, `sd`, `hh`) do not load in the artifact**; make drums from synths (below). On strudel.cc itself, samples work.
- The workspace has no audio. Never say you "heard" anything.

## Rules that bite
**Effects**
- Worklet effects (`crush`, `coarse`, `shape`, `distort`, `phaser`, `tremolo`, `supersaw`) played silent notes in the artifact: the user said the lead "never kicks in" when `.crush(7)` was on it. The template calls `strudel.initAudio()` before the first evaluate, which loads worklets in plain Chrome, but that is **unverified inside the artifact**. Default to avoiding them. If the user wants real bitcrush, try it and ask them to listen.
- Safe: `lpf`, `hpf`, `lpq`, `fm`, `vib`, `detune`, `delay`/`delaytime`/`delayfeedback`, `room`, `attack`/`release`/`decay`/`sustain`, `gain`, `pan`, `penv`/`pdecay`, `orbit`. Grit without worklets: `.s("square").lpf(2200).lpq(9).fm(1).detune(.15)`.
- Synth drums (no samples):
  - kick: `s("sine").note("c2").struct("x ~ x ~").penv(30).pdecay(.05).decay(.2).sustain(0).gain(.5)`
  - snare: `s("white").struct("~ x ~ x").hpf(1800).decay(.12).sustain(0).gain(.35)`
  - hats: `s("white").struct("x*8").hpf(7000).decay(.04).sustain(0).gain(.1)`
  - water or crash: `s("pink").hpf(2500).gain(sine.range(0,.12).slow(2))`, or `s("white")...decay(1.5)` for a crash
**Patterns**
- `n("...").fast(2).scale(x)` **doubles every note**. Write all the steps out (16 steps for a 16th-note bar) or put `.scale()` first, but then a `<chord>` pattern inside the scale also speeds up.
- `.fast(n)` also speeds up any `<a b>` inside it. For per-bar changes write the steps explicitly.
- Per-bar content: `note("<[bar1] [bar2]!4 [bar3]>")`, one entry per bar, `!n` repeats. A held note is a single bracket entry, `[eb6]`.
- `arrange([bars, pat], ...)` starts each section at its own cycle 0, so `<...>` and signals count from the start of the section. `hush` resets the clock, so Play all starts at the first section.
- Layering by bar: `.mask("<0 0 1 1 1 1 1 1>")`. Crescendo or fade: `gain(saw.range(a,b).slow(bars))`, filter opens with `lpf(saw.range(600,4500).slow(8))`, fade out `saw.range(a,0).slow(8)`.
- A voice a diatonic third below another: write it out by hand in the scale, do not shift by semitones.
- Headless eval only: `globalThis.setcps = () => {}` (it only exists in the browser).

## Composing
- Keep recurring elements (a bass line, a rhythm, a motif) across sections so the song hangs together. Later sections add, swap or strip layers around them instead of replacing them.
- Try call and response: one section states an idea and another answers it, or the two sit together later on.
- Think in structure, loosely: contrast sections (sparse and dense, tense and open), repeat with small changes, build to a peak, and let the ending echo the opening. Save heavy rhythm for peaks.
- Start from what the user asks for and the material they give, never from a previous song. When they react in plain words, map each note to one lever and say which.

## Check (headless, seconds)
```bash
mkdir -p /tmp/st && cd /tmp/st && npm init -y >/dev/null
npm i @strudel/core @strudel/mini @strudel/tonal @strudel/transpiler
# packaging bug: @kabelsalat/web's main lacks the SalatRepl export
sed -i 's#"main": "dist/index.js"#"main": "dist/index.mjs"#' node_modules/@kabelsalat/web/package.json
```
```js
// check-song.mjs song.json : builds the same program the page does, prints sounds present mid-section
import fs from 'fs';
import { evalScope, evaluate } from '@strudel/core';
import { transpiler } from '@strudel/transpiler';
globalThis.setcps = () => {};
await evalScope(import('@strudel/core'), import('@strudel/mini'), import('@strudel/tonal'));
const S = JSON.parse(fs.readFileSync(process.argv[2], 'utf8'));
let p = S.shared + '\n';
S.sections.forEach((s, i) => { p += s.ref ? `const __s${i} = __s${S.sections.findIndex(x => x.name === s.ref)};\n` : `const __s${i} = (() => {\n${s.code}\n})();\n`; });
p += 'arrange(' + S.sections.map((s, i) => `[${s.bars}, __s${i}]`).join(',') + ')';
const { pattern } = await evaluate(p, transpiler);
let c = 0;
for (const s of S.sections) {
  const h = pattern.queryArc(c + s.bars / 2, c + s.bars / 2 + 1);
  console.log(String(c).padStart(3), s.name.padEnd(28), [...new Set(h.map(x => x.value.s))].sort().join(','));
  c += s.bars;
}
```
Ignore the "cannot use window" warning. A thrown error means a code mistake. To check a melody, query one bar (`pattern.queryArc(c, c+1).filter(x => x.hasOnset())`) and print `x.value.note` per voice (tell voices apart by `pan` or `gain`). That is how a doubled-note bug and wrong chord tones were caught. Check that the repeat plays the same as the original and that the total length matches the intent (bars x 2.4 s at 100 bpm).

## Build and publish
1. Edit `song.json`, run the check.
2. Generate the page: replace `/*SONG*/` in the template below with `json.dumps(song, indent=1).replace('</', '<\\/')` (a `</` inside the JSON would close the script tag), write `index.html`.
3. Publish with `publish_files` (title "Strudel Section Composer", icon "music"). First publish creates it, later ones reuse the **same slug** with the latest `base_version` and mode patch. If the artifact is shared or public, call `get_artifact` first and pass `shared_live` and `artifact_title` too, otherwise the call is refused.
4. Reply in a few lines: what changed, what to listen for, one lever if it is not right. Do not repeat the link.

## Export to strudel.cc
The **Export for strudel.cc** button copies plain Strudel that runs as is: shared code, each section as `const secName = (() => { ... })();` (a repeat is `const secRepeat = secOriginal;`), then `$: arrange([bars, secName], ...)`. This was checked to play exactly the same events as the page. If the clipboard API is blocked the button falls back to a temporary-textarea copy, and **Open in strudel.cc** still works. **Open in strudel.cc** is a plain link to `https://strudel.cc/#<hash>` where hash is `encodeURIComponent(btoa(utf8 bytes))`, the library's own `code2hash` (round-trip checked). That strudel.cc reads the hash on load is from memory. Never export loose `return`s: that is why sections are wrapped.

For drum samples or the full Strudel UI the user opens that link (or pastes the export) on strudel.cc, where Ctrl+Enter plays and Ctrl+. stops. Only drive strudel.cc in the built-in browser if the user asks. To keep a finished track as a file, save the export text as a `.js` file and send it.

## Palette
`#fbf5ec` cream (page), `#dac7a5` sand (borders, secondary buttons), `#dba039` amber (primary, live section), `#9f8360` taupe (button borders, fine lines; too light for small text). Derived for text: `#3a2e1f` ink, `#6e5a3e` muted. Cards `#fffdf8`. Status `#3f6b2a` ok, `#a3321f` error. Light theme only.

## Template (index.html with `/*SONG*/` placeholder)
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Strudel Section Composer</title>
<style>
  :root { --sand:#dac7a5; --amber:#dba039; --cream:#fbf5ec; --taupe:#9f8360; --ink:#3a2e1f; --muted:#6e5a3e; }
  body { font-family: system-ui, sans-serif; max-width: 720px; margin: 1.5rem auto; padding: 0 1rem; background: var(--cream); color: var(--ink); }
  h1 { font-size: 1.3rem; margin: 0 0 .25rem; }
  .sub { font-size: 13px; color: var(--muted); margin: 0 0 1rem; }
  .card { border: 1px solid var(--sand); background: #fffdf8; border-radius: 8px; padding: .75rem; margin-bottom: .75rem; }
  .card.live { border-color: var(--amber); box-shadow: 0 0 0 3px #dba03955; }
  .head { display: flex; align-items: center; gap: .5rem; flex-wrap: wrap; margin-bottom: .5rem; }
  .head b { flex: 1; min-width: 6rem; }
  .pos { font-size: 12px; color: var(--muted); min-width: 5rem; text-align: right; }
  textarea { width: 100%; box-sizing: border-box; font: 13px/1.4 ui-monospace, monospace; padding: .5rem; min-height: 7rem; resize: vertical; background: var(--cream); color: var(--ink); border: 1px solid var(--sand); border-radius: 6px; }
  button { font: inherit; padding: .35rem .8rem; cursor: pointer; background: var(--sand); color: var(--ink); border: 1px solid var(--taupe); border-radius: 6px; }
  button:hover { background: #d1bb92; }
  a { color: var(--muted); font-size: 13px; }
  button.on, button.primary { background: var(--amber); border-color: #b9822a; font-weight: 600; }
  input[type=number] { width: 3.5rem; font: inherit; background: var(--cream); color: var(--ink); border: 1px solid var(--taupe); border-radius: 4px; }
  label { font-size: 13px; color: var(--muted); }
  #status { font-size: 13px; margin: .5rem 0; white-space: pre-wrap; }
  #log { font: 12px ui-monospace, monospace; color: var(--muted); white-space: pre-wrap; margin-top: .5rem; }
  .ok { color: #3f6b2a; } .err { color: #a3321f; }
</style>
</head>
<body>
<h1>Strudel Section Composer</h1>
<p class="sub">A song built from sections. Each card is its own player: Play from here runs on into the next section and loops back round. Export gives you the whole arrangement as plain Strudel code. Synths only here (no sample banks).</p>
<div class="head">
  <button id="toggle" class="primary">Play all</button>
  <button id="copy">Export for strudel.cc</button>
  <a id="open" href="#" target="_blank" rel="noopener">Open in strudel.cc &#8599;</a>
</div>
<div id="status">Loading Strudel…</div>
<div class="card"><div class="head"><b>Shared (tempo and parts every section can use)</b></div><textarea id="shared"></textarea></div>
<div id="secs"></div>
<div id="log"></div>
<script type="application/json" id="song">/*SONG*/</script>
<script src="https://unpkg.com/@strudel/web@1.3.0/dist/index.js"></script>
<script>
const SONG = JSON.parse(document.getElementById('song').textContent);
const $ = (id) => document.getElementById(id);
const statusEl = $('status');
const say = (m, c) => { statusEl.textContent = m; statusEl.className = c || ''; };
let ready = null, repl = null, session = null, audioReady = false;

// ---- build the UI
$('shared').value = SONG.shared;
const secsEl = $('secs');
SONG.sections.forEach((s, i) => {
  const d = document.createElement('div');
  d.className = 'card'; d.id = 'card' + i;
  d.innerHTML = '<div class="head"><b></b><span class="pos" id="pos' + i + '"></span>' +
    '<label>bars <input type="number" min="1" id="bars' + i + '"></label>' +
    '<button class="primary" data-play="' + i + '">Play from here</button>' +
    '<button data-loop="' + i + '">Loop this</button></div>' +
    (s.ref ? '<div class="sub">Repeats ' + s.ref + ' (edit that section to change both)</div>' : '<textarea id="code' + i + '"></textarea>');
  d.querySelector('b').textContent = s.name;
  secsEl.appendChild(d);
  $('bars' + i).value = s.bars;
  if (!s.ref) $('code' + i).value = s.code;
});
const fit = (el) => { el.style.height = 'auto'; el.style.height = (el.scrollHeight + 4) + 'px'; };
document.querySelectorAll('textarea').forEach(el => { fit(el); el.addEventListener('input', () => fit(el)); });

const read = () => ({
  shared: $('shared').value,
  sections: SONG.sections.map((s, i) => ({ name: s.name, ref: s.ref, bars: Math.max(1, +$('bars' + i).value || 1), code: s.ref ? '' : $('code' + i).value })),
});

function program(data, order) {
  let p = data.shared + '\n';
  data.sections.forEach((s, i) => {
    if (s.ref) { p += 'const __s' + i + ' = __s' + data.sections.findIndex(x => x.name === s.ref) + ';\n'; }
    else p += 'const __s' + i + ' = (() => {\n' + s.code + '\n})();\n';
  });
  return p + 'arrange(' + order.map(i => '[' + data.sections[i].bars + ', __s' + i + ']').join(', ') + ')';
}

async function play(start, loopOne) {
  try {
    if (typeof strudel === 'undefined') throw new Error('Strudel library did not load');
    if (!ready) ready = strudel.initStrudel({ onEvalError: (e) => say('Code error: ' + (e && e.message || e), 'err') });
    repl = await ready; window.__repl = repl;
    // loads the audio worklets (crush, distort...). Harmless if it fails: those effects then stay silent.
    if (!audioReady) { audioReady = true; try { await strudel.initAudio(); } catch (e) { console.log('initAudio failed: ' + e.message); } }
    const data = read(), n = data.sections.length;
    const order = loopOne ? [start] : data.sections.map((_, k) => (start + k) % n);
    await strudel.evaluate(program(data, order));
    session = { order, bars: order.map(i => data.sections[i].bars), loopOne, start };
    document.querySelectorAll('button[data-loop]').forEach(b => b.classList.toggle('on', loopOne && +b.dataset.loop === start));
    say('Playing' + (loopOne ? ' (looping ' + data.sections[start].name + ')' : ' from ' + data.sections[start].name), 'ok');
  } catch (e) { say('Error: ' + e.message, 'err'); }
}

secsEl.addEventListener('click', (e) => {
  const b = e.target.closest('button'); if (!b) return;
  if (b.dataset.play !== undefined) play(+b.dataset.play, false);
  if (b.dataset.loop !== undefined) play(+b.dataset.loop, true);
});
const toggle = $('toggle');
toggle.addEventListener('click', () => {
  if (session) { try { strudel.hush(); session = null; say('Stopped.'); } catch (e) { say('Error: ' + e.message, 'err'); } }
  else play(0, false);
});
// ---- export: plain Strudel that runs as-is in strudel.cc (no loose returns)
const ident = (name, i, used) => {
  let w = name.replace(/[^A-Za-z0-9]+/g, ' ').trim().split(' ').map(x => x[0].toUpperCase() + x.slice(1)).join('');
  let id = 'sec' + (w || i); while (used.has(id)) id += i; used.add(id); return id;
};
function exportCode() {
  const d = read(), used = new Set(), ids = d.sections.map((s, i) => ident(s.name, i, used));
  let out = '// ' + document.title + '. Paste into strudel.cc and press Ctrl+Enter.\n' + d.shared + '\n';
  d.sections.forEach((s, i) => {
    out += '\n// ' + s.name + ' (' + s.bars + ' bars)\n';
    if (s.ref) out += 'const ' + ids[i] + ' = ' + ids[d.sections.findIndex(x => x.name === s.ref)] + ';\n';
    else out += 'const ' + ids[i] + ' = (() => {\n' + s.code.split('\n').map(l => '  ' + l).join('\n') + '\n})();\n';
  });
  return out + '\n$: arrange(\n' + d.sections.map((s, i) => '  [' + s.bars + ', ' + ids[i] + ']').join(',\n') + '\n)\n';
}
const hash = (code) => { let bin = ''; new TextEncoder().encode(code).forEach(b => { bin += String.fromCharCode(b); }); return encodeURIComponent(btoa(bin)); };
function refreshExport() { const c = exportCode(); $('open').href = 'https://strudel.cc/#' + hash(c); return c; }
document.querySelectorAll('textarea, input').forEach(el => el.addEventListener('input', refreshExport));
refreshExport();
$('copy').addEventListener('click', async () => {
  const c = refreshExport();
  try { await navigator.clipboard.writeText(c); say('Exported to your clipboard. Paste it into strudel.cc.', 'ok'); }
  catch (e) {
    // fallback: copy through a temporary textarea
    const ta = document.createElement('textarea'); ta.value = c; ta.style.position = 'fixed'; ta.style.opacity = '0';
    document.body.appendChild(ta); ta.select();
    let ok = false; try { ok = document.execCommand('copy'); } catch (e2) {}
    ta.remove();
    say(ok ? 'Exported to your clipboard. Paste it into strudel.cc.' : 'Copy was blocked here. Use "Open in strudel.cc" instead.', ok ? 'ok' : 'err');
  }
});

function tick() {
  toggle.textContent = session ? 'Stop' : 'Play all';
  document.querySelectorAll('.card').forEach(c => c.classList.remove('live'));
  SONG.sections.forEach((_, i) => { $('pos' + i).textContent = ''; });
  if (session && repl) {
    let now = 0; try { now = repl.scheduler.now(); } catch (e) {}
    const total = session.bars.reduce((a, b) => a + b, 0);
    let p = ((now % total) + total) % total;
    for (let k = 0; k < session.order.length; k++) {
      if (p < session.bars[k]) {
        const i = session.order[k];
        $('card' + i).classList.add('live');
        $('pos' + i).textContent = 'bar ' + (Math.floor(p) + 1) + '/' + session.bars[k];
        break;
      }
      p -= session.bars[k];
    }
  }
  requestAnimationFrame(tick);
}
tick();

// ---- log + load check
const logEl = $('log'); const lines = [];
document.addEventListener('strudel.log', (e) => {
  lines.push((e.detail && e.detail.message) || ''); while (lines.length > 4) lines.shift();
  logEl.textContent = lines.join('\n');
});
window.addEventListener('load', () => {
  typeof strudel !== 'undefined' ? say('Ready. Press Play from here on any section.', 'ok') : say('Strudel library did NOT load.', 'err');
});
</script>
</body>
</html>
```