# VisionDesk AI

AI-powered PPE (Personal Protective Equipment) detection and safety compliance monitoring, built with Flask, YOLOv8, and MongoDB.

**Live demo:** https://visiondesk-ai.onrender.com

> Hosted on Render's free tier. The app sleeps after ~15 minutes of no traffic, so the first visit after an idle period takes 30-60 seconds to wake up. After that it responds normally.

## Features

Verified working on the live deployment (Nov 2026): admin login, image PPE detection, video PPE detection, and the compliance dashboard with detection history.

- **PPE detection**: upload images or short videos (JPG, PNG, MP4, AVI, MOV, up to 50MB) and detect workers, helmets, and high-visibility vests with a custom-trained YOLOv8 model. Missing gear is flagged as a violation with an annotated result image.
- **Compliance dashboard**: audit counts, violation stats, compliance rate, and detection history.

Also implemented in the codebase but not yet exercised on the live deployment:

- **Documents AI**: upload PDF/DOCX/TXT safety documents for knowledge extraction and search.
- **AI Assistant**: RAG chat over uploaded documents and compliance data. Needs an `OPENAI_API_KEY` or `GOOGLE_API_KEY` env var; no key is configured on the current deployment, so the assistant is not active there.
- **Reports**: export compliance and knowledge reports as PDF.
- **Alerts and analytics**: track and resolve safety alerts over time.

## Tech stack

- Flask 2.3 + gunicorn (production server)
- YOLOv8 (ultralytics) with a custom-trained PPE checkpoint (`ppe_yolov8.pt`, included in this repo)
- PyTorch (CPU build in production)
- OpenCV (headless) for image annotation and video frame processing
- MongoDB Atlas (pymongo) for users, detection records, and documents
- bcrypt password hashing, CSRF protection, rate limiting, secure session cookies

## Deployment (Render free tier)

The app is deployed as a Render web service (Python 3.10, 512MB free instance).

- **Build command:** `pip install torch==2.9.1+cpu torchvision==0.24.1+cpu --extra-index-url https://download.pytorch.org/whl/cpu && pip install -r requirements.txt`
  (CPU-only torch is installed first so the free instance stays under its memory limit; the default CUDA build is too large for 512MB.)
- **Start command:** `gunicorn -w 1 -k gthread --threads 1 -t 300 app:app`
- **Environment variables:** `PYTHON_VERSION=3.10`, `SECRET_KEY`, `MONGO_URI` (MongoDB Atlas connection string), `MONGO_DB=visiondesk_db`

Note: pushing to GitHub does not auto-deploy this service. Deploy from the Render dashboard with Manual Deploy > Deploy latest commit.

## Deployment notes

Changes made for hosting:

- `opencv-python` swapped for `opencv-python-headless` (no GUI libraries on a server)
- gunicorn added for production serving
- The trained checkpoint loads through an explicit allowlist of standard YOLOv8 classes, keeping torch's safe loading (weights_only) active. If the checkpoint ever fails to load, the app falls back to the generic YOLOv8n model, which is COCO-pretrained and does NOT detect PPE classes (workers, helmets, vests) - detection results would no longer be PPE-specific. The deployed instance loads the custom checkpoint successfully (confirmed in the boot log).
- Fixed a bug where uploaded images were saved without detection annotations: the annotated frame is now written explicitly with `cv2.imwrite` instead of the ultralytics `Results.save` call, which saved the unannotated original in this setup.
- MongoDB connection logging now redacts credentials (the full connection string, including the database password, was printed to logs by earlier revisions).

## Known limitations (free hosting)

- **Cold starts**: first request after ~15 min idle takes 30-60s (free tier sleep).
- **Ephemeral uploads**: files on the server's disk (uploads, result images) disappear on restart or redeploy. Detection records and stats persist in MongoDB.
- **Memory**: 512MB instance. Image inference and short videos work well; long or near-50MB videos may be slow or fail.
- **AI Assistant**: not active on the current deployment (no LLM API key configured).
- Single worker by design (free tier): one long request can briefly delay others.

## Run locally

```bash
pip install -r requirements.txt
export MONGO_URI="mongodb://localhost:27017/"   # or your Atlas URI
export MONGO_DB="visiondesk_db"
export SECRET_KEY="change-me"
python app.py
```

Then open http://127.0.0.1:5000. On first run with an empty database the app creates a default admin (admin / safety2026) - change this immediately. The deployed instance uses a pre-seeded strong admin password instead.

## Model

`ppe_yolov8.pt` is a YOLOv8 checkpoint custom-trained on a PPE dataset (workers, helmets, vests). Training code is in `train.py`; dataset config in `ppe_dataset/`.

## Security

- Passwords stored as bcrypt hashes
- CSRF protection and rate limiting enabled
- Session cookies: Secure, HttpOnly, SameSite=Lax
- Deployment secrets (database credentials, keys) come from environment variables
- The repo still contains the local first-run default admin credentials in `app.py` (see Run locally) - change them before any real use
