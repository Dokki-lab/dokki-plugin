---
name: ad-film-production
description: Take an ad film from a one-line brief to a shot table that can actually be generated — single-minded proposition, a hook that lands in three seconds, shot-by-shot prompts written in the form video models accept, and the continuity discipline (reference sheet, last-frame-to-first-frame) that stops the product changing shape between cuts. Use when someone asks for a commercial, a product film, a 15/30/60-second spot, or a storyboard for AI video generation.
license: Proprietary
metadata:
  author: Dokki
  version: "1.0.0"
  protocol: dokki-ad-film-production@1
---

# Ad film production

An ad is not a video with a logo at the end. It is **one idea, delivered before
the viewer leaves.** Everything below serves that.

Work in this order. Do not skip to prompts — a beautiful shot of the wrong idea
is a wasted generation, and generations cost real money and minutes.

## 1. Force the brief down to one sentence

Before any shot exists, get these four. Ask for what is missing; ask once, in
one message, not as a form.

- **The one thing** — the single-minded proposition. If the answer has an "and"
  in it, it is two ads. Make them choose.
- **Who** — not a demographic, a person in a situation.
- **Length** — 15 / 30 / 60. This decides the structure, so it is not optional.
- **What they should do** — the call to action.

Write these into the project's `BRIEF.md`. Every later argument gets settled by
reading it.

## 2. Lay the arc for that length

The first three seconds decide whether the rest exists. Open on the most
arresting thing you have — never on a logo, never on a slow establishing drift.

- **15–30s:** hook → problem → benefit → one proof → call to action
- **30–60s:** hook → problem → benefit → two proofs or a testimonial → call to
  action → brand close

A 30-second script is about **70–85 words** of voiceover. If the script is
longer, the film is longer; do not "read it faster".

Give each shot **one job** from the arc. A shot that carries two beats carries
neither.

## 3. Write each shot in the form the model accepts

Six parts, in this order:

> **Subject → Action → Environment → Camera → Style → Constraints**

Rules that change the output more than anything else:

- **One clear action per shot.** "A woman sprints across a wet field", not
  "a woman runs and then looks back and smiles".
- **Be specific about the verb.** `sprints` beats `runs`; `flaps wildly` beats
  `flaps`. Adverbs do real work here.
- **Name the camera in camera words** — size (wide / medium / close-up), angle
  (eye level / low angle / overhead), movement (static locked-off / slow dolly
  in / handheld / orbit). "Cinematic" is not a camera move.
- **Write what SHOULD happen, never what should not.** Seedance has no negative
  prompt; "no text on screen" tends to produce text on screen. Describe the
  clean frame you want instead.
- **Order multi-step action in time.** The model follows the sentence order.

Full vocabulary: [`references/shot-grammar.md`](references/shot-grammar.md).

## 4. Hold continuity — this is where ad films actually fail

Generation is the easy part now. **Continuity is the hard part**: each shot is
born independently, so the jacket changes colour, the light jumps, the bottle
changes shape between cuts. Treat drift as a defect you prevent, not weather you
suffer.

Two disciplines, both supported by the shot table:

**A reference sheet, carried everywhere.** Before generating any motion, make
the stills that lock the look: the product from the angles you will use, the
character in three-quarter and profile, the palette. Every shot containing that
subject carries the same reference images. This is what keeps a face or a
package recognisable across cuts.

**Last frame becomes the next first frame.** For any two shots meant to read as
continuous, export the last frame of shot N and hand it to shot N+1 as its first
frame. Two shots joined this way cut together; two shots generated independently do not.

## 5. Know what the models can and cannot do

This decides which shots are even possible, and what they cost. Verified against
the live gateway 2026-09-07:

| | first/last frame | multi-reference | length | relative cost |
|---|:--:|:--:|---|---|
| MiniMax H3 Max *(default)* | ✓ | **✗** | 5–15s | 1× |
| MiniMax H3 | ✓ | ✓ | 4–15s | 1× |
| Seedance 2.5 | ✓ | ✓ | 4–**30s** | 2× |
| Grok Imagine 1.5 | **✗** | ✓ | 1–15s | 1.5× |

Consequences worth saying out loud before someone plans around them:

- **A reference sheet forces a model change.** The default cannot take
  reference images, so any film with a locked character or product runs on
  Seedance or MiniMax H3 — and Seedance is 2× the price. Say this when the
  reference sheet is proposed, not when the bill arrives.
- **Only Seedance reaches 30 seconds.** A single continuous 30-second shot is
  one model's privilege; everything else must be cut from shorter pieces.
- **Grok cannot do first/last frame**, so it cannot carry a continuous cut.

## 6. Deliver the cutdowns

A shoot that produced a 60 is expected to yield 30 and 15 as well. Plan which
shots survive each cut while writing the shot table — the hook and the call to
action always survive; proofs are what get dropped first.

## Working inside a Dokki video project

When the work sits in a video project folder, the shot table is the single
source of truth — the director's board and the cutting bench are two views of
it. Write shots there rather than in prose, so the person can reorder and edit
them and so generation can read the same rows.

Never fire a batch of generations without showing what it will cost first. Seven
shots on Seedance is not "a few clips"; it is a real charge and several minutes.
