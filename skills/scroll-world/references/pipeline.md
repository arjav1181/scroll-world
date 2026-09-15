# Pipeline: copy-paste scripts (bash 3.2 safe)

Set these once. `NAMES` is the ordered section ids; the last is the hero/finale.

```bash
WORK=/tmp/scroll-world           # scratch dir for prompts, sources, frames
ASSETS=./assets                  # where the site reads stills (webp) + clips (mp4)
mkdir -p "$WORK" "$ASSETS/vid"
NAMES="farm kitchen shop delivery plaza finale"   # <-- your section ids, in order

# Chain video model — ONE for every chained clip (SKILL Step 4 roster).
# Must accept --start-image AND --end-image (verify: higgsfield model get <model>):
# seedance_2_0 | kling3_0 | seedance_2_0_mini (draft tier). Reference-only models can't
# hold a seam; models without --mode (e.g. kling3_0_turbo) need their own flag branch below.
# DEFAULT backend: Monid pay-per-clip (§7 — gen_dive_monid/gen_conn_monid with
# VRES=1080p|720p|480p instead of VOPTS). The gen_dive/gen_conn functions in §2/§4
# below are the Higgsfield-credits FALLBACK (and the only home of kling3_0/mini).
VMODEL=seedance_2_0
case "$VMODEL" in                                  # per-model flags + durations (bash 3.2 safe)
  kling3_0)          VOPTS="--mode std --sound off";          DIVE_DUR=10; CONN_DUR=5 ;;  # no --resolution param on Kling
  seedance_2_0_mini) VOPTS="--mode std --resolution 720p";    DIVE_DUR=8;  CONN_DUR=5 ;;  # cheap frame-locked previz
  *)                 VOPTS="--mode std --resolution 1080p";   DIVE_DUR=8;  CONN_DUR=5 ;;  # seedance_2_0 default
esac
```

Higgsfield generations take minutes — every `higgsfield ... --wait` call below is meant
to run inside a **backgrounded** script. Launch the whole script with your tool's
background/detached mode and poll the progress log; never block the foreground.

## 1. Scene stills (Step 2)

Write one prompt file per section to `$WORK/still_<name>.txt` (see prompts.md), then:

```bash
gen_still() { # name
  higgsfield generate create gpt_image_2 --prompt "$(cat "$WORK/still_$1.txt")" \
    --aspect_ratio 3:2 --resolution 2k --quality high --wait --wait-timeout 15m --json \
    > "$WORK/still_$1.json" 2> "$WORK/still_$1.err"
  url=$(jq -r '.[0].result_url // empty' "$WORK/still_$1.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/still_$1.png" && echo "still $1 ok" || echo "still $1 FAIL"
}
for n in $NAMES; do gen_still "$n" & done ; wait
```

Codex variant (STILLS_SOURCE=codex, SKILL Step 1.7 — subscription-billed, zero
credits; ~1–3 min each, parallelize in small batches):

```bash
gen_still_codex() { # name   (< /dev/null is REQUIRED for parallel calls — see SKILL Gotchas)
  codex exec -C "$WORK" -s workspace-write --skip-git-repo-check \
    'Use the image generation tool ($imagegen) to generate: '"$(cat "$WORK/still_$1.txt")"' Wide 3:2 landscape, high resolution. Save it as ./still_'"$1"'.png. Do not do anything else.' \
    > "$WORK/still_$1.codex.log" 2>&1 < /dev/null
  [ -f "$WORK/still_$1.png" ] && echo "still $1 ok (codex)" || echo "still $1 FAIL (see .codex.log)"
}
```

Convert to webp for the site (and optionally run knockout.py first for transparency):

```bash
for n in $NAMES; do cwebp -quiet -q 84 -resize 1800 0 "$WORK/still_$n.png" -o "$ASSETS/$n.webp"; done
```

Review the stills for cohesion before continuing. Re-roll any off-style one (optionally
add `--image "$WORK/still_<good>.png"` to lock style).

## 2. Dive-in clips (Step 4)

Prompt files at `$WORK/dive_<name>.txt`. Start image = the solid-bg still PNG.

```bash
gen_dive() { # name                       ($VOPTS is unquoted on purpose — word-split flags)
  higgsfield generate create "$VMODEL" --prompt "$(cat "$WORK/dive_$1.txt")" \
    --start-image "$WORK/still_$1.png" \
    $VOPTS --aspect_ratio 16:9 --duration "$DIVE_DUR" \
    --wait --wait-timeout 20m --json > "$WORK/dive_$1.json" 2> "$WORK/dive_$1.err"
  url=$(jq -r '.[0].result_url // empty' "$WORK/dive_$1.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/dive_$1.mp4" && echo "dive $1 ok" || echo "dive $1 FAIL"
}
for n in $NAMES; do gen_dive "$n" & done ; wait
```

Re-roll individual failures (503 / credit race are transient):
`gen_dive shop`  (just that one).

## 3. Extract boundary frames — the seam handoff (Step 5)

For each adjacent pair, the connector's start = dive_i's LAST frame, end = dive_{i+1}'s
FIRST frame — extracted from the **rendered videos**, never the stills.

```bash
set -- $NAMES
prev=""
for n in "$@"; do
  ffmpeg -v error -ss 0 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/first_$n.png"      # establishing
  ffmpeg -v error -sseof -0.15 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/last_$n.png" # interior
done
```

## 4. Connector clips (Step 5)

Prompt files at `$WORK/conn_<i>.txt` (i = 1..N-1). Iterate adjacent pairs:

```bash
gen_conn() { # i startPng endPng          (end-image required → seedance/kling3_0 only)
  higgsfield generate create "$VMODEL" --prompt "$(cat "$WORK/conn_$1.txt")" \
    --start-image "$2" --end-image "$3" \
    $VOPTS --aspect_ratio 16:9 --duration "$CONN_DUR" \
    --wait --wait-timeout 20m --json > "$WORK/conn_$1.json" 2> "$WORK/conn_$1.err"
  url=$(jq -r '.[0].result_url // empty' "$WORK/conn_$1.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/conn_$1.mp4" && echo "conn $1 ok" || echo "conn $1 FAIL"
}
set -- $NAMES ; i=0 ; prev=""
for n in "$@"; do
  if [ -n "$prev" ]; then i=$((i+1)); gen_conn "$i" "$WORK/last_$prev.png" "$WORK/first_$n.png" & fi
  prev="$n"
done ; wait
```

