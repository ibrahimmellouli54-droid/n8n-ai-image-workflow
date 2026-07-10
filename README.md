# 🎨 AI Image Generation Workflow — n8n + Ollama + Pollinations + SD Forge

> n8n workflow for AI image generation with automatic prompt optimization, multi-criteria critique, and an iterative improvement loop.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Workflow Architecture](#workflow-architecture)
- [Requirements](#requirements)
- [Step-by-Step Installation](#step-by-step-installation)
- [Credentials Setup](#credentials-setup)
- [Importing the Workflow](#importing-the-workflow)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [Useful Links](#useful-links)

---

## Overview

This workflow automates high-quality image generation through a complete pipeline:

1. **Input**: the user sends a text prompt (and optionally a reference image) via the n8n chat
2. **Detection & analysis**: the workflow checks whether an image was attached; if so, Qwen2.5-VL analyzes it and merges its content into the prompt
3. **Initial generation**: 3 images are generated in parallel via Pollinations.ai (default, Flux, and Turbo models)
4. **Automatic critique**: each image is scored by Qwen2.5-VL on 3 criteria (fidelity, aesthetic, technical), combined into a weighted score: `fidelity × 0.5 + technical × 0.2 + aesthetic × 0.3`
5. **Iterative optimization**: if the best score is ≤ 9/10 and fewer than 3 loops have run, Qwen3 rewrites the 3 prompts based on their critiques and generates 3 new images (up to 3 loops total)
6. **Final rendering**: once a score > 9/10 is reached (or the loop limit is hit), an AI Agent (DeepSeek-R1) produces the final generation code, which is sent to Stable Diffusion Forge (`img2img`) for the final render, returned in the chat

---

## Workflow Architecture

```
[Tchat (Chat Trigger)]
      │
      ▼
[Détecteur] → [If: has_image?]
      │                    │
      NO                  YES
      │                    ▼
      │            [base64] → [Qwen2.5 analyse] → [Fusion]
      │                                                │
      ▼                                                │
[Prompt Engineer Sans image] ←─────────────────────────┘
      │
      ▼
[URL maker] → [Gen1 (default)] ┐
              [Gen2 (Flux)]     ├─> [Base64 GenX] → [Prompt CritiqueX] → [CritiqueX (Ollama)] → [Extrac NOTEX]
              [Gen3 (Turbo)]   ┘
                                                     │
                                                     ▼
                                             [Merge] → [Stock compteur] → [Comparaison]
                                                     │
                                      best_score > 9 OR loop_count == 3?
                                            │                  │
                                           YES                 NO
                                            │                  │
                                            │           [Prompt Optimizer (Qwen3 III)] → [Structure Output]
                                            │                  │
                                            │           [URL maker new Gen] → [Gen4/5/6 + Critique] ──> loop back
                                            │
                                            ▼
                                      [AI Agent (DeepSeek-R1)]
                                            │
                                            ▼
                                [Code in JavaScript] → [HTTP Request → SD Forge /sdapi/v1/img2img]
                                            │
                                            ▼
                                      [Final image]
```

---

## Requirements

| Component | Minimum Version | Role |
|---|---|---|
| **n8n** | ≥ 1.40.0 | Workflow orchestration |
| **Docker** | ≥ 24.0 | n8n containerization |
| **Ollama** | ≥ 0.3.0 | Local LLM server |
| **Stable Diffusion WebUI Forge** | latest | Final img2img rendering |
| **GPU** | 8 GB VRAM (recommended) | Ollama + SD Forge inference |

---

## Step-by-Step Installation

### Step 1 — Install Ollama

**Linux / macOS:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**Windows:** download the installer from [https://ollama.com/download](https://ollama.com/download)

Check that the service is running:
```bash
ollama --version
```

#### Download the required models

```bash
# Prompt engineering and optimization model (8B)
ollama pull qwen3:latest

# Vision model for image analysis and critique (7B)
ollama pull qwen2.5vl:7b

# Reasoning model for the final generation step
ollama pull deepseek-r1:latest
```

> ⚠️ The 3 models together take up roughly **15–20 GB** of disk space.

Check installed models:
```bash
ollama list
```

---

### Step 2 — Install n8n via Docker

```bash
docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -v n8n_data:/home/node/.n8n \
  --add-host=host.docker.internal:host-gateway \
  docker.n8n.io/n8nio/n8n
```

> 🔑 **Important**: the `--add-host=host.docker.internal:host-gateway` flag is required. It lets the n8n container communicate with Ollama and SD Forge running on the host machine.

Access n8n at: [http://localhost:5678](http://localhost:5678)

---

### Step 3 — Install Stable Diffusion WebUI Forge

**Clone the repository:**
```bash
git clone https://github.com/lllyasviel/stable-diffusion-webui-forge.git
cd stable-diffusion-webui-forge
```

**Launch Forge with the API enabled:**
```bash
# Linux / macOS
./webui.sh --api --listen

# Windows
webui-user.bat
# Add --api to COMMANDLINE_ARGS in webui-user.bat
```

Check that the API is reachable:
```
http://localhost:7860/sdapi/v1/sd-models
```

#### Download a model for SD Forge

Place a `.safetensors` model in the `models/Stable-diffusion/` folder.

Recommended for realistic rendering:
- **Realistic Vision V6**: [https://civitai.com/models/4201](https://civitai.com/models/4201)

---

### Step 4 — Verify connectivity

From your terminal, verify that n8n can reach Ollama and SD Forge:

```bash
# Test Ollama
curl http://localhost:11434/api/tags

# Test SD Forge API
curl http://localhost:7860/sdapi/v1/sd-models
```

If you're running Docker on Linux, test with `host.docker.internal`:
```bash
docker exec n8n curl http://host.docker.internal:11434/api/tags
docker exec n8n curl http://host.docker.internal:7860/sdapi/v1/sd-models
```

---

## Credentials Setup

### Ollama credential in n8n

1. In n8n, go to **Settings → Credentials → New Credential**
2. Search for **Ollama**
3. Fill in:
   - **Base URL**: `http://host.docker.internal:11434`
4. Save it as: `Ollama account`

This credential is used by the following nodes: `Qwen3 I`, `Qwen3 II`, `Qwen3 III`, `Ollama Chat Model`.

> ℹ️ No credential is required for **Pollinations.ai** — the API is public and free.
> ℹ️ No credential is required for **SD Forge** — the local API doesn't require authentication by default.

---

## Importing the Workflow

1. Open n8n: [http://localhost:5678](http://localhost:5678)
2. Go to **Workflows** → **+** button → **Import from file**
3. Select the file `Final_version.json`
4. Click **Import**
5. Check that the Ollama credentials are properly linked to the relevant nodes
6. Save the workflow (Ctrl+S)
7. Activate the workflow with the toggle in the top right

---

## Usage

### Starting the chat

1. Open the **Tchat** node (Chat Trigger) in the editor
2. Click **Open Chat** to open the chat interface
3. Or access it via the workflow's production link

### Text-only mode

Simply type your description in the chat:
```
A red fox sitting on a rock at sunset, photorealistic style
```

### Text + reference image mode

Attach an image in the chat (the workflow automatically detects it) and describe the scene you want:
```
[attached image: photo of a red cat]
Place this cat in an enchanted forest at dusk
```

### What happens next

- The workflow generates **3 images in parallel** using different Pollinations models (default, Flux, Turbo)
- Each image is **automatically scored** by the AI (Fidelity / Aesthetic / Technical, combined into a weighted score)
- If the best score is **9/10 or lower**, prompts are **rewritten based on their critiques** and new images are generated (up to 3 loops total)
- Once the score threshold is reached (or the loop limit is hit), the best result goes through an **AI Agent (DeepSeek-R1)** and **SD Forge img2img** for the final render
- The final image is returned in the chat

---

## Troubleshooting

| Issue | Likely Cause | Solution |
|---|---|---|
| `Connection refused` on Ollama | Ollama isn't running | `ollama serve` |
| `host.docker.internal` not resolving | Docker on Linux without the flag | Add `--add-host=host.docker.internal:host-gateway` to `docker run` |
| SD Forge unreachable | Forge launched without `--api` | Restart with `--api --listen` |
| Ollama model not found | Model not downloaded | `ollama pull <model_name>` |
| Generation timeout | Models too heavy for the GPU | Reduce `max_tokens` or use lighter models |
| Ollama credential not linked | Partial import | Manually re-assign the credential in each Ollama node |

---

## Useful Links

| Resource | URL |
|---|---|
| n8n Documentation | [https://docs.n8n.io](https://docs.n8n.io) |
| n8n Docker Hub | [https://hub.docker.com/r/n8nio/n8n](https://hub.docker.com/r/n8nio/n8n) |
| Ollama — Official site | [https://ollama.com](https://ollama.com) |
| Ollama — Model library | [https://ollama.com/library](https://ollama.com/library) |
| Qwen3 on Ollama | [https://ollama.com/library/qwen3](https://ollama.com/library/qwen3) |
| Qwen2.5-VL on Ollama | [https://ollama.com/library/qwen2.5vl](https://ollama.com/library/qwen2.5vl) |
| DeepSeek-R1 on Ollama | [https://ollama.com/library/deepseek-r1](https://ollama.com/library/deepseek-r1) |
| Pollinations.ai API | [https://pollinations.ai](https://pollinations.ai) |
| SD WebUI Forge GitHub | [https://github.com/lllyasviel/stable-diffusion-webui-forge](https://github.com/lllyasviel/stable-diffusion-webui-forge) |
| Realistic Vision (CivitAI) | [https://civitai.com/models/4201](https://civitai.com/models/4201) |

---

## Ollama Models Used

| n8n Node | Model | Usage |
|---|---|---|
| Qwen3 I, II, III | `qwen3:latest` | Prompt engineering, merging, optimization |
| Ollama Chat Model | `deepseek-r1:latest` | Final generation / img2img instructions for SD Forge |
| Critique1 / Critique2 / Critique3, Qwen2.5 analyse | `qwen2.5vl:7b` | Visual analysis + image scoring |

---

*Workflow built with [n8n](https://n8n.io) — open-source automation platform.*
