# Student Registration System

A simple **Flask** web application to manage student records with **MongoDB** as the backend database. Users can **add, view, update, and delete** student details.

---

## Features

* List all students on the home page
* Add a new student
* Update existing student details
* Delete a student with confirmation
* Simple and responsive UI using Bootstrap

---

## Tech Stack

* **Backend:** Python, Flask
* **Database:** MongoDB (via Flask-PyMongo)
* **Frontend:** HTML, Jinja2 templates, Bootstrap 5
* **Environment Variables:** Managed via `.env` file

---

## Setup Instructions

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd <repo-folder>
```

### 2. Create and activate a virtual environment

```bash
python -m venv venv
# Activate venv
# Windows:
venv\Scripts\activate
# Linux / Mac:
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

**`requirements.txt` example:**

```
Flask
Flask-PyMongo
python-dotenv
bson
```

### 4. Configure environment variables

Create a `.env` file in the project root:

```
MONGO_URI=<your-mongodb-connection-string>
SECRET_KEY=<your-secret-key>
```

### 5. Run the application

```bash
python app.py
```

Open your browser at: [http://localhost:8000](http://localhost:8000)

---

## Project Structure

```
project/
│
├── templates/
│   ├── base.html
│   ├── index.html
│   ├── add_student.html
│   ├── update_student.html
│
├── app.py
├── requirements.txt
└── .env
```

---
## CI/CD Pipelines

This project includes two parallel CI/CD implementations for automated testing and deployment:

1. **Jenkins** — [`Jenkinsfile`](./Jenkinsfile)
2. **GitHub Actions** — [`.github/workflows/ci-cd.yml`](./.github/workflows/ci-cd.yml)

### Prerequisites

| Requirement | Notes |
|---|---|
| Jenkins (LTS) | Installed via `apt`, Docker, or a cloud Jenkins service |
| Java 11/17 | Required by Jenkins itself |
| Python 3.8+ | Used to build/test the app |
| `python3-venv`, `pip`, `rsync`, `git` | Build/deploy tooling |
| Jenkins plugins | **Git**, **Pipeline**, **GitHub Integration**, **Email Extension**, **JUnit** |
| MongoDB URI + Secret Key | The app needs `MONGO_URI` and `SECRET_KEY` — store these as Jenkins credentials / GitHub secrets, never in code |

### Part 1 — Jenkins Pipeline

1. **Install Jenkins**
   ```bash
   sudo apt update
   sudo apt install openjdk-17-jre python3 python3-venv python3-pip git rsync -y

   curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
     /usr/share/keyrings/jenkins-keyring.asc > /dev/null
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
     https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
   sudo apt update
   sudo apt install jenkins -y
   sudo systemctl enable --now jenkins
   ```
   Open `http://<server-ip>:8080`, unlock with `/var/lib/jenkins/secrets/initialAdminPassword`,
   install the suggested plugins plus the ones listed above.

2. **Add credentials**
   *Manage Jenkins → Credentials → System → Global credentials* → add two **Secret text** credentials with IDs
   `MONGO_URI` and `SECRET_KEY`. The `Jenkinsfile` references these automatically.

3. **Create the Pipeline job**
   - Jenkins Dashboard → **New Item** → *Pipeline* → name it `flask-cicd-pipeline`.
   - Under **Pipeline** → *Pipeline script from SCM*:
     - SCM: `Git`
     - Repository URL: `https://github.com/Chinnu5119/flask_Practice.git`
     - Branch: `*/main`
     - Script Path: `Jenkinsfile`

4. **Configure the push trigger (GitHub webhook)**
   - Job config → **Build Triggers** → enable *GitHub hook trigger for GITScm polling*.
   - GitHub repo → **Settings → Webhooks → Add webhook**:
     - Payload URL: `http://<jenkins-server>:8080/github-webhook/`
     - Content type: `application/json`
     - Event: *Just the push event*
   - The `Jenkinsfile` also polls SCM every 5 minutes as a fallback.

5. **Configure email notifications**
   - *Manage Jenkins → System* → set SMTP server, port, and credentials (use an app password, not your real one).
   - Set the `NOTIFY_EMAIL` environment variable on the job, or edit the `mail to:` line in the `Jenkinsfile`.
   - The pipeline emails automatically on both success and failure.

#### Pipeline Stages

| Stage | What it does |
|---|---|
| **Checkout** | Pulls the latest code from `main` |
| **Build** | Creates a virtualenv, installs `requirements.txt` |
| **Test** | Runs `pytest test_app.py` with coverage, publishes JUnit results |
| **Deploy** | If tests pass and branch is `main`, syncs the app to `staging_deploy/`, writes a `.env` from the Jenkins credentials, and starts it on port `8000` |

Screenshots of the Build/Test/Deploy stages succeeding are in [`/screenshots/jenkins`](./screenshots/jenkins).

### Part 2 — GitHub Actions Workflow

#### Branch Strategy

- `main` — pushes/PRs trigger install + test.
- `staging` — pushes trigger install → test → build → **deploy to staging**.
- **Releases** (a tag published via GitHub Releases) trigger install → test → build → **deploy to production**.

#### Required Repository Secrets

Set under **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Used by | Description |
|---|---|---|
| `MONGO_URI` | test, deploy jobs | MongoDB connection string |
| `SECRET_KEY` | test, deploy jobs | Flask secret key |
| `STAGING_HOST` | deploy-staging | Staging server hostname/IP |
| `STAGING_USER` | deploy-staging | SSH username for staging |
| `STAGING_SSH_KEY` | deploy-staging | Private SSH key for staging |
| `PROD_HOST` | deploy-production | Production server hostname/IP |
| `PROD_USER` | deploy-production | SSH username for production |
| `PROD_SSH_KEY` | deploy-production | Private SSH key for production |

Secrets are encrypted at rest, only exposed as environment variables during a run, and masked in logs — never commit real values to the repo.

#### Jobs

| Job | Trigger | What it does |
|---|---|---|
| `install-and-test` | push/PR to `main`/`staging` | Installs `requirements.txt`, runs `pytest test_app.py` with coverage |
| `build` | after tests pass | Packages the app into a `.tar.gz` artifact |
| `deploy-staging` | push to `staging` | Downloads the artifact, deploys to the staging server over SSH |
| `deploy-production` | a GitHub Release is published | Downloads the artifact, deploys to production over SSH |

#### How to Trigger Each Job

1. **Install/Test** — open a PR against `main`, or push to `main`/`staging`.
2. **Build** — runs automatically once tests pass.
3. **Deploy to Staging** — push/merge to the `staging` branch.
4. **Deploy to Production** — tag a commit (e.g. `v1.0.0`) and publish it as a GitHub Release.

Screenshots of successful workflow runs are in [`/screenshots/github-actions`](./screenshots/github-actions).

## Screenshots

**Home Page**
Lists all students with Edit/Delete buttons.
- <img width="1902" height="607" alt="image" src="https://github.com/user-attachments/assets/a58a6a6d-4978-4769-8074-232e4d31e69d" />


**Add Student**
Form to add a new student.
- <img width="1897" height="801" alt="image" src="https://github.com/user-attachments/assets/d65d25c3-ebb5-410a-adb1-e130ad7c5878" />


**Update Student**
Form pre-filled with student details.
- <img width="1905" height="897" alt="image" src="https://github.com/user-attachments/assets/04febf01-879f-431f-ab07-abcfb993acf1" />



---

## Notes

* Make sure MongoDB is running and accessible via the URI in `.env`
* Delete action includes a confirmation page to prevent accidental deletion
* Uses `ObjectId` from `bson` to work with MongoDB document IDs
* If you use MongoDB Atlas on macOS, install dependencies again (`pip install -r requirements.txt`). This project now uses `certifi` CA bundle explicitly to avoid common TLS certificate verification failures with `pymongo`.

---

## License

MIT License

---



