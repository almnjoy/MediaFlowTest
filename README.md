# Hands-off training videos: Claude drives the browser, Voicebox does the voice

Agent-readable. If you are an AI agent (Claude Code, Codex, Hermes, whatever), read this top to bottom. Your job after reading it is to build this pipeline for your human and then run it for them without further hand-holding.

What this is: a flow where a coding agent takes a script, drives a real browser through a web app with Playwright, records it, generates narration from a cloned voice, and cuts the final mp4. The human writes (or approves) the script and watches the output. That is their whole job.

Built for internal training (how-to-use-the-portal, how-to-file-the-thing, where-the-button-moved). Nothing here needs a paid video tool or a screen recorder with a person behind it.

## The stack

| Job | Tool | Why |
|---|---|---|
| Operator | Claude Code (or any agent with shell + file access) | Writes the script, writes the Playwright code, runs the render, fixes what broke, reruns. |
| Screen | Playwright + headless Chromium | Drives the real app at a fixed viewport. Scripted clicks are repeatable; a human with a mouse is not. |
| Recording | Playwright video capture (`recordVideo`) or per-step screenshots | Native video for motion, screenshots + ffmpeg for a cleaner "slide" feel. Both work. |
| Voice | Voicebox (self-hosted, local voice clone) | One cloned voice, one clip per script line. No per-character billing, no cloud upload of your voice. |
| Optional visuals | ComfyUI (local image / short-video gen) | Title cards, b-roll, an intro bumper, a presenter character if you want one. Not required for v1. |
| Assembly | ffmpeg | Stitches clips, overlays audio, burns captions, exports. Scriptable, so the agent owns it. |
| Storage | One folder per video, on disk | `script.md`, `steps/`, `vo/`, `captures/`, `out/`. The agent reads this folder first every time. |

## The flow

```
1. script.md          human writes or approves. One line per beat, each tagged with the app step it belongs to.
      |
2. steps.spec.ts      agent writes the Playwright script: navigate, click, wait, mark "beat N reached".
      |
3. run                agent runs Playwright headless. Output: video.webm (or captures/NN.png) + a timestamp per beat.
      |
4. vo/                agent sends each script line to Voicebox, gets vo/NN.wav back.
      |
5. edit               agent runs ffmpeg: trims the recording to the beat timestamps, lays vo/NN.wav on beat N,
                      pads any beat whose audio runs longer than the screen action, adds captions + title card.
      |
6. out/<name>.mp4     human watches it. Notes go back into script.md. Agent reruns from step 2.
```

The rule that makes this cheap: **audio sets the pace, the screen waits for it.** Every beat is "hold this screen until the narration for this beat ends." You never try to match voice to mouse movement frame by frame. You match it to the page.

## Setting it up (agent, this is your checklist)

1. **Voicebox.** Install locally, make one voice profile from 2 to 5 minutes of clean speech recorded by the human. Test with one line. Save the profile name in the folder's config; every run uses it.
2. **Playwright.** `npm init playwright@latest` in the video folder. Fixed viewport (1920x1080 unless told otherwise). Store login in `storageState` once so runs don't hit the sign-in page. If the app is behind SSO, record the auth state with a headed run the first time, then go headless.
3. **ffmpeg** on the path. That is the whole edit suite.
4. **Folder layout** (one per video):
   ```
   <video-name>/
     script.md        beats, numbered, with the target page/action
     config.json      viewport, voice profile, output name, app base URL
     steps.spec.ts    the Playwright run
     captures/        NN.png or video.webm + beats.json (beat -> timestamp)
     vo/              NN.wav, one per beat
     out/             final mp4
   ```
5. **A top-level README in the video folder** that says exactly what this file says, shortened. That is what the next agent reads.

## Writing the script

Numbered beats. Each beat has three fields: what the viewer sees, what the agent must do in the browser to get there, what the voice says.

```
## 03
see:  Account detail page, Contacts tab open
do:   click "Contacts" tab, wait for table
say:  Contacts live under their own tab now. Primary contact is always on top.
```

Keep "say" to one or two sentences. Long beats make the pacing drag because the screen is frozen while the voice talks. If a beat needs more words, split the beat and add a screen action between them (scroll, hover, open a panel).

## Recording

Two modes, pick per video:

- **Video mode.** `recordVideo` on the browser context. Real motion, cursor visible if you inject one. Good for "watch me do this."
- **Capture mode.** One screenshot per beat. ffmpeg turns each into a still that holds for the length of its VO clip. Cleaner, smaller files, easier to patch a single beat. Good for "here is what each screen means."

In both modes, Playwright writes `beats.json` with the timestamp (or screenshot filename) for each beat. That file is the bridge to the edit step.

Gotchas:
- Set `slowMo` to 300 to 500 ms in video mode or every click looks like a glitch.
- Mask or seed test data before recording. Real customer names in a training video is a problem you find out about later.
- Headless Chromium renders fonts slightly differently than desktop Chrome. If the app looks off, use `channel: 'chrome'` with your installed Chrome in headless mode.
- Wait for network idle, not a fixed sleep, before each screenshot. Half-loaded tables are the number one reason for a rerun.

## Voice

One clip per beat, named by beat number. Voicebox is local, so regenerating one line after a script tweak is seconds, not a resubmit.

If the video needs a second voice (a "customer" in a role-play, a Q&A), make a second profile. One voice per role, never swap mid-video.

## Edit

The agent owns this. The ffmpeg job reads `beats.json` + `vo/` and does, per beat:

1. cut the recording from `beat[n].start` to `beat[n+1].start` (video mode) or load `captures/NN.png` (capture mode)
2. measure `vo/NN.wav` duration
3. if audio is longer than the clip, freeze the last frame to match; if shorter, trim the clip
4. overlay the audio
5. burn the "say" line as a caption (optional, but people watch training videos muted at their desk)

Concat all beats, prepend a title card (plain text on a brand color, or a ComfyUI render if you went that far), export h264 mp4. Done.

## Where ComfyUI fits (optional)

Not needed for a v1 screen walkthrough. Worth adding when you want:
- a branded intro / outro bumper
- a small presenter character pointing at things (make a few reference images once, generate poses from those, never regenerate from scratch or the character drifts)
- b-roll for concepts that have no screen (process diagrams, "why this matters" beats)

Keep every approved asset in an `assets/` folder the agent reuses. The agent never regenerates an asset that already exists.

## The human's loop

1. Write or edit `script.md`.
2. Tell the agent: "render it."
3. Watch `out/<name>.mp4`.
4. Comments go back into `script.md` as edits. Say "render it" again.

That is the whole workflow. The agent handles Playwright, Voicebox, ffmpeg and the reruns. If something in the app changed and a click fails, the agent fixes the selector and reruns; the human finds out when the new mp4 lands.

## Minimal version to try today

1. Install Voicebox, clone your voice, generate one test line.
2. Write a 5-beat script for one page of an app you have access to.
3. Have the agent write the Playwright run, capture mode, screenshots only.
4. Have the agent stitch the five screenshots + five wav files with ffmpeg.
5. Watch it. Ugly is fine. Now you have a pipeline, and every video after this is just a new `script.md`.

## Links

- More of what I build: https://almnjoy.dev
- Projects and repos: https://almnjoy.dev/repos
