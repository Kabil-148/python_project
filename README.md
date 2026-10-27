# hello-flask

Tiny Flask app for practicing a Jenkins CI/CD pipeline.

## Run locally
    python3 -m venv venv && . venv/bin/activate
    pip install -r requirements.txt
    python app.py          # http://localhost:5000

## Test
    pytest --cov=app

## Endpoints
- `GET /`        Hello message
- `GET /health`  Health check (returns OK)
