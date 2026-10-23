# EasyGEN — Generative Visual Content and Editing Suite

EasyGEN is an academic prototype that combines a browser-based interface with six separate notebook-hosted AI services for generating and editing images and videos. The interface is in `app.html`; each model service is defined in its own Jupyter/Colab notebook. These are pretrained models, not models trained by this project.

> **Project status:** The repository contains frontend code and backend notebooks, not a unified deployable application. Each notebook must be configured and started separately. The notebooks and frontend have not been verified end-to-end as a clean install; see [Known implementation gaps](#known-implementation-gaps) before attempting a demo.

## Features

1. **Text to image** — Generate an image from a prompt using Stable Diffusion XL.
2. **Text to video** — Generate a short video from a prompt using the ModelScope text-to-video pipeline.
3. **Image to image** — Transform an uploaded image with Stable Diffusion 1.5 and a text prompt.
4. **Image to video** — Animate an uploaded image using Stable Video Diffusion.
5. **Advanced image transformation** — Transform an uploaded image using Realistic Vision V5.1 with controls for strength, guidance, steps, scheduler, and output size.
6. **Smart inpainting** — Use a painted mask for object removal, background replacement, or custom fill. The supplied backend uses LaMa inpainting.

Model output quality, generation time, and available settings depend on the selected model, hardware, runtime package versions, and backend implementation.

## Architecture

```text
Browser (app.html)
  ├── Text-to-image       → FastAPI notebook → Stable Diffusion XL
  ├── Text-to-video       → FastAPI notebook → ModelScope text-to-video
  ├── Image-to-image      → FastAPI notebook → Stable Diffusion 1.5
  ├── Image-to-video      → FastAPI notebook → Stable Video Diffusion
  ├── Advanced transform  → Flask notebook   → Realistic Vision V5.1
  └── Inpainting          → Flask notebook   → LaMa
```

The notebooks create local HTTP services and use ngrok tunnels to make them reachable by the browser. Tunnel URLs are temporary and must be taken from the current notebook session; the frontend URL settings are intentionally blank in the checked-in source.

## Repository contents

| File | Workflow / purpose |
| --- | --- |
| `app.html` | Single-page interface, upload/preview controls, and API requests to the six services. |
| `model1_IMAGE.ipynb` | Text-to-image API; SDXL; local port 8000. |
| `MODEL2_VIDEO.ipynb` | Text-to-video API; ModelScope text-to-video; local port 8001. |
| `MODEL3_IAMGETOIMAGE.ipynb` | Image-to-image API; Stable Diffusion 1.5; local port 8002. |
| `MODEL4.ipynb` | Image-to-video API; Stable Video Diffusion; local port 8004. |
| `MODEL5.ipynb` | Advanced image transformation API; Realistic Vision V5.1; local port 5000. |
| `MODEL6.ipynb` | LaMa inpainting API; local port 5001. |

Notebook filenames are preserved as supplied. Port numbers are notebook-local defaults; ensure there are no port conflicts in the runtime.

## API map

The frontend must call the matching running service. Image endpoints generally return binary PNG data; text-to-video returns an MP4 file. Image-to-video returns JSON containing a `video_url`, which the frontend then requests from the same service.

| Feature | Notebook | Method and path | Local port | Main request fields |
| --- | --- | --- | ---: | --- |
| Text to image | `model1_IMAGE.ipynb` | `POST /api/text2img` | 8000 | JSON: `prompt` |
| Text to video | `MODEL2_VIDEO.ipynb` | `POST /api/text2video` | 8001 | JSON: `prompt`, optional `num_frames`, `fps` |
| Image to image | `MODEL3_IAMGETOIMAGE.ipynb` | `POST /api/img2img` | 8002 | Multipart: `image`, `prompt`, optional `strength`, `guidance_scale` |
| Image to video | `MODEL4.ipynb` | `POST /api/image2video`; then `GET /video/{filename}` | 8004 | Multipart: `image`, optional `prompt`, `quality` |
| Advanced transformation | `MODEL5.ipynb` | `POST /api/img2img`; `GET /health` | 5000 | Multipart: `image`, `prompt`, `neg_prompt`, `strength`, `guidance_scale`, `num_steps`, `scheduler`, `width`, `height` |
| Inpainting | `MODEL6.ipynb` | `POST /api/inpaint`; `GET /health` | 5001 | Multipart: `image`, `mask`, `mode`, `mask_blur`, prompt and generation settings |

The two image-to-image APIs intentionally use the same route on different base URLs. The frontend calls them through separate URL variables.

## Requirements

- Jupyter Notebook or Google Colab; the notebooks are written in a Colab-oriented style.
- A GPU runtime is strongly recommended. The notebooks request CUDA and some model pipelines are memory-intensive; generation may fail or be impractically slow without a suitable GPU.
- Internet access to install Python packages and download pretrained model weights (some are several gigabytes).
- An ngrok account and an ngrok authentication token for public tunnel access.
- A modern browser for the frontend.

The repository does not include a shared `requirements.txt` or pinned environment specification. Each notebook installs dependencies independently, and those dependency versions may conflict when notebooks share one runtime. Use a fresh runtime for each backend and resolve installation errors there rather than assuming the notebooks are interchangeable.

## Setup and run

### 1. Open the frontend

Open `app.html` in a browser for a UI preview. For browser requests to remote APIs, serving it over HTTP is more reliable than opening it with a `file://` URL; for example, run a local static server from the project directory:

```bash
python -m http.server 5500
```

Then visit [http://localhost:5500/app.html](http://localhost:5500/app.html). This starts only the frontend; it does not start any AI backend.

### 2. Start each required backend

For every feature you plan to demo:

1. Open its corresponding notebook in Google Colab or Jupyter.
2. Use a fresh runtime, select a GPU where available, and run the dependency-installation cells.
3. Store your ngrok token in the runtime's secret manager under the name `NGROK_AUTHTOKEN`. In Colab, retrieve that secret into the environment before running the notebook's server cell (for example, use `from google.colab import userdata` and `os.environ["NGROK_AUTHTOKEN"] = userdata.get("NGROK_AUTHTOKEN")`). In a local Jupyter session, set the environment variable outside the notebook. Do not paste the token literal into notebook source or output.
4. Run the model loading and API cells in order. Model weights may take time and disk space to download.
5. Keep the server cell running and copy the current public tunnel URL printed by that session.
6. Confirm that the service is healthy (where `/health` is provided) and test the endpoint before using the UI.

The backend notebooks are separate processes. Colab runtimes may disconnect, recycle, run out of GPU memory, or issue a different tunnel URL after restart. There is no single command that starts all six services.

### 3. Connect the frontend to the services

After starting each service, edit the matching `BASE_URL_*` constant near the bottom of `app.html` to that service's current ngrok URL:

| Frontend setting | Workflow |
| --- | --- |
| `BASE_URL_IMG` | Text to image |
| `BASE_URL_T2V` | Text to video |
| `BASE_URL_IMG2IMG` | Image to image |
| `BASE_URL_I2V` | Image to video |
| `BASE_URL_GEMINI` | Advanced image transformation (name retained from the supplied code; this backend does not use Gemini) |
| `BASE_URL_INPAINT` | Inpainting |

Do not commit personal tunnel URLs or credentials. For a team/public deployment, replace these source constants with environment-specific configuration and add suitable access controls.

### 4. Try a workflow

Select a tool, provide its prompt and/or image, adjust available settings, and submit. Wait for inference to finish before retrying. Use small inputs and low-cost settings first. Generated outputs are returned by the backend and previewed in the browser; the UI also exposes downloads where supported.

## Known implementation gaps

- `MODEL4.ipynb` returns a relative video path in `video_url`; the frontend concatenates it to the tunnel base URL. Verify that this works with the current ngrok/runtime configuration.
- The inpainting frontend sends image, mask, prompt, negative prompt, guidance, steps, and blur, while the supplied `MODEL6.ipynb` handler primarily selects a mode and mask blur and runs LaMa. Prompt-driven modes and their UI controls are not fully wired to the backend as supplied.
- The notebook source includes broad-open CORS configurations and binds development services to all interfaces. Do not expose these endpoints publicly without authentication, upload limits, and origin restrictions.
- Frontend error handling is present but there is no unified backend health dashboard, retry policy, request timeout strategy, or automated integration test suite.
- Dependency installation is not reproducible across notebooks. The inpainting notebook has recorded package-install conflicts/errors in its outputs; a fresh run should be checked rather than trusting prior output cells.
- Several notebooks contain old execution outputs. These may describe a prior runtime and are not proof that a backend is currently online or working.
- GPU memory use and latency are high, particularly for video generation. Running several models concurrently may exceed available resources.
- A portable Windows virtual environment was present in the source archive but is not part of this repository; do not copy or commit virtual-environment directories.

## Security and model-use notes

- The supplied notebooks previously contained literal ngrok authentication tokens. Their source now reads `NGROK_AUTHTOKEN` from the environment instead. **Revoke/rotate every token that was previously present**; removing it from the current files does not invalidate the credential or erase copies from earlier backups.
- Do not commit API tokens, credentials, private tunnel URLs, or real user uploads. If any credential was committed elsewhere, rotate it and review Git history and shared copies.
- Uploaded images are sent to the selected backend. Use only images you have permission to process; do not upload sensitive/private content to public tunnels.
- Review the licenses and use restrictions for each pretrained model and dependency before redistribution or commercial use. Model weights may have separate terms from this repository.
- This is a prototype, not a hardened production service. Add authentication, rate limiting, input/file validation, resource quotas, and restrictive CORS before deployment
- 

**Team and My Role**
**Project title**: Generative Visual Content and Editing Suite (EasyGEN)

**Project type:** Team project (4 students)

**Institute:** Hi-Tech Institute of Technology

**Project guide:** Prof. S. P. Nikam

**Team members**: Prajakta Olekar, Priyanka Satpute, Sanskruti Wagh, and Gayatri Machhale.

**My role**: Contributed to the FastAPI backend and API integration support, connected the project components, and prepared the project report and documentation.

## License

No project-level `LICENSE` file is included. Confirm the team's intended license and add its text before redistribution. Also check the individual model licenses and dependency terms.
