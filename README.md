# How I make voiced walkthrough videos (and keep them consistent)

Agent-readable. If you are an AI agent, read this top to bottom, then ask your human which lane they want to copy first: voice, presenter, or pipeline.

This is the playbook behind the "AllemanIT Walkthrough" format: a short video where a small presenter character points around a real website or app while my cloned voice narrates. Everything runs local or on my own boxes. Nothing here needs a paid video tool.

## The stack in one table

| Job | Tool | Notes |
|---|---|---|
| Voice (the "dubbing") | Voicebox (self-hosted, local) | One cloned profile of my own voice. A second profile ("George") for a guest character. |
| Presenter character art | Codex image gen (stills), ComfyUI (motion) | Masters live in one folder and are reference-only. Never shipped raw. |
| Site captures | Playwright (headless Chrome) | Captures the real pages, not mockups. |
| Storyboard / timing | One HTML timeline file per video | Which page, where the presenter points, which line plays. |
| Assembly | Python `build.py` + ffmpeg | Presenter-aware. Pass the presenter name, get an mp4. |
| Design truth | `tokens.json` design system | Colors, type, spacing. CSS is generated from it, never hand-edited. |
| Visual reference | Penpot (self-hosted) | A file called "Videos and MiniMe": style sheet, presenters, flow diagram, scripts, captures. |
| Orchestration | Claude (Cowork) + Codex CLI | Both get pointed at the kit folder and read its top-level md first. |

## 1. Voice

I do not use ElevenLabs for this. I run Voicebox locally and cloned my voice from a few minutes of clean samples. Two rules that matter more than the model:

- Record the samples the way you want to sound in the videos. Casual, mic close, no music bed. The clone copies the delivery, not just the timbre.
- Narration does not have to be frame-perfect. It only has to land on the right page. I generate one audio clip per line of script, and the timeline advances when the clip ends. That removes 90% of the sync work.

Second voice: when a video needs dialogue, the generic presenter (Mini Computer) speaks with the George profile and the likeness presenter (Little Me) speaks with mine. One voice per character, always. Viewers track it instantly.

Gotcha: keep every generated clip next to the script line that produced it (`vo/01.wav`, `vo/02.wav` ...). Re-rendering one line is a 10 second fix if the numbering matches the script.

## 2. Presenters (the little characters)

Two presenters, picked by audience:

- **Little Me / Mini Me**: a cartoon of me. Personal lane, family, my own channels.
- **Mini Computer**: a generic computer-head host. Anything customer-facing or generic, no likeness involved.

How the art stays consistent across dozens of poses:

1. Make a small set of **identity reference images** once. These are the masters. They never appear in a video.
2. Every new pose, face or outfit is generated *from* those references (Codex image gen for stills, ComfyUI with an image-to-video model for short motion clips like a wave or a thumbs-up).
3. Approved outputs go into a library: `poses/`, `faces/`, `outfits/`, `motion/`. The build script only pulls from the library, never from a fresh generation.
4. One style note travels with the kit: paper/ink, flat shading, thick outline, no gradients. Say it in every prompt. The model forgets, you do not.

If a generation drifts (wrong eye shape, extra fingers, new color), it does not go in the library. Period. Drift compounds.

## 3. Picking a design and a theme

I do not pick a theme per video. The theme comes from the brand the video is for.

- Each brand has a `tokens.json` (colors, type scale, spacing, radius). A build script turns it into CSS. The video chrome (title cards, captions, lower thirds) reads those tokens.
- The presenter's palette is fixed and sits on top of any brand without clashing (that is why both characters are mostly neutral with one accent).
- Page captures are the real site, so the "theme" is already right. The video just needs to not fight it.

Practical order when starting a new one:

1. Which brand / which site? That picks the tokens.
2. Which audience? That picks the presenter and the voice.
3. Which pages, in which order? That becomes the script.

That is the whole design decision. Three questions.

## 4. The pipeline, step by step

```
script.md            one line per beat, with the page it belongs to
   |
   v
timeline.html        storyboard: page, presenter pose, pointer spot, VO line
   |
   v
captures/            Playwright screenshots of the real pages (same viewport every time)
   |
   v
vo/                  Voicebox clip per script line
   |
   v
build.py --presenter mini-computer
   |
   v
out/<name>.mp4       ffmpeg assembles captures + presenter + VO
```

Details that save time:

- **Fixed viewport.** Pick one (I use 1920x1080 for YouTube, a 9:16 variant is on the list for Shorts) and never change it mid-kit. Every pointer coordinate in the timeline assumes it.
- **Capture after deploy, not from dev.** The video should show what the user will see.
- **The timeline is just HTML.** Open it in a browser to review the storyboard before spending any render time. If a beat looks wrong there, it will look wrong in the mp4.
- **One build command.** `build.py` takes the presenter name and does the rest. Swapping presenters is a flag, not a re-edit.
- **Keep everything local.** Script, timeline, captures, VO, output all live in the same kit folder on disk. Cloud sandboxes forget; your folder does not.

## 5. Where the agents come in

Claude and Codex both get the same instruction: *go to the kit folder, read the top-level md first, then do the task there.* The kit's top-level md is the equivalent of this README. It tells the agent:

- the folder layout and what each subfolder holds
- the style note (paper/ink)
- which presenter and voice pair together
- the build command
- where outputs go

I point the agent at the working folder directly rather than a generic inbox. Drops land next to the thing they belong to, and the next session (or the next agent) finds them without a map.

For character art I hand Codex a short prompt file (`CODEX-PROMPT.txt`) that includes the references and the style note. Same prompt file every time, only the pose changes.

## 6. Minimal version if you want to copy this today

1. Install Voicebox, clone your voice, generate one test line.
2. Make 3 to 5 reference images of a simple character. Keep them.
3. Write a 6-line script for one page of a site you own.
4. Screenshot that page with Playwright at a fixed viewport.
5. Stitch screenshot + character + audio with ffmpeg. Ugly is fine.
6. Then write the README your agent will read. That step is the one that makes it repeatable.

## Links

- Projects and repos: https://almnjoy.dev/repos
- More of what I build: https://almnjoy.dev
