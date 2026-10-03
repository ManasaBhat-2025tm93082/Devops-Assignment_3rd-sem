# ACEest Fitness & Gym - Automated CI/CD Pipeline

A Flask web application for gym and fitness management, delivered through an automated
DevOps workflow: Git/GitHub version control, Pytest unit tests, Docker containerization,
a GitHub Actions CI pipeline and a Jenkins build.

**Repository:** https://github.com/ManasaBhat-2025tm93082/Devops-Assignment_3rd-sem

---

## 1. Overview

ACEest Fitness & Gym started as a series of Tkinter desktop applications (versions 1.0 to
3.2.4, kept in `legacy_versions/`). This project ports the core business logic into a
Flask web application with a browser UI and a JSON API, then wraps it in a CI/CD workflow.

**Application features**

- Training programs (Fat Loss, Muscle Gain, Beginner) with workout and diet plans
- Daily calorie estimation per program and body weight
- BMI calculation with category (Underweight, Normal, Overweight, Obese)
- Client management (add, list, view, delete) stored in SQLite
- Weekly adherence (progress) tracking with averages
- Workout logging per client
- Membership status check (Active / Expired)

## 2. Project Structure

```
.
|-- app.py                       Flask application (app factory, routes, business logic)
|-- templates/index.html         Browser UI
|-- tests/test_app.py            Pytest unit tests
|-- requirements.txt             Runtime dependencies
|-- requirements-dev.txt         Runtime + pytest + flake8
|-- pytest.ini                   Pytest configuration
|-- .flake8                      Lint configuration
|-- Dockerfile                   Multi-stage image (base runtime + test)
|-- .dockerignore                Keeps the image small
|-- Jenkinsfile                  Jenkins pipeline (Windows agent, no Docker)
|-- Jenkinsfile.linux-docker     Optional Jenkins pipeline for Linux agents with Docker
|-- .github/workflows/main.yml   GitHub Actions CI pipeline
|-- legacy_versions/             Original ACEest desktop versions 1.0 - 3.2.4
`-- README.md
```

## 3. Local Setup and Execution

**Prerequisites:** Python 3.10 or newer and Git.

**Windows (Command Prompt)**

```bat
git clone https://github.com/ManasaBhat-2025tm93082/Devops-Assignment_3rd-sem.git
cd Devops-Assignment_3rd-sem
python -m venv venv
venv\Scripts\activate
python -m pip install -r requirements-dev.txt
python app.py
```

**Linux / macOS**

```bash
git clone https://github.com/ManasaBhat-2025tm93082/Devops-Assignment_3rd-sem.git
cd Devops-Assignment_3rd-sem
python3 -m venv venv
source venv/bin/activate
python -m pip install -r requirements-dev.txt
python app.py
```

Open **http://localhost:5000** to use the web UI. The port can be changed with the `PORT`
environment variable, and the database location with `ACEEST_DB`.

### API Reference

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/` | Web UI |
| GET | `/api` | API information |
| GET | `/health` | Health check |
| GET | `/programs` | List programs |
| GET | `/programs/<name>` | Program workout, diet and calorie factor |
| GET | `/calories?program=&weight=` | Estimated daily calories |
| GET | `/bmi?weight=&height=` | BMI and category |
| POST | `/clients` | Create a client (JSON body) |
| GET | `/clients` | List clients |
| GET / DELETE | `/clients/<name>` | Read or delete a client |
| GET | `/clients/<name>/bmi` | Client BMI |
| GET | `/clients/<name>/membership` | Membership status |
| POST / GET | `/clients/<name>/progress` | Record or read weekly adherence |
| POST / GET | `/clients/<name>/workouts` | Log or read workouts |

Example request:

```bash
curl "http://localhost:5000/calories?program=Fat%20Loss%20(FL)&weight=80"
```

## 4. Running the Tests Manually

With the virtual environment active:

```bash
python -m pytest -v        # run the unit tests
python -m flake8 .         # run the linter
python -m py_compile app.py   # syntax check
```

The suite contains 24 tests covering calorie and BMI logic, membership rules, input
validation, every API endpoint, error responses (400, 404, 409) and the UI page. Each test
uses an isolated temporary SQLite database, so tests never affect each other or real data.

## 5. Docker

The `Dockerfile` has two stages:

- `base` - the runtime image (python:3.12-slim, Gunicorn, non-root user, health check)
- `test` - extends `base` with pytest and the test files

