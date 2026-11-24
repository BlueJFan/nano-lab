# Nano Lab — Arduino Nano AVR Emulator, Verifier & Grader

Client-side React + TypeScript + Tailwind app for the 4CCS1CS1 CS Arduino coursework
(ATmega328P @ 16 MHz, 8 LEDs: bit 7..4 = PD7..PD4 / D7..D4, bit 3..0 = PB3..PB0 / D11..D8).

## Run

```bash
npm install
npm run dev        # http://localhost:5173
npm test           # emulator, verifier, input & export tests (node, via tsx)
npm run build      # static site in dist/ (works from any web server)
```

`dist-standalone/nano-lab.html` is a prebuilt single file — open it straight from disk.

## Files

| File | What it does |
|---|---|
| `src/avrAssembler.ts` | 2-pass GNU avr-as compatible assembler, ATmega328P constants, rich `{line, severity, message, howToFix}` diagnostics, Intel HEX |
| `src/avrCpu.ts` | Cycle-accurate core, exact SREG maths, Timer1 (Normal/CTC, TEMP register, OCF1A/B, TOV1, interrupts), coalesced LED log, runtime linter |
| `src/testBench.ts` | Expected-vs-actual decoder for T1.1–T2.2, wait-time tables, headless verifier, static checks, 100-mark rubric, delay/Timer1 calculators |
| `src/BreadboardSVG.tsx` | Nano + breadboard + 8 LEDs/resistors/jumpers in SVG, 8-channel logic analyzer |
| `src/exportTools.ts` | KEATS packager, avr-gcc/avrdude commands, Wokwi diagram.json, AI prompt builders, BYOK AI client |
| `src/App.tsx` | The UI |

## Emulation notes

* GNU semantics: labels are **byte** addresses, `.org` is a byte address (Timer1 COMPA vector = `.org 0x2C`),
  numbers with a leading 0 are octal. Built-in register names use I/O addresses (`PORTB = 0x05`), so
  `out PORTB, r16` works; Timer1 registers are extended I/O and need `lds`/`sts`.
* A file with `main:` and no `.org` gets avr-gcc's start-up code (vector table, SP = RAMEND, r1 = 0, `call main`);
  ISRs are bound by name (`TIMER1_COMPA_vect` / `__vector_11`) exactly like avr-gcc.
* Software delay loops and polling/idle loops are fast-forwarded **bit-exactly** (n − 1 identical iterations are
  skipped, the last runs for real, never past a timer event). The tests compare every LED event cycle against
  plain stepping. This is how all 50 Morse loops (~6–10 min of simulated time) verify in well under a second.
  The 1.5 billion-cycle safety cap applies to cycles actually stepped.
* `PORTB`/`PORTD` writes within 10 cycles are coalesced into one LED change.

* Live runs stop on their own at the **Auto-stop** limit (1–60 simulated minutes, default 10) so a runaway loop
  cannot grow the LED log forever; each frame has an 8 ms budget. **Reset** kills the run loop, cancels a
  running verification and power-on resets the CPU.

## Inputs

Any name works: accents are folded (Zoë → ZOE), hyphens/apostrophes ignored, palindromes adjusted from the rest of
the name (ADA → ADL). The Morse field accepts 1–12 letters (3 is the coursework rule; other lengths are graded as
practice). **Ada Lovelace test** fills in a fictional sample student in one click.

## Export

* **avr-gcc / Wokwi (.S)** — adds `.equ` for register names the emulator knows built in, `#define` for `.def`
  aliases and `ZL`/`ZH`…, maps `TIMER1_COMPA_vect` to `__vector_11`, adds `.global main`.
* **Microchip Studio (.asm)** — AVRASM2 syntax: `.include "m328pdef.inc"`, word-addressed `.org`, `.db/.dw`,
  `low/high`, `low(2*label)` for LPM tables, renamed numeric local labels, and a reset vector + start-up code when
  the program relies on avr-gcc's `main`.
* **Debugging / Rubric-grading prompts** — your code, inputs, target Morse string and (after Verify) the measured
  timings, ready to paste into any AI chat.

## Marks

Correctness items are measured by the simulator (SIM). Comment / method / code-quality items start as static
heuristics (HEUR) and can be judged by an AI provider (AI). It is a practice estimate, not an official mark.

## AI providers

Gemini (default), OpenAI, Anthropic, any OpenAI-compatible endpoint (e.g. OpenRouter). Keys stay in
`localStorage` and requests go directly from the browser to the provider. Model IDs are editable.
The **Debugging prompt** and **Rubric-grading prompt** buttons bundle everything for a web chat instead.
