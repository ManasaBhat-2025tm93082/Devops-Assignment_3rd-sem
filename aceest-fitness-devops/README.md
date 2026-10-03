# ACEest Fitness & Gym - DevOps CI/CD Project

A Flask web/API application for gym management (programs, calorie estimation, BMI,
clients, weekly adherence, workout logs, membership checks), packaged with Docker and
validated by **GitHub Actions** and **Jenkins**.

The app is the web port of the original Tkinter desktop versions kept in `legacy_versions/`.

## Project structure
```
app.py                  Flask application (app factory + routes + business logic)
tests/test_app.py       Pytest suite
requirements.txt        Runtime dependencies
requirements-dev.txt    + pytest, flake8
Dockerfile              Multi-stage: `base` (runtime) and `test` (adds pytest + tests)
Jenkinsfile             Jenkins pipeline
.github/workflows/main.yml   GitHub Actions pipeline
legacy_versions/        Original ACEest versions 1.0 - 3.2.4 (reference)
```

## Run locally
```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements-dev.txt
python app.py                     # http://localhost:5000
```
Try it:
```bash
curl http://localhost:5000/health
curl "http://localhost:5000/calories?program=Fat%20Loss%20(FL)&weight=80"
curl -X POST http://localhost:5000/clients -H "Content-Type: application/json" \
     -d "{\"name\":\"Ravi\",\"program\":\"Fat Loss (FL)\",\"weight\":80,\"height\":175}"
```

## API summary
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/programs`, `/programs/<name>` | Program list / details |
| GET | `/calories?program=&weight=` | Estimated daily calories |
| GET | `/bmi?weight=&height=` | BMI + category |
| POST/GET | `/clients`, `/clients/<name>` | Create / list / read client |
| DELETE | `/clients/<name>` | Delete client |
| GET | `/clients/<name>/bmi`, `/membership` | Client BMI, membership status |
| POST/GET | `/clients/<name>/progress` | Weekly adherence |
| POST/GET | `/clients/<name>/workouts` | Workout log |

## Run tests manually
```bash
pip install -r requirements-dev.txt
pytest -v
flake8 .
```

## Docker
```bash
docker build --target base -t aceest-fitness .
docker run -p 5000:5000 aceest-fitness           # http://localhost:5000

# run tests inside a container
docker build --target test -t aceest-fitness:test .
docker run --rm aceest-fitness:test
```
Image is based on `python:3.12-slim`, runs as a non-root user with gunicorn, uses layer
caching (dependencies installed before code is copied) and a `.dockerignore`.

## CI/CD overview

### GitHub Actions (`.github/workflows/main.yml`)
Triggered on every `push` and `pull_request`:
1. **Build & Lint** - install dependencies, `py_compile` syntax check, `flake8`.
2. **Docker Build & Test** - build the runtime image, build the test image, run `pytest` inside the container.

### Jenkins (`Jenkinsfile`)
Jenkins acts as the secondary build/quality gate. The job pulls the latest code from
GitHub, rebuilds a clean virtualenv, runs flake8 and pytest, then builds the Docker
images and runs the containerised tests. Any failing stage fails the build.

To configure the job:
New Item -> Pipeline -> "Pipeline script from SCM" -> Git -> repo URL -> branch `main`
-> Script Path `Jenkinsfile`.

## Git workflow used
- `main` is stable; work happens on `feature/*`, `bugfix/*`, `infra/*` branches merged via PR.
- Commit style: `feat:`, `fix:`, `test:`, `ci:`, `docker:`, `docs:`.