```bash
# Build and run the application
docker build --target base -t aceest-fitness .
docker run -p 5000:5000 aceest-fitness

# Run the Pytest suite inside a container
docker build --target test -t aceest-fitness:test .
docker run --rm aceest-fitness:test
```

**Optimization and security choices**

- Slim base image and `--no-cache-dir` pip installs keep the image small
- Dependencies are copied and installed before the source code, so rebuilds reuse cached layers
- The application runs as a non-root user
- `.dockerignore` excludes tests, virtual environments, databases and Git data from the runtime image
- Gunicorn serves the app instead of the Flask development server

## 6. CI/CD Integration

### 6.1 GitHub Actions (`.github/workflows/main.yml`)

Triggered on **every push and every pull request**.

| Job | Steps |
|-----|-------|
| **Build & Lint** | Check out code, set up Python 3.12, install dependencies, compile check (`py_compile`), lint with flake8 |
| **Docker Build & Test** | Build the runtime image, build the test image, run Pytest inside the container |

The Docker job runs only if Build & Lint succeeds (`needs: build-lint`). A failure in any
step marks the commit as failed, so broken code is visible immediately on the pull request.

### 6.2 Jenkins (`Jenkinsfile`)

Jenkins acts as a second, independent build and quality gate. The job is a Pipeline project
configured with **Pipeline script from SCM**, pointing to this repository, branch `*/main`
and script path `Jenkinsfile`.

| Stage | Action |
|-------|--------|
| Checkout | Pulls the latest code from GitHub |
| Clean Build Environment | Deletes any old virtual environment, creates a fresh one, installs `requirements-dev.txt` |
| Compile Check | `python -m py_compile app.py` |
| Lint | `python -m flake8 .` |
| Unit Tests | `python -m pytest -v` |

Any failing stage stops the build and marks it failed. Because every build starts from a
clean environment, a green build shows that the project installs and runs from scratch.
`Jenkinsfile.linux-docker` is an alternative that also builds and tests the Docker images on
a Linux agent with Docker installed.

**Setting up Jenkins locally**

1. Install Java 17 or 21 and download `jenkins.war` (Stable LTS) from https://www.jenkins.io/download/
2. Start it: `java -jar jenkins.war --httpPort=9090`
3. Open http://localhost:9090, enter the initial admin password, install the suggested plugins and create a user
4. Create a **Pipeline** job, set Definition to **Pipeline script from SCM**, SCM to **Git**, enter the repository URL, branch `*/main` and script path `Jenkinsfile`
5. Click **Build Now** and check the **Console Output**

### 6.3 How the pieces fit together

```
 Developer -> git push -> GitHub repository
                              |
             +----------------+-----------------+
             |                                  |
      GitHub Actions                        Jenkins
      (automatic, every                     (pulls latest code,
       push / pull request)                  clean build)
             |                                  |
   Build & Lint -> Docker build            Clean venv -> Compile
   -> Pytest in container                  -> Lint -> Pytest
```

GitHub Actions validates every change in the cloud, including the Docker image and the tests
inside the container. Jenkins independently rebuilds the project from a clean state on a
separate server as a secondary validation layer.

## 7. Version Control Strategy

- `main` holds stable, working code
- Work is done on short-lived branches: `feature/*`, `bugfix/*`, `infra/*`
- Changes are merged through pull requests, which trigger the GitHub Actions pipeline
- Commit messages follow a type prefix: `feat:`, `fix:`, `test:`, `docker:`, `ci:`, `docs:`, `chore:`
- Original ACEest versions are preserved in `legacy_versions/`

## 8. Technology Stack

Python 3.12, Flask 3, SQLite, Gunicorn, Pytest, Flake8, Docker, GitHub Actions, Jenkins, Git.

## 9. Troubleshooting

| Problem | Fix |
|---------|-----|
| `Address already in use` when starting the app or Jenkins | Another program uses the port. Set a different one (`set PORT=5001`, or `--httpPort=9090` for Jenkins) |
| `pip.exe` blocked by an organization policy | Run tools through Python: `python -m pip`, `python -m pytest`, `python -m flake8` |
| Jenkins error `Cannot run program "sh"` | The Jenkins agent is Windows; use the `bat`-based `Jenkinsfile` from this repository |
| Docker Desktop reports virtualization is not detected | Enable the Virtual Machine Platform and WSL Windows features, or rely on GitHub Actions for the Docker build and test |
| GitHub Actions tab shows "Get started" and no runs | The workflow must be at `.github/workflows/main.yml` in the repository root |

## 10. Author

**Manasa Bhat** - Introduction to DevOps (CSIZG514 / SEZG514), Assignment 1.