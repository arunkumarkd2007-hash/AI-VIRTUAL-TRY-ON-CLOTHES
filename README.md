# Forma — AI-Based Virtual Clothing Try-On with Size Detection

Forma is a local-first college project with a responsive React website, FastAPI backend, editable demonstration size charts, optional MediaPipe pose landmarks, and an explicitly labeled silhouette preview fallback.

**Current capability:** the preview endpoint overlays a simple clothing silhouette on an uploaded photo. It is not a photorealistic try-on and does not use a generative model. No generative try-on weights are bundled or downloaded. Optional MediaPipe Pose landmark detection is separate: the first request that invokes it may download its small lite landmark model if the installed package does not already contain it. A separate generative model adapter can be added later without changing the size engine. Circumference recommendations use only tape measurements supplied by the user. Photo landmarks may estimate shoulder width and torso length when MediaPipe and a height value are available; those estimates are low confidence.

## Requirements

- Windows 11, PowerShell 5.1 or later
- Python 3.10–3.12 (tested project syntax target: Python 3.10)
- Node.js 20 or later and npm
- Internet access for initial package installation and the Google Fonts/Unsplash home page imagery. The API does not send personal photos to external services.

## Windows PowerShell setup

From the project root:

```powershell
py -3.10 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r backend\requirements.txt
```

If PowerShell blocks activation, use the venv Python directly:

```powershell
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r backend\requirements.txt
```

Install frontend packages:

```powershell
cd frontend
npm.cmd install
cd ..
```

The MediaPipe package is optional at runtime. If it cannot be installed for your Python version, remove the `mediapipe` line from `backend/requirements.txt` and install the other requirements; uploads and size recommendations still work, while photo landmark estimates report unavailable. When enabled, MediaPipe uses the lite CPU pose graph (no CUDA setup or GPU acceleration). Its source repository is Apache 2.0 licensed, and the [BlazePose GHUM 3D model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20BlazePose%20GHUM%203D.pdf) lists Apache 2.0; the [official model documentation](https://github.com/google-ai-edge/mediapipe/blob/master/docs/solutions/models.md) identifies the lite pose landmark asset. The framework may fetch this asset on first use. This is not a try-on generator.

## Run the app

Open two PowerShell terminals at the project root.

Terminal 1 — backend:

```powershell
.\.venv\Scripts\Activate.ps1
uvicorn backend.main:app --reload --host 127.0.0.1 --port 8000
```

Terminal 2 — frontend:

```powershell
cd frontend
npm.cmd run dev
```

Open <http://localhost:5173>. The API health endpoint is <http://localhost:8000/api/health>, and FastAPI's interactive docs are at <http://localhost:8000/docs>.

Optional frontend API override: copy `.env.example` to `frontend/.env.local`, then set `VITE_API_URL=http://localhost:8000`.

## Features

- Home, try-on studio, size guide, collection, and about views
- Responsive React + TypeScript interface with Framer Motion and Lucide icons
- Image validation by decoded content, accepted image formats, dimension checks, and 8 MB upload limit
- In-memory image handling; uploads are discarded when each API request ends
- Optional MediaPipe Pose estimates for shoulder width and torso length, scaled by user-entered height
- No circumference fabrication; missing circumference estimates stay unset
- Transparent size selection from JSON charts, with fit preference, alternatives, and warnings
- Save/download of the labeled silhouette output
- CORS restricted to the local Vite development origins

## API

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/api/health` | Service and current preview mode |
| GET | `/api/clothing` | Catalog; optional `?category=tops` |
| GET | `/api/size-charts` | Editable chart data |
| POST | `/api/upload/person` | Validate an image and optionally estimate landmarks; multipart `file`, optional `height_cm` |
| POST | `/api/measurements/estimate` | Estimate available photo proportions; multipart `file`, optional `height_cm` |
| POST | `/api/size/recommend` | Recommend from tape measurements; JSON `measurements`, `category`, `fit`, optional `brand` |
| POST | `/api/tryon/generate` | Return a labeled silhouette preview; multipart `file`, `garment_id` |

## Size charts

Edit [`data/size_charts.json`](data/size_charts.json) to change the demonstration ranges or add brands/categories. Chart values are body measurement ranges in centimetres and are explicitly illustrative, not official retailer charts. `Form House` is synthetic comparison data shifted by 2 cm from `Studio Basics`, so the brand selector demonstrates how chart differences affect results. The recommender supports XS through XXL and uses category-relevant chest, waist, and/or hip tape measurements. With no relevant measurements it returns no recommendation. Fit preference only breaks a tie when measurements match more than one adjacent range (slim favors the smaller size, loose the larger); treat results as guidance, not fit guarantees.

## Genuine AI try-on extension

The current app does not bundle a generative virtual try-on network, checkpoint, or model download. The `backend/tryon_engine.py` function is the replacement seam. Integrate a separately reviewed compatible model there, keep its weights outside source control, and update the API `mode` and UI wording only after verified inference. This project currently makes no claims about a specific model's license, VRAM needs, or CUDA compatibility because none is integrated. Do not send photos to a hosted model without explicit user consent.

For the optional MediaPipe pose pipeline, Python imports the package locally and processes an in-memory image. On first use, the official MediaPipe solution may fetch the pose landmark lite model if it is not present in the package; pose model weights are not included in this repository. It detects pose points but cannot infer accurate body circumferences from one ordinary photo. This configuration uses CPU; the app does not require an NVIDIA driver or CUDA toolkit. The generated clothing silhouette does not invoke MediaPipe.

## Tests and build

Run backend checks from the project root:

```powershell
.\.venv\Scripts\Activate.ps1
python -m pytest
```

Build frontend assets:

```powershell
cd frontend
npm.cmd run build
```

## Troubleshooting

- **Port is occupied:** choose another Uvicorn port and set `VITE_API_URL` to match; restart Vite after changing its `.env.local`.
- **PowerShell script policy blocks activation:** use `.\.venv\Scripts\python.exe` directly as shown above.
- **MediaPipe install fails:** use Python 3.10–3.12 and update pip, or omit MediaPipe; the app will report photo landmark estimates unavailable. The other endpoints do not depend on it.
- **CUDA or RTX 4050:** no CUDA path is enabled in this implementation; CPU is expected. There are no generative try-on weights. Optional MediaPipe Pose may download its lite landmark model on first use, from the source documented above.
- **Upload rejected:** use JPEG, PNG, or WebP under 8 MB, at least 160×240 pixels, and a clear front-facing image. The API validates decoded image data rather than trusting the filename.
- **API connection error:** start FastAPI on port 8000 and Vite on port 5173. Verify `/api/health` and check `VITE_API_URL`.
- **Photo estimates missing:** install MediaPipe, provide height, and use a well-lit full-body photo. Circumference estimates intentionally remain blank.
- **Fonts or home photography unavailable:** the page uses browser serif/system fallbacks; sample garment visuals are generated with CSS and the core workflow remains local.

## Privacy

Uploaded photos are sent only to the local FastAPI process at the configured API URL, decoded in memory, and not saved to disk. The frontend preview uses a browser object URL and includes a reset action. Home-page editorial photography and web fonts load from external public CDNs; uploaded photos are never included in those requests. If you host the API remotely, update this disclosure and obtain informed consent before transmitting personal photos.
