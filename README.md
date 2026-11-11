# Strudel Section Composer (skill)

A Claude skill for composing music in [Strudel](https://strudel.cc), the browser live-coding environment. You describe the music and react to it, Claude writes the code, and you listen in a small web page that plays it.

[ Example Composition ](https://claude.ai/artifact/G4obMGMiVb28EGt1F5xSue)

## What it's for

Building a song section by section (intro, verses, chorus, bridge, outro) and hearing it as you go. Claude can't hear, so the loop relies on you: Claude writes and checks the code, you listen and give notes in plain words ("too busy", "lead never kicks in", "make the chorus brighter"), and Claude changes only what you asked.

## How it's used

1. **Ask for music.** Describe a beat, loop or song, or paste a line you already have. Claude builds around your material.
2. **Open the Strudel Section Composer page.** Claude publishes it as an artifact. Each section is its own card with editable code and a bar count.
3. **Listen.**
   - **Play all** runs the whole arrangement and loops back to the start.
   - **Play from here** starts at any section and carries on through the rest.
   - **Loop this** repeats one section while you work on it.
4. **Reply in chat** with what you hear, or edit the code in a card and paste it back. Claude updates the same page.
5. **Take it to Strudel.** **Export for strudel.cc** copies the whole arrangement as plain Strudel code, and **Open in strudel.cc** opens it there. Use strudel.cc when you want drum samples, which the page can't load.

## How a song is structured

A song is a shared block (tempo and parts every section can use) plus an ordered list of sections. Each section has a name, a bar count and a pattern. A section can also repeat an earlier one, so editing it once changes both.

## Good to know

- Sounds are synths only on the page: sawtooth, square, triangle, sine, noise, and a synth kick, snare and hats. Sample banks don't load there.
- Effects that need audio worklets (`crush`, `distort`, `phaser` and similar) may play silent. Claude avoids them by default.
- Claude checks the pattern with a quick headless test before publishing, but it can't judge how the music sounds. That part is yours.

## Files

- `SKILL.md`: the instructions Claude follows, including the page template, the checking scripts and the pitfalls learned along the way.
