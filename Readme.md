# ComfyUI on Kaggle — Free GPU Image Generation

Run [ComfyUI](https://github.com/Comfy-Org/ComfyUI) (a node-based AI image/video generation UI) entirely inside a free Kaggle notebook using a T4 GPU. No local GPU needed. Access it from your browser via a public tunnel URL.

---

## What's in this repo

| File | Purpose |
|---|---|
| `comfy-kaggle-text-to-image.ipynb` | Text → Image using **Z-Image-Turbo** + Lyara LoRA |
| `comfy-kaggle-image-to-image.ipynb` | Image → Edited Image using **Wan 2.2 ChronoEdit 14B** |
| `comfy-kaggle-video-generation.ipynb` | Text → Video and Image → Video using **Wan 2.2 5B** |
| `magnetic_lion_anime_lora_api.json` | Example workflow for the text-to-image notebook |
| `chronoedit_14b_distill_api.json` | Example workflow for the image-to-image notebook |
| `wan22_5b_text2video_api.json` | Text → Video workflow for the video generation notebook |
| `wan22_5b_img2video_api.json` | Image → Video workflow for the video generation notebook |
| `prompts.txt` | Sample prompts for the Lyara anime LoRA |
| `output/` | Sample generated images and videos |

---

## Prerequisites

- A free [Kaggle](https://www.kaggle.com) account
- A free [Hugging Face](https://huggingface.co) account (for model downloads)
- Internet access enabled on the Kaggle notebook (see setup below)

---

## One-time Kaggle setup

**1. Enable GPU and Internet on your notebook**
- Open the notebook on Kaggle
- On the right panel: Accelerator → **GPU T4 x1**
- Also enable **Internet** (required to download models)

**2. Add your Hugging Face token as a Kaggle Secret**
- Go to [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens) → create a token with **Read** access
- In Kaggle: Settings → Secrets → Add new secret
  - Name: `HF_TOKEN`
  - Value: your token

---

## How to run

**Run all cells top to bottom.** Each cell is numbered and self-explanatory.

1. **Cell 1** — Installs system deps, clones ComfyUI + ComfyUI-Manager, installs Python requirements
2. **Cell 2** — Storage hack: moves `models/`, `input/`, `output/` to `/tmp` (which has more space on Kaggle) and symlinks them back. This avoids the 20GB `/kaggle/working` disk limit.
3. **Cell 3** — Loads your `HF_TOKEN` from Kaggle Secrets
4. **Cell 4** — Installs `aria2` (fast multi-connection downloader)
5. **Cell 5** — Downloads all required model files from Hugging Face using `aria2c -x 16` (16 parallel connections)
6. **Last cell** — Starts ComfyUI on port 8188, then opens a [Pinggy](https://pinggy.io) SSH reverse tunnel. Prints a public HTTPS URL like `https://xxxx.free.pinggy.net`

Open that URL in your browser → ComfyUI is running.

> **Important:** Keep the last cell running. If it stops, the tunnel dies. The free Pinggy tunnel expires after 60 minutes — just re-run the last cell to get a new URL.

---

## Loading a workflow

Once ComfyUI is open in your browser:

1. Drag and drop one of the `.json` files onto the ComfyUI canvas, **or** click the menu → Load → select the file
2. If any nodes show as red/missing, click **Manager** → **Install Missing Custom Nodes** → restart ComfyUI (re-run the last cell)
3. Click **Queue Prompt** to generate

---

## Notebooks & models explained

### Video Generation (`comfy-kaggle-video-generation.ipynb`)

Generates short MP4 videos from either a text prompt alone or a starting image + text prompt. Uses the **Wan 2.2 5B** text-image-to-video model. This is the lightest of the three notebooks — the 5B model fits comfortably in the T4's 16 GB VRAM at fp16.

**Models downloaded:**
| Model | File | Role |
|---|---|---|
| Wan 2.2 5B diffusion model | `wan2.2_ti2v_5B_fp16.safetensors` (~9.3 GB) | Main video generator |
| Wan 2.2 VAE | `wan2.2_vae.safetensors` (~1.3 GB) | Encodes/decodes video frames |
| UMT5-XXL text encoder | `umt5_xxl_fp8_e4m3fn_scaled.safetensors` (~6.2 GB) | Understands your prompt |

**Workflows:**

`wan22_5b_text2video_api.json` — Text → Video
- Generates a 33-frame video at 832×480 from a text prompt only
- Sampler: `uni_pc`, 20 steps, CFG: `5.0`
- Edit the **Positive Prompt** node with your scene description
- Edit the **Negative Prompt** node to exclude unwanted elements

`wan22_5b_img2video_api.json` — Image → Video
- Same as above but also loads a starting image (`start.png`) from ComfyUI's input folder to anchor the first frame
- Upload your image to ComfyUI's input folder and name it `start.png`, or change the filename in the **Load Image** node
- The video animates from that image according to your prompt

**Both workflows output** a 33-frame MP4 at 24 fps (~1.4 seconds). Outputs are saved to `output/video/`.

> Note: CFG `5.0` and 20 steps is used here (unlike the distilled image models which use CFG `1` and 8 steps). The Wan 2.2 5B video model is not distilled, so it benefits from higher CFG and more steps.

---

### Text-to-Image (`comfy-kaggle-text-to-image.ipynb`)

Generates images from a text prompt.

**Models downloaded:**
| Model | File | Role |
|---|---|---|
| Z-Image-Turbo diffusion model | `z_image_turbo_bf16.safetensors` (~11 GB) | Main image generator |
| Qwen 3 4B text encoder | `qwen_3_4b.safetensors` (~7.4 GB) | Understands your prompt |
| AE VAE | `ae.safetensors` (~320 MB) | Decodes latent → pixel image |
| Lyara anime LoRA | `lyara-lora.safetensors` (~163 MB) | Anime character style fine-tune |

**Workflow (`magnetic_lion_anime_lora_api.json`):**
- Loads the diffusion model + Lyara LoRA at strength `0.7`
- Generates a 1024×1024 image in **8 steps** (fast, distilled model)
- Sampler: `res_multistep`, CFG: `1` (distilled models don't use CFG > 1)
- To use your own prompt: open the workflow, edit the **Positive Prompt** node

**Sample prompts** are in `prompts.txt` — all use the `lyara` trigger word to activate the LoRA.

---

### Image-to-Image (`comfy-kaggle-image-to-image.ipynb`)

Takes an input image and edits it based on a text description. Uses Wan 2.2's ChronoEdit model which is designed for image editing via a video diffusion trick (generates a short video, takes the last frame as the edited image).

**Models downloaded:**
| Model | File | Role |
|---|---|---|
| ChronoEdit 14B diffusion model | `chrono_edit_14B_fp16.safetensors` (~30 GB) | Main editing model |
| ChronoEdit distill LoRA | `chronoedit_distill_lora.safetensors` (~358 MB) | Speeds up sampling (distillation) |
| UMT5-XXL text encoder | `umt5_xxl_fp8_e4m3fn_scaled.safetensors` (~6.2 GB) | Understands edit instructions |
| Wan 2.1 VAE | `wan_2.1_vae.safetensors` (~242 MB) | Encodes/decodes images |
| CLIP Vision H | `clip_vision_h.safetensors` (~1.2 GB) | Encodes the input image visually |

**Workflow (`chronoedit_14b_distill_api.json`):**
- Loads input image from `image_chrono_edit_input_image.png` (upload this to ComfyUI's input folder)
- Runs `WanImageToVideo` to generate a 5-frame video at 720×720
- Takes frame index 4 (the last frame) as the output edited image
- Sampler: `uni_pc`, 8 steps, CFG: `1`
- To use: edit the **Positive Prompt** node with a description of what you want the final image to look like

---

## Technical notes

### Storage hack (why it's needed)
Kaggle gives `/kaggle/working` only ~20 GB. The models alone are 40+ GB. The notebooks move `models/`, `input/`, and `output/` to `/tmp` which has ~100 GB, then create symlinks so ComfyUI still finds them at the expected paths.

```
/kaggle/working/ComfyUI/models  →  /tmp/ComfyUI_Storage/models
/kaggle/working/ComfyUI/input   →  /tmp/ComfyUI_Storage/input
/kaggle/working/ComfyUI/output  →  /tmp/ComfyUI_Storage/output
```

### Why Pinggy instead of ngrok?
ngrok requires authentication tokens. Pinggy works over plain SSH with no account needed — just `ssh -R0:127.0.0.1:8188 a.pinggy.io`. The free tier gives a 60-minute tunnel.

### Why aria2c?
`aria2c -x 16 -s 16` opens 16 parallel connections per file, saturating Kaggle's ~300–400 MB/s HuggingFace bandwidth. A 30 GB model downloads in ~2 minutes instead of 20+.

### Model precision
- `bf16` / `fp16` = full precision, best quality, more VRAM
- `fp8_e4m3fn` = 8-bit quantized, fits in T4's 16 GB VRAM with large models

---

## Troubleshooting

**"ComfyUI never came up"** — Check `/kaggle/working/comfy.log`. Usually a missing dependency. Re-run Cell 1.

**Red/missing nodes in workflow** — Open ComfyUI Manager → Install Missing Custom Nodes → restart (re-run last cell).

**Tunnel URL not appearing** — Wait 30–60 seconds after "ComfyUI up" prints. The Pinggy log is at `/kaggle/working/pinggy.log`.

**Out of disk space** — Make sure Cell 2 (storage hack) ran successfully before Cell 5 (model downloads).

**Session expired / models gone** — Kaggle sessions reset after ~9 hours or on kernel restart. `/tmp` is wiped. You need to re-run all cells from the start. Models re-download automatically (aria2c skips files that already exist and are > 1 MB, but `/tmp` is gone so they'll re-download).