## 5. Encode everything for scrubbing (Step 6)

Native resolution (1080p from seedance std; kling3_0 std returned **720p** in testing —
never upscale, encode what ffprobe reports), crf 20, GOP 8, light sharpen, no audio,
faststart. Same for dives + connectors.

```bash
enc() { ffmpeg -v error -y -i "$1" -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart "$2"; echo "enc $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do enc "$WORK/dive_$n.mp4" "$ASSETS/vid/$n.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); enc "$f" "$ASSETS/vid/conn$i.mp4"; done
```

Now the engine config's `sections[k].clip = assets/vid/<name>.mp4` and
`connectors = [assets/vid/conn1.mp4, …]` (length N-1, in order).

## 6. Centre-crop mobile encodes — FALLBACK ONLY, not the mobile version

**The mobile version is the native 9:16 portrait chain (§6b).** This section's crop
encodes exist for one case: the user opted into mobile but credits can't cover the
portrait chain — and shipping them must be called out and approved, never silent
(portrait phones will see the landscape film's centre ~26%). The encode mechanics
matter either way: scrubbing sets `currentTime` every frame, and a phone decoder's
**seek cost scales with how many frames it must decode from the nearest keyframe** — so
a 1080p `-g 8` master that scrubs fine on a laptop stutters on a phone. A **smaller
frame + tighter GOP** fixes that (and halves the bytes on cellular). The crop `-m.mp4`
sibling per clip:

```bash
# 720p, GOP 4 (twice the keyframes = ~half the seek-decode work), crf 23, same sharpen/faststart.
encm() { ffmpeg -v error -y -i "$1" -an -vf "scale=-2:720,unsharp=5:5:0.6:5:5:0.0" \
  -c:v libx264 -preset slow -crf 23 -pix_fmt yuv420p \
  -g 4 -keyint_min 4 -sc_threshold 0 -movflags +faststart "$2"; echo "encm $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do encm "$WORK/dive_$n.mp4" "$ASSETS/vid/$n-m.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); encm "$f" "$ASSETS/vid/conn$i-m.mp4"; done
```

Wire the variants in the engine config — the engine serves them automatically on phones,
falling back to the desktop `clip` when a mobile one is absent:

```js
sections[k].clipMobile = 'assets/vid/<name>-m.mp4';
connectorsMobile = ['assets/vid/conn1-m.mp4', …];   // length N-1, in order
```

If phone scrubbing still stutters, tighten the GOP further (`-g 2`, or `-g 1` for all-intra
= instant seeks at the cost of larger files); if cellular weight is the bigger worry, raise
`crf` (24–26) or drop to `scale=-2:600`. If the master is already 720p (e.g. kling3_0 std),
the mobile encode still pays off — the tighter GOP is what makes phone seeks cheap. All-mobile encodes stay 16:9 — the engine
centre-crops them; see the portrait note in SKILL Step 8 / prompts.md.

## 6b. Native 9:16 portrait chain — THE mobile version (Step 1.6 opt-in)

When the user opts into mobile, this is what they get: a **parallel 9:16 chain** rendered
natively for phones and shipped as the mobile variants — never the §6 crops (those are the
no-credits stopgap). Same seam laws as the main chain — the portrait chain frame-locks
against its own rendered frames, never the landscape ones. Budget ~2N-1 video gens +
re-rolls (interiors trip the NSFW filter in portrait too); state the credit cost at the
Step 1.6 interview.

1. **Portrait start canvases.** Don't hand the video model a 3:2 still and hope: composite
   each scene onto a 1080×1920 canvas in the page bg colour (island at ~94% width, visual
   centre at ~45% height). The render then opens exactly on what the portrait poster shows.
   For knocked-out stills, composite the RGBA over the bg colour first.
2. **Dives/legs**: same prompt templates with a portrait clause up front ("Vertical
   portrait composition, the diorama centered with generous [bg] space above and below"),
   `--aspect_ratio 9:16`, same model/params as the main chain. Review each last frame
   before chaining, as ever.
3. **Connectors**: extract first/last frames **from the 9:16 renders** and generate 9:16
   connectors between them. A native 9:16 scene mixed into cropped-16:9 neighbours pops at
   both seams — the portrait chain must be complete, not partial.
4. **Encode** with the §6 settings but portrait-oriented scale: `scale=720:-2` (720 wide),
   `-g 4`, crf 23 → these ARE the `-m.mp4` mobile files (and they replace any §6 crop
   stopgaps that shipped earlier).
5. **Posters**: extract each 9:16 dive's first frame → webp → wire as the section's
   `stillMobile` so the poster matches the portrait video's frame 0 (no landscape→portrait
   flash when the clip paints). Engine support: `sections[k].stillMobile`.

## 7. Monid backend — Seedance 2.0 pay-per-clip (the DEFAULT; qualified 2026-07-25)

`bytedance /v1/video/seedance-2.0` via Monid is the roster's `seedance_2_0` billed
per clip in USD, and the **default** chain backend (SKILL Step 4 → Monid backend;
both probes passed) — use these functions in the §2/§4 loops unless the build fell
back to Higgsfield credits. Same chain laws as everywhere else — only the I/O
differs: **frames ride Monid's free `sfs` file
system** (inline base64 is rejected), **`ratio` must be explicit** (the adaptive
default follows the input image's aspect), and runs are fire-and-poll. Token-priced
`w×h×24×sec/1024` at $7/1M (480p/720p), $7.7/1M (1080p) — measured: 1080p ≈ $2.99
dive / $1.87 connector; 720p ≈ $1.21 / $0.76; 480p previz ≈ $0.28 / $0.35.

```bash
# helper: upload a local frame to sfs, print a signed public URL for it ($0).
# NB: /cat and /ls take the SAME relative path you gave /put — not the
# "home/..."-prefixed path /put echoes back (that one 404s).
monid_frame_url() { # localPng remoteName  (JPEG-compresses on the way up)
  jpg="$WORK/sfs_$2.jpg"
  ffmpeg -v error -y -i "$1" -vf "scale='min(1536,iw)':-2" -q:v 2 "$jpg"
  size=$(stat -f%z "$jpg")
  up=$(NO_COLOR=1 monid run -p sfs -e /put \
    -i "{\"path\":\"chain/$2.jpg\",\"sizeBytes\":$size,\"ttl\":\"1h\"}" -w 60 -j \
    | jq -r '.output.uploadUrl')
  curl -fsS -T "$jpg" "$up" > /dev/null
  NO_COLOR=1 monid run -p sfs -e /cat -i "{\"path\":\"chain/$2.jpg\",\"ttl\":\"1d\"}" \
    -w 60 -j | jq -r '.output.url'
}

# fire-and-poll (the CLI's -w caps at 120s and seedance can exceed it)
monid_wait() { # runId outJson
  while :; do
    NO_COLOR=1 monid runs get -r "$1" -j > "$2" 2>/dev/null
    case "$(jq -r '.status // empty' "$2")" in
      COMPLETED|FAILED|BLOCKED|STOPPED|TIME_OUT) break ;;
    esac
    sleep 8
  done
}

gen_dive_monid() { # name   (VRES=1080p|720p|480p; DIVE_DUR as usual)
  furl=$(monid_frame_url "$WORK/still_$1.png" "still_$1")
  jq -n --arg p "$(cat "$WORK/dive_$1.txt")" --arg u "$furl" --arg r "$VRES" \
    '{content:[{type:"text",text:$p},
               {type:"image_url",image_url:{url:$u},role:"first_frame"}],
      resolution:$r, duration:'"$DIVE_DUR"', ratio:"16:9", generate_audio:false}' \
    > "$WORK/dive_$1.body.json"
  rid=$(NO_COLOR=1 monid run -p bytedance -e /v1/video/seedance-2.0 \
    -f "$WORK/dive_$1.body.json" -j | jq -r '.runId')
  monid_wait "$rid" "$WORK/dive_$1.json"
  url=$(jq -r '.output.content.video_url // empty' "$WORK/dive_$1.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/dive_$1.mp4" \
    && echo "dive $1 ok (\$$(jq -r '.cost.value' "$WORK/dive_$1.json"))" \
    || echo "dive $1 FAIL ($(jq -r '.status' "$WORK/dive_$1.json"))"
}

gen_conn_monid() { # i startPng endPng
  su=$(monid_frame_url "$2" "conn$1_start"); eu=$(monid_frame_url "$3" "conn$1_end")
  jq -n --arg p "$(cat "$WORK/conn_$1.txt")" --arg s "$su" --arg e "$eu" --arg r "$VRES" \
    '{content:[{type:"text",text:$p},
               {type:"image_url",image_url:{url:$s},role:"first_frame"},
               {type:"image_url",image_url:{url:$e},role:"last_frame"}],
      resolution:$r, duration:'"$CONN_DUR"', ratio:"16:9", generate_audio:false}' \
    > "$WORK/conn_$1.body.json"
  rid=$(NO_COLOR=1 monid run -p bytedance -e /v1/video/seedance-2.0 \
    -f "$WORK/conn_$1.body.json" -j | jq -r '.runId')
  monid_wait "$rid" "$WORK/conn_$1.json"
  url=$(jq -r '.output.content.video_url // empty' "$WORK/conn_$1.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/conn_$1.mp4" \
    && echo "conn $1 ok (\$$(jq -r '.cost.value' "$WORK/conn_$1.json"))" \
    || echo "conn $1 FAIL ($(jq -r '.status' "$WORK/conn_$1.json"))"
}
```

Default usage in §2/§4: same loops, `gen_dive_monid`/`gen_conn_monid` in place of
the Higgsfield `gen_dive`/`gen_conn` (mobile chain: `ratio:"9:16"` and the §6b
portrait canvases). Previz: same functions with `VRES=480p`.
Result URLs expire (~24–48 h) — the functions download immediately. Read the billed
`cost.value` per clip (echoed in the ok-line) and `monid balance` between phases; a
`BLOCKED` status is a workspace budget/run cap — terminal, surface it to the user.
Frame extraction, encoding, QA: identical to the Higgsfield path.

**Qualification harness for future/changed endpoints** (each probe = one cheap 480p
clip): (1) prompt + first_frame from a real still → downloaded video's frame 0 must
match the still (eyeball + PSNR ≳ 30 dB) and `cost.value` must match the advertised
cell; (2) add a last_frame from a *different* still → the final frame must land on
that composition (Seedance-style near-miss ok — the crossfade covers it). Known
failure to watch for (it's why the harness exists): `minimax /v1/video_generation`
still silently drops the image when `prompt` is present — image-only output proves
nothing about steerability.

## 8. Free backends — $0-cash chains (BACKEND shootout)

Set once. `BACKEND` is the Step 1.7 stack answer; `CAP` is its live capability flag
(SKILL Step 4 — CHAIN = start+end frames, A-ONLY = start frame only). The guard fails
fast on mismatch so a start-only backend can never be asked for connectors.

```bash
BACKEND=pollinations   # pollinations | local | cloudflare | huggingface | siliconflow | novita
POLL_MODEL=alibaba/wan-2.2-fast   # a model whose live video_capabilities fit CAP (no paid_only flag)
CAP=A-ONLY             # CHAIN or A-ONLY (re-check /image/models every build)
DIVE_DUR=8; CONN_DUR=5 # defaults — ALWAYS snap to the live grid before batching:
ASPECT=16:9            # 9:16 for the §6b mobile chain
# DIVE_DUR=$(snap_dur "$POLL_MODEL" 8); CONN_DUR=$(snap_dur "$POLL_MODEL" 5)
# (nova-reel-class takes 6s multiples — blind durations 400. §8e helpers.)
[ -n "$POLLINATIONS_KEY" ] || echo "WARN: no POLLINATIONS_KEY — free tier throttles/fails"

# Guard: capability gates architecture (SKILL Step 4 rule).
case "$CAP" in
  CHAIN)  echo "backend $BACKEND ($POLL_MODEL): arch A or B" ;;
  A-ONLY) echo "backend $BACKEND ($POLL_MODEL): arch A ONLY — connectors disabled" ;;
  *)      echo "backend $BACKEND DISQUALIFIED for chain duty"; exit 1 ;;
esac
```

Frames ride public URLs on hosted backends (they take URLs, never local paths).
Free file host for the handoff frames — test the URL before burning Pollen on it:

```bash
# upload a local frame, print a public HOTLINK url (verified 2026-09-15: uguu.se
# round-trips byte-identical; expires in ~3h — upload inside the build, use at once).
poll_frame_url() { # localPng  (JPEG-compresses on the way up)
  jpg="$WORK/up_$(basename "$1" .png).jpg"
  ffmpeg -v error -y -i "$1" -vf "scale='min(1536,iw)':-2" -q:v 2 "$jpg"
  curl -fsSL -F "files[]=@$jpg" https://uguu.se/upload.php | jq -r '.files[0].url // empty'
}
```

### 8a. Free stills (Step 2 — one source for all N stills)

```bash
# Pollinations (free-hosted default): 1536x1024, fixed seed per scene.
# Model via $POLL_IMAGE_MODEL (default flux; cascade overrides for mirrors).
gen_still_poll() { # name seed
  curl -fsSL --max-time 600 --get "https://gen.pollinations.ai/image/$(python3 -c "import urllib.parse,sys;print(urllib.parse.quote(open('$WORK/still_$1.txt').read()))")" \
    --data-urlencode "model=${POLL_IMAGE_MODEL:-flux}" --data-urlencode "width=1536" --data-urlencode "height=1024" \
    --data-urlencode "seed=$2" --data-urlencode "nologo=true" \
    -H "Authorization: Bearer $POLLINATIONS_KEY" -o "$WORK/still_$1.png" \
    && echo "still $1 ok (poll/${POLL_IMAGE_MODEL:-flux})" || echo "still $1 FAIL"
}

# Cloudflare Workers AI (trials lane): schnell, then force 3:2 (output runs square-ish).
gen_still_cf() { # name
  curl -fsSL "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/ai/run/@cf/black-forest-labs/flux-1-schnell" \
    -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" -H 'Content-Type: application/json' \
    -d "$(jq -n --arg p "$(cat "$WORK/still_$1.txt")" '{prompt:$p,steps:4}')" \
    | jq -r '.result.image // empty' | base64 -d > "$WORK/still_$1.sq.png"
  ffmpeg -v error -y -i "$WORK/still_$1.sq.png" -vf "scale=1536:1024:force_original_aspect_ratio=increase,crop=1536:1024" "$WORK/still_$1.png" \
    && echo "still $1 ok (cf)" || echo "still $1 FAIL"
}

# Hugging Face Inference (trials lane): schnell via InferenceClient. Mind the free credit.
gen_still_hf() { # name seed
  HF_PROMPT="$WORK/still_$1.txt" HF_OUT="$WORK/still_$1.png" HF_SEED="$2" python3 -c "
import os; from huggingface_hub import InferenceClient
c = InferenceClient(token=os.environ['HF_TOKEN'])
img = c.text_to_image(open(os.environ['HF_PROMPT']).read(),
      model='black-forest-labs/FLUX.1-schnell')
img.save(os.environ['HF_OUT'])" \
    && echo "still $1 ok (hf)" || echo "still $1 FAIL"
}

# SiliconFlow (trials lane): schnell, freeform 1536x1024. URLs expire in ~1h — instant download.
gen_still_sf() { # name seed
  url=$(curl -fsSL https://api.siliconflow.com/v1/images/generations \
    -H "Authorization: Bearer $SILICONFLOW_API_KEY" -H 'Content-Type: application/json' \
    -d "$(jq -n --arg p "$(cat "$WORK/still_$1.txt")" --argjson s "$2" \
      '{model:"black-forest-labs/FLUX.1-schnell",prompt:$p,image_size:"1536x1024",seed:$s}')" \
    | jq -r '.images[0].url // empty')
  [ -n "$url" ] && curl -fsSL "$url" -o "$WORK/still_$1.png" && echo "still $1 ok (sf)" || echo "still $1 FAIL"
}

# Local diffusers (optional lane): FLUX.1-schnell, unlimited, commercial-safe. Needs GPU.
gen_still_local() { # name seed
  STILL_PROMPT="$WORK/still_$1.txt" STILL_OUT="$WORK/still_$1.png" STILL_SEED="$2" python3 -c "
import os, torch
from diffusers import FluxPipeline
p = FluxPipeline.from_pretrained('black-forest-labs/FLUX.1-schnell', torch_dtype=torch.bfloat16)
p.enable_model_cpu_offload()
img = p(prompt=open(os.environ['STILL_PROMPT']).read(), width=1536, height=1024,
        num_inference_steps=4, generator=torch.Generator().manual_seed(int(os.environ['STILL_SEED']))).images[0]
img.save(os.environ['STILL_OUT'])" \
    && echo "still $1 ok (local)" || echo "still $1 FAIL"
}

# HF Spaces Gradio (free-hosted, KEYLESS — verified 2026-09-15: 1536x1024 schnell
# still, $0, no signup; ZeroGPU quotas: ~2 min/day unauth, ~5 min/day free account —
# a 4-step schnell still costs seconds, so N=6 fits, barely; add HF_TOKEN to raise it).
# Needs: pip install gradio_client. Files land in /tmp/gradio/<hash>/ — copy them out.
gen_still_hfspace() { # name seed
  STILL_PROMPT="$WORK/still_$1.txt" STILL_OUT="$WORK/still_$1.png" STILL_SEED="$2" python3 -c "
import os, shutil, glob
from gradio_client import Client
c = Client('black-forest-labs/FLUX.1-schnell')
path, _ = c.predict(prompt=open(os.environ['STILL_PROMPT']).read(),
        seed=int(os.environ['STILL_SEED']), randomize_seed=False,
        width=1536, height=1024, num_inference_steps=4, api_name='/infer')
shutil.copy(path, os.environ['STILL_OUT'])" \
    && echo "still $1 ok (hfspace)" || echo "still $1 FAIL (quota? retry later / add HF_TOKEN)"
}
```

If Pollinations `flux` ever flips to `paid_only`, stay keyless by swapping the model in
`gen_still_poll` to a free community mirror (live-verified free 2026-09-15:
`community/MarcosFRG/flux-1-schnell`, `community/CloudCompile/flux-2-klein-4b`,
`community/CloudCompile/sdxl-lightning`) — same params, then re-run one still and
eyeball the style before batching.

Run the batch with your lane's function (`seed = index*77` keeps scenes distinct but
reproducible): `i=0; for n in $NAMES; do gen_still_poll "$n" $((i*77)); i=$((i+1)); done`
— then webp + cohesion review exactly as §1.

### 8b. Free video chain — Pollinations (CHAIN or A-ONLY per live flags)

Dives take the start-frame URL; connectors append `|<end-url>` (end-frame ignored by
models without it — that's the downgrade-to-A-ONLY signal, SKILL Gotchas). Duration
must sit on the model's grid and in `[min_duration,max_duration]` (read them off
`/image/models`).

```bash
gen_dive_poll() { # name
  furl=$(poll_frame_url "$WORK/still_$1.png")
  curl -fsSL --get "https://gen.pollinations.ai/video/$(python3 -c "import urllib.parse;print(urllib.parse.quote(open('$WORK/dive_$1.txt').read()))")" \
    --data-urlencode "model=$POLL_MODEL" --data-urlencode "duration=$DIVE_DUR" \
    --data-urlencode "aspectRatio=$ASPECT" --data-urlencode "image=$furl" --data-urlencode "audio=false" \
    -H "Authorization: Bearer $POLLINATIONS_KEY" -o "$WORK/dive_$1.mp4" \
    && echo "dive $1 ok (poll/$POLL_MODEL)" || echo "dive $1 FAIL"
}
gen_conn_poll() { # i startPng endPng   (CAP=CHAIN only — the guard above enforces it)
  [ "$CAP" = "CHAIN" ] || { echo "conn $1 REFUSED ($POLL_MODEL is A-ONLY)"; return 1; }
  su=$(poll_frame_url "$2"); eu=$(poll_frame_url "$3")
  curl -fsSL --get "https://gen.pollinations.ai/video/$(python3 -c "import urllib.parse;print(urllib.parse.quote(open('$WORK/conn_$1.txt').read()))")" \
    --data-urlencode "model=$POLL_MODEL" --data-urlencode "duration=$CONN_DUR" \
    --data-urlencode "aspectRatio=$ASPECT" --data-urlencode "image=$su|$eu" --data-urlencode "audio=false" \
    -H "Authorization: Bearer $POLLINATIONS_KEY" -o "$WORK/conn_$1.mp4" \
    && echo "conn $1 ok (poll/$POLL_MODEL)" || echo "conn $1 FAIL"
}
for n in $NAMES; do gen_dive_poll "$n" & done ; wait
```

A-ONLY lanes (nova-reel-class, SiliconFlow/Novita I2V) render **architecture-A legs**:
same `gen_dive_poll`, sequentially, each leg's `--start-image` = previous leg's ACTUAL
last frame (§3 extraction unchanged), no `--end-image` anywhere, `connectors: []`.
Mobile chain: same functions with `ASPECT=9:16` + the §6b portrait canvases.

### 8b2. Trial CHAIN — Novita Wan-I2V first+last-frame (verified 2026-09: `wan2.7-i2v`
takes `image_url` + `last_frame_url`; async submit → poll `task-result`)

```bash
novita_wait() { # taskId outJson
  while :; do sleep 10
    NO_COLOR=1 curl -fsSL "https://api.novita.ai/v3/async/task-result?task_id=$1" \
      -H "Authorization: Bearer $NOVITA_API_KEY" > "$2" 2>/dev/null
    case "$(jq -r '.status // .task.status // empty' "$2")" in
      TASK_STATUS_SUCCEED|SUCCESS|COMPLETED) break ;;
      TASK_STATUS_FAILED|FAILED) break ;;
    esac
  done
}
gen_novita() { # outMp4 startPng endPngOrEmpty promptTxt dur res (720P|1080P)
  su=$(poll_frame_url "$2"); [ -n "$3" ] && eu=$(poll_frame_url "$3") || eu=""
  body=$(jq -n --arg p "$(cat "$5")" --arg s "$su" --arg e "$eu" --arg r "$7" --argjson d "$6" \
    '{model:"wan2.7-i2v", input:{prompt:$p, image_url:$s} + (if $e != "" then {last_frame_url:$e} else {} end),
      parameters:{resolution:$r, duration:$d}}')
  tid=$(curl -fsSL https://api.novita.ai/v3/async/wan2.7-i2v \
    -H "Authorization: Bearer $NOVITA_API_KEY" -H 'Content-Type: application/json' \
    -d "$body" | jq -r '.task_id // empty')
  [ -z "$tid" ] && { echo "clip $1 FAIL (submit)"; return 1; }
  novita_wait "$tid" "$1.task.json"
  url=$(jq -r '.videos[0].url // .video_url // .output.video_url // empty' "$1.task.json")
  [ -n "$url" ] && curl -fsSL "$url" -o "$1" && verify_clip "$1" \
    && echo "clip $1 ok (novita)" || echo "clip $1 FAIL ($(jq -r '.status' "$1.task.json"))"
}
# dives/legs: gen_novita "$WORK/dive_$n.mp4" "still_$n.png" "" "$WORK/dive_$n.txt" 8 1080P
# connectors (CAP=CHAIN only): gen_novita "$WORK/conn_$i.mp4" "last_$prev.png" "first_$n.png" "$WORK/conn_$i.txt" 5 1080P
# Trial credit (~$0.50) covers qualification + short chains, not N=6 1080p — meter it.
```

### 8c. Free video chain — local ComfyUI Wan-FLF2V (CHAIN, $0 forever)

Needs the Wan2.1/2.2-FLF2V workflow in ComfyUI (`WanFirstLastFrameToVideo`,
diffusion + umt5 + vae + clip_vision_h loaded; 14B fp8 ≈ 15 GB VRAM, 480×854
fallback below). Drop stills/frames into ComfyUI's `input/` dir, keep one workflow
template with `START_PNG`/`END_PNG`/`PROMPT_TXT` placeholders, then:

```bash
COMFY=http://127.0.0.1:8188
gen_flf_local() { # outMp4 startPng endPngOrEmpty promptTxt
  jq --arg s "$(basename "$2")" --arg e "$(basename "$3")" --arg p "$(cat "$4")" \
    '(.nodes[] | select(.id=="START") | .inputs.image) = $s
     | (.nodes[] | select(.id=="END")   | .inputs.image) = $e
     | (.nodes[] | select(.id=="PROMPT") | .inputs.text) = $p' \
    "$WORK/flf_template.json" > "$WORK/flf_job.json"
  pid=$(curl -fsSL "$COMFY/prompt" -H 'Content-Type: application/json' \
    -d "{\"prompt\": $(cat "$WORK/flf_job.json")}" | jq -r '.prompt_id')
  while :; do sleep 10
    done=$(curl -fsSL "$COMFY/history/$pid" | jq -r ".[\"$pid\"].status.completed // false")
    [ "$done" = "true" ] && break
  done
  vid=$(curl -fsSL "$COMFY/history/$pid" | jq -r ".. | .gifs? // empty | .[0].filename // empty")
  curl -fsSL "$COMFY/view?filename=$vid&subfolder=video" -o "$1" \
    && echo "clip $1 ok (local)" || echo "clip $1 FAIL"
}
# dives: gen_flf_local "$WORK/dive_$n.mp4" "still_$n.png" "" "$WORK/dive_$n.txt" (parallel ok)
# legs (arch A): sequential, start = previous leg's actual last frame (§3)
# connectors (arch B): gen_flf_local "$WORK/conn_$i.mp4" "last_$prev.png" "first_$n.png" "$WORK/conn_$i.txt"
```

Frame extraction (§3), encoding (§5/§6), mobile (ASPECT/§6b) and QA are identical on
every backend — only generation differs. Result URLs on hosted lanes expire
(hours–days): every function above downloads immediately; never batch-generate then
batch-download.

### 8d. Hardening — retries, verify, manifest (use on every free lane)

Free lanes fail transiently (queues, quotas, file-host hiccups). Wrap every generator:

```bash
# retry3 <tries> <cmd...> — 3 tries, exponential backoff, logs to $WORK/retry.log
retry3() { t=$1; shift; i=0
  while [ $i -lt $t ]; do
    "$@" && return 0
    i=$((i+1)); echo "retry $i/$t: $* ($(date +%H:%M))" >> "$WORK/retry.log"; sleep $((15*i))
  done; return 1
}
# verify_clip <mp4> <minSec> — real video? (ffprobe duration + nonzero frames)
verify_clip() {
  dur=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$1" 2>/dev/null || echo 0)
  python3 -c "import sys; sys.exit(0 if float('${dur:-0}') >= ${2:-4} else 1)"
}
# manifest line per finished clip: name backend model bytes seconds (Step 1.7 accounting)
note() { printf '%s | %s | %s | %sB | %ss\n' "$1" "$BACKEND" "${2:-$POLL_MODEL}" \
  "$(stat -f%z "$WORK/$1" 2>/dev/null || stat -c%s "$WORK/$1")" \
  "$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$WORK/$1")" >> "$WORK/MANIFEST.txt"; }
# usage: retry3 3 gen_dive_poll farm && verify_clip "$WORK/dive_farm.mp4" 7 && note "dive_farm.mp4"

# Pre-flight: confirm the free allowance BEFORE batching (Step 1.7 calibration).
poll_models() { # dump live catalog flags for the chosen model
  curl -fsSL https://gen.pollinations.ai/image/models | python3 -c "
import json,sys,os
want = os.environ.get('POLL_MODEL','')
for m in json.load(sys.stdin):
    if m.get('name')==want or want in (m.get('aliases') or []):
        print('model:', m['name']); print('paid_only:', m.get('paid_only', False))
        print('video_cap:', m.get('video_capabilities')); print('dur:', m.get('min_duration'), '-', m.get('max_duration'))
        print('price:', m.get('pricing'))"
}
```

Free-lane previz: run the whole chain once at the cheapest grid first (Pollinations
lowest duration, Novita 720P/2s, local 480×854) to validate journey + seams, then
re-render finals — the same habit as the premium `mini` previz tier, at $0.

### 8e. Fallback cascades — the free tier manages itself

No single free provider survives contact with the catalog: models flip to `paid_only`,
quotas exhaust, hosts go 503. So nothing calls a provider directly — everything goes
through a cascade: an ordered list, first live+verified entry wins, per-clip failover,
resume-safe. Order = cheapest-reliable first; edit the order, never the runners.

```bash
STILLS_CASCADE="pollflux pollmirror hfspace cf hf sf local"
CHAIN_CASCADE="novita pollflf localflf"     # full A+B (needs end-frame)
LEG_CASCADE="pollaonly novita-first sflocal" # arch-A legs (start-frame only)

# live probe: is this Pollinations model free AND capable? (need = image|start_frame|end_frame)
# Prints ok:free | no:<reason> (paid_only, missing-cap, unknown-model, catalog-error).
# EXACT name match first — aliases can collide with :paid twins (verified 2026-09-15:
# community/MarcosFRG/flux-1-schnell vs its :paid twin). Verdicts append to retry.log.
_poll_catalog() { # refresh $WORK/poll_models.json if older than 1h
  if [ -f "$WORK/poll_models.json" ]; then
    age=$(( $(date +%s) - $(stat -f%m "$WORK/poll_models.json" 2>/dev/null || stat -c%Y "$WORK/poll_models.json") ))
    [ "$age" -lt 3600 ] && return 0
  fi
  curl -fsSL --max-time 20 https://gen.pollinations.ai/image/models -o "$WORK/poll_models.json" 2>/dev/null
}
poll_cap() { # model need
  _poll_catalog || { echo "no:catalog-error" | tee -a "$WORK/retry.log"; return 1; }
  WORK_DIR="$WORK" POLL_CAP_MODEL="$1" POLL_CAP_NEED="$2" python3 -c "
import json,os
want=os.environ['POLL_CAP_MODEL']; need=os.environ['POLL_CAP_NEED']; verdict='no:unknown-model'
try:
    ms=json.load(open(os.environ['WORK_DIR']+'/poll_models.json'))
    cand=[m for m in ms if m.get('name')==want] or [m for m in ms if want in (m.get('aliases') or [])]
    if cand:
        m=cand[0]
        if m.get('paid_only'): verdict='no:paid_only'
        elif need=='image' and m.get('category')=='image': verdict='ok:free'
        elif need in (m.get('video_capabilities') or []): verdict='ok:free'
        else: verdict='no:missing-cap'
except Exception: verdict='no:catalog-error'
print(verdict)" | tee -a "$WORK/retry.log"
}
# snap_dur <model> <wantSec> — clamp+round to the model's live duration grid
# (nova-reel-class takes 6s multiples; blind durations 400 — verified gap 2026-09-15)
snap_dur() { # model wantSec -> prints grid-snapped seconds
  _poll_catalog || { echo "$2"; return 0; }
  WORK_DIR="$WORK" POLL_SNAP_MODEL="$1" POLL_SNAP_WANT="$2" python3 -c "
import json,os
want=int(os.environ['POLL_SNAP_WANT']); out=want
try:
    for m in json.load(open(os.environ['WORK_DIR']+'/poll_models.json')):
        if m.get('name')==os.environ['POLL_SNAP_MODEL'] or os.environ['POLL_SNAP_MODEL'] in (m.get('aliases') or []):
            lo=m.get('min_duration') or 1; hi=m.get('max_duration') or want; step=m.get('duration_step') or 1
            out=max(lo,min(hi,want)); out=lo+round((out-lo)/step)*step; out=max(lo,min(hi,out)); break
except Exception: pass
print(out)"
}
need_key() { [ -n "$1" ]; }   # need_key "$POLLINATIONS_KEY" || continue
verify_still() { [ -s "$1" ] && ffprobe -v error -show_entries stream=width -of csv=p=0 "$1" >/dev/null 2>&1; }
comfy_live() { curl -fsSL --max-time 10 "$COMFY/system_stats" >/dev/null 2>&1; }

# pre-flight: print which lanes are alive (run at Step 0, show the user only live lanes)
detect_backends() {
  echo "== backend autodetect =="
  need_key "$POLLINATIONS_KEY" \
    && echo "pollinations: KEY (stills + video per live flags)" \
    || echo "pollinations: no key (keyless image may still work; VIDEO always needs a key — run probe_video)"
  python3 -c "import gradio_client" 2>/dev/null && echo "hfspace: READY (keyless schnell)" || echo "hfspace: pip install gradio_client"
  need_key "$CLOUDFLARE_API_TOKEN" && echo "cloudflare: READY" || echo "cloudflare: no token"
  need_key "$HF_TOKEN" && echo "huggingface: READY" || echo "huggingface: no token"
  need_key "$SILICONFLOW_API_KEY" && echo "siliconflow: READY" || echo "siliconflow: no key"
  need_key "$NOVITA_API_KEY" && echo "novita: READY (trial meter)" || echo "novita: no key"
  python3 -c "import diffusers, torch; assert torch.cuda.is_available()" 2>/dev/null \
    && echo "local: GPU READY" || echo "local: no GPU (lane unavailable)"
  comfy_live && echo "comfyui: LIVE ($COMFY)" || echo "comfyui: down"
  case "$(poll_cap flux image)" in ok*) echo "poll/flux: FREE+LIVE";; *) echo "poll/flux: DOWN/PAID (mirrors next)";; esac
}

# probe_video — MANDATORY 30-second Step 0 check, before any real work or frame
# uploads. Proves whether a video lane is actually callable (key-presence alone
# proves nothing — keyless video 401s). Burns one tiny upload, zero Pollen on 401.
probe_video() {
  ffmpeg -v error -y -f lavfi -i color=c=black:s=320x180:d=0.5 -frames:v 1 "$WORK/probe.jpg" || return 1
  tiny=$(curl -fsSL --max-time 60 -F "files[]=@$WORK/probe.jpg" https://uguu.se/upload.php | jq -r '.files[0].url // empty')
  [ -n "$tiny" ] || { echo "probe: frame host down — video lanes untestable"; return 1; }
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 60 --get "https://gen.pollinations.ai/video/probe" \
    --data-urlencode "model=${POLL_MODEL:-amazon/nova-reel-v1}" --data-urlencode "duration=6" \
    --data-urlencode "aspectRatio=16:9" --data-urlencode "image=$tiny" --data-urlencode "audio=false" \
    ${POLLINATIONS_KEY:+-H "Authorization: Bearer $POLLINATIONS_KEY"})
  case "$code" in
    200) echo "probe: video lane LIVE ($POLL_MODEL)" ;;
    401|403) echo "probe: video needs a key (HTTP $code) — chain unavailable keyless; stills+page or §8f previz only" ;;
    *) echo "probe: unclear (HTTP $code) — run the Step 4 qualification before batching" ;;
  esac
}

# run_still <name> <seed> — first verified PNG wins; finished clips are skipped (resume-safe)
run_still() {
  verify_still "$WORK/still_$1.png" && { echo "still $1 cached"; return 0; }
  for b in $STILLS_CASCADE; do
    case $b in
      pollflux)   need_key "$POLLINATIONS_KEY" || continue
                  case "$(poll_cap "${POLL_IMAGE_MODEL:-flux}" image)" in ok*) ;; *) continue;; esac
                  retry3 2 gen_still_poll "$1" "$2" || continue ;;
      pollmirror) need_key "$POLLINATIONS_KEY" || continue
                  for m in community/MarcosFRG/flux-1-schnell community/CloudCompile/flux-2-klein-4b community/CloudCompile/sdxl-lightning; do
                    case "$(poll_cap "$m" image)" in ok*) ;; *) continue;; esac
                    POLL_IMAGE_MODEL="$m" retry3 2 gen_still_poll "$1" "$2" && break
                  done ;;
      hfspace)    retry3 2 gen_still_hfspace "$1" "$2" || continue ;;
      cf)         need_key "$CLOUDFLARE_API_TOKEN" || continue
                  retry3 2 gen_still_cf "$1" || continue ;;
      hf)         need_key "$HF_TOKEN" || continue
                  retry3 2 gen_still_hf "$1" "$2" || continue ;;
      sf)         need_key "$SILICONFLOW_API_KEY" || continue
                  retry3 2 gen_still_sf "$1" "$2" || continue ;;
      local)      retry3 1 gen_still_local "$1" "$2" || continue ;;
    esac
    if verify_still "$WORK/still_$1.png"; then note "still_$1.png" "still:$b"; echo "still $1 ok (cascade: $b)"; return 0; fi
    echo "still $1 via $b unverified — failing over" >> "$WORK/retry.log"
  done
  echo "still $1 FAIL (cascade exhausted)"; return 1
}

# run_chain_clip <outMp4> <startPng> <endPng> <promptTxt> <dur> — connectors/legs with end-frames
run_chain_clip() {
  verify_clip "$1" "$5" 2>/dev/null && { echo "clip $1 cached"; return 0; }
  for b in $CHAIN_CASCADE; do
    case $b in
      novita) need_key "$NOVITA_API_KEY" || continue
              retry3 2 gen_novita "$1" "$2" "$3" "$4" "$5" 1080P || continue ;;
      pollflf) need_key "$POLLINATIONS_KEY" || continue
              case "$(poll_cap "$POLL_MODEL" end_frame)" in ok*) [ "$CAP" = "CHAIN" ] || continue;; *) continue;; esac
              retry3 2 gen_conn_poll_chain "$1" "$2" "$3" "$4" "$5" || continue ;;
      localflf) comfy_live || continue
              retry3 1 gen_flf_local "$1" "$2" "$3" "$4" || continue ;;
    esac
    if verify_clip "$1" "$5"; then note "$(basename "$1")" "chain:$b"; echo "clip $1 ok (cascade: $b)"; return 0; fi
    echo "clip $1 via $b unverified — failing over" >> "$WORK/retry.log"
  done
  echo "clip $1 FAIL (chain cascade exhausted — downgrade to arch A? confirm with user)"; return 1
}

# run_leg <outMp4> <startPng> <promptTxt> <dur> — arch-A legs (start-frame only backends)
run_leg() {
  verify_clip "$1" "$4" 2>/dev/null && { echo "leg $1 cached"; return 0; }
  for b in $LEG_CASCADE; do
    case $b in
      pollaonly) need_key "$POLLINATIONS_KEY" || continue
              case "$(poll_cap "$POLL_MODEL" start_frame)" in ok*) ;; *) continue;; esac
              retry3 2 gen_leg_poll "$1" "$2" "$3" "$4" || continue ;;
      novita-first) need_key "$NOVITA_API_KEY" || continue
              retry3 2 gen_novita "$1" "$2" "" "$3" "$4" 1080P || continue ;;
      sflocal) comfy_live || continue
              retry3 1 gen_flf_local "$1" "$2" "" "$3" || continue ;;
    esac
    if verify_clip "$1" "$4"; then note "$(basename "$1")" "leg:$b"; echo "leg $1 ok (cascade: $b)"; return 0; fi
    echo "leg $1 via $b unverified — failing over" >> "$WORK/retry.log"
  done
  echo "leg $1 FAIL (leg cascade exhausted)"; return 1
}
```

Two small adapters the cascades assume (single-purpose, no logic changes elsewhere):

```bash
# dive/leg over Pollinations with explicit out/start/prompt/dur (wraps §8b for the cascade)
gen_leg_poll() { # outMp4 startPng promptTxt dur
  furl=$(poll_frame_url "$2"); [ -n "$furl" ] || return 1
  curl -fsSL --get "https://gen.pollinations.ai/video/$(python3 -c "import urllib.parse;print(urllib.parse.quote(open('$3').read()))")" \
    --data-urlencode "model=$POLL_MODEL" --data-urlencode "duration=$4" \
    --data-urlencode "aspectRatio=$ASPECT" --data-urlencode "image=$furl" --data-urlencode "audio=false" \
    -H "Authorization: Bearer $POLLINATIONS_KEY" -o "$1" \
    && echo "leg ok (poll/$POLL_MODEL)" || { echo "leg FAIL"; return 1; }
}
# connector over Pollinations with explicit out/start/end/prompt/dur (CAP=CHAIN enforced)
gen_conn_poll_chain() { # outMp4 startPng endPng promptTxt dur
  [ "$CAP" = "CHAIN" ] || return 1
  su=$(poll_frame_url "$2"); eu=$(poll_frame_url "$3"); [ -n "$su" ] && [ -n "$eu" ] || return 1
  curl -fsSL --get "https://gen.pollinations.ai/video/$(python3 -c "import urllib.parse;print(urllib.parse.quote(open('$4').read()))")" \
    --data-urlencode "model=$POLL_MODEL" --data-urlencode "duration=$5" \
    --data-urlencode "aspectRatio=$ASPECT" --data-urlencode "image=$su|$eu" --data-urlencode "audio=false" \
    -H "Authorization: Bearer $POLLINATIONS_KEY" -o "$1" \
    && echo "conn ok (poll/$POLL_MODEL)" || { echo "conn FAIL"; return 1; }
}
```

Re-probe triggers (when the cascade re-checks the catalog instead of trusting cache):
a provider failing twice in a row, any `paid_only`/402/BLOCKED response, any 429 burst,
or a new build day. Cross-CAP downgrade (CHAIN→A-ONLY mid-build) is the one failover
that is NOT automatic — it changes the film's grammar, so confirm with the user first
(SKILL Step 4 rule); everything within a CAP fails over silently and is reported via
the manifest.

### 8f. Synthetic previz lane — $0, zero-dep, sanctioned (NOT final art)

When no video lane is live (or before spending anything anywhere), validate the whole
page — journey, pacing, seams, QA — with synthetic push-in stand-ins rendered from the
real stills. This is an explicit previz step, not a hack: same durations, same §5
scrub encodes, same engine wiring (`connectors: []`, arch A). The journey it validates
translates directly to the final render. Verified 2026-09-15: 8.0 s, 1920×1080,
~5 MB, gentle push-in (frame-diff 0.06). NEVER ship these as the final film without
telling the user they are synthetic — filename them `standin_*`, record them as such
in the manifest, and replace with the real chain when a lane goes live.

```bash
# standin <name> — slow push-in over the scene still, then the §5 scrub encode.
standin() { # name (uses $DIVE_DUR snapped to whole seconds)
  ffmpeg -v error -y -loop 1 -i "$WORK/still_$1.png" -vf \
    "scale=3072:-2,zoompan=z='1+0.06*on/($DIVE_DUR*24)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=$((DIVE_DUR*24)):s=1920x1080:fps=24" \
    -t "$DIVE_DUR" -c:v libx264 -pix_fmt yuv420p "$WORK/standin_$1.raw.mp4"
  enc "$WORK/standin_$1.raw.mp4" "$ASSETS/vid/$1.mp4"   # §5 encoder: -g 8, crf 20, faststart
}
for n in $NAMES; do standin "$n"; done
```

## Notes

- `.[0].result_url` is the field on the `--wait --json` job object. `.min_result_url` is
  a lower-res preview if you ever want it.
- **NSFW fallback across models**: if one clip keeps getting flagged on seedance after
  re-rolls + prompt scrubbing, regenerate just that clip on `kling3_0` with the SAME
  start/end frames: `VMODEL=kling3_0; VOPTS="--mode std --sound off"; gen_conn 3 …` —
  then restore your chain model. See SKILL Gotchas for the trade-off.
- **Previz on the cheap**: run the whole chain once with `VMODEL=seedance_2_0_mini`
  (frame-locking intact, ~720p) to validate the journey and seams before spending
  full-model credits — because it's still seamless, the previz translates directly to the
  final render. Don't reach for reference-only models here: without `--start/--end-image`
  they can't hold a seam, so their output can't be chained (Step 4 rule).
- If a whole batch stalls, check `higgsfield workspace list` for credits and
  `$WORK/*.err` for the reason.
- Concurrency: launching ~5–6 gens at once is fine; much more can trigger transient
  credit/race errors — stagger or re-roll.
