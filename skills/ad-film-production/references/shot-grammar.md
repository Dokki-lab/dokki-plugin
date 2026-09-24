# Shot grammar for AI video prompts

Vocabulary the models respond to. Reach for these words rather than adjectives
like "cinematic", "epic" or "high quality", which describe a feeling and change
almost nothing about the frame.

## The six-part form

> Subject → Action → Environment → Camera → Style → Constraints

```
A barista in a black apron          ← subject, specific enough to redraw
slides a glass of iced coffee       ← ONE action, specific verb
across a scratched walnut counter   ← environment, one concrete detail
medium close-up, eye level,         ← camera: size, angle,
slow dolly in                       ←         movement
warm morning window light,          ← style: light before look
shallow depth of field
condensation beading on the glass   ← constraint, stated as a thing PRESENT
```

## Shot size

| Term | What it frames | Use it for |
|---|---|---|
| extreme wide | figure small in the space | establishing, scale, isolation |
| wide | whole figure, head to toe | geography, physical action |
| medium | waist up | dialogue, product in hand |
| medium close-up | chest up | reaction, the moment a claim lands |
| close-up | face, or the product alone | emotion, texture, the hero shot |
| extreme close-up | eyes, a switch, a seam | detail as proof |

## Angle

`eye level` (neutral, honest) · `low angle` (power, scale, heroic product) ·
`high angle` (vulnerability, overview) · `overhead` / `bird's eye` (pattern,
layout, flat-lay) · `dutch` (unease — rare in ads, and never by accident)

## Movement

`static locked-off` · `slow dolly in` / `dolly out` · `tracking` (follows the
subject) · `pan` (pivot horizontally) · `tilt` (pivot vertically) ·
`orbit` (circles the subject) · `handheld` (energy, documentary honesty) ·
`crane up` / `crane down`

One move per shot. A shot that dollies in while orbiting and panning produces
mush.

## Light

Name the source and the time, not the mood: `warm morning window light`,
`overcast diffuse daylight`, `single hard key from camera left`,
`practical neon signage`, `dim indoor glow`, `backlit at golden hour`.

Light is the single biggest lever on whether two shots look like the same film.
Write the same light description into every shot of a scene.

## Writing constraints without negatives

Video models are trained to render what the prompt names. Naming a thing to
exclude tends to summon it. Convert every negative into the positive frame you
actually want:

| Instead of | Write |
|---|---|
| no text on screen | a clean frame, the product unobstructed |
| don't show other people | the shop empty except for the barista |
| avoid blurry footage | crisp focus on the glass, background softly out of focus |
| no camera shake | static locked-off camera on a tripod |

## Words that do nothing

`cinematic`, `4K`, `high quality`, `masterpiece`, `award-winning`,
`professional`. They cost tokens and buy no control. If you want the look they
gesture at, name the lens, the light and the movement instead.

## Sources

- [Seedance 2.5 prompt guide](https://www.seedance.tv/blog/seedance-2-5-prompt-guide)
- [Prompting tips per model — Kling, Veo, Sora, Seedance (Artlist)](https://help.artlist.io/hc/en-us/articles/31558164653213-Prompting-tips-tailored-to-different-AI-Video-models-Kling-Veo-Sora-and-Seedance)
- [Veo 3.1 field guide to cinematic control](https://sider.ai/blog/ai-tools/best-prompt-techniques-for-veo-3_1-video-output-a-field-guide-to-cinematic-control)
