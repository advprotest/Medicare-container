# MediCare+ Hospital Management System

MediCare+ is a hospital management app with a **FastAPI** backend, a **React (Vite)** frontend and **MongoDB** as its only database. It runs as three containers with **Podman** and **podman-compose**, and is built and started by the **Jenkins** pipeline in `Jenkinsfile`. Docker is not used anywhere.

## 1. Project layout

```
medicare-container-main/
├── Jenkinsfile               # CI/CD pipeline (Podman)
├── podman-compose.yml        # mongo + backend + frontend stack
├── .env.example              # settings read by podman-compose (copy to .env)
├── medicare-plus/            # FastAPI backend (Python 3.12)
│   ├── Containerfile
│   ├── .containerignore
│   ├── app/                  # main.py, config.py, database.py, routers/, models/, core/, utils/
│   ├── tests/test_smoke.py   # pytest smoke tests (no MongoDB needed)
│   ├── seed.py               # demo users, doctors, patient, pharmacy stock
│   ├── requirements.txt
│   └── .env.example
└── frontend/                 # React 19 + Vite 8 SPA
    ├── Containerfile         # multi-stage: node build -> nginx
    ├── nginx.conf
    ├── .containerignore
    ├── src/
    └── package.json          # scripts: dev, build, lint (oxlint), preview
```

## 2. How the pieces talk

| Service  | Image                                     | Container port | Host port (default) | Notes |
|----------|-------------------------------------------|----------------|---------------------|-------|
| mongo    | `docker.io/library/mongo:7.0`             | 27017          | not published       | data kept in the `mongo-data` volume |
| backend  | `localhost/medicare-plus-backend:latest`  | 8000           | 8000                | Swagger UI at `/docs`, health at `/health` |
| frontend | `localhost/medicare-plus-frontend:latest` | 80 (nginx)     | 8081                | static build of the React app |

- The backend reads its settings from environment variables via `app/config.py`: `MONGO_URI`, `MONGO_DB_NAME`, `JWT_SECRET_KEY`, `JWT_ALGORITHM`, `ACCESS_TOKEN_EXPIRE_MINUTES`, plus optional `RAZORPAY_*`, `SMTP_*`, `SMS_GATEWAY_API_KEY`. In the compose file, `MONGO_URI` is `mongodb://mongo:27017`.
- The frontend calls the API at `VITE_API_BASE_URL` (`frontend/src/api/client.js`). **Vite bakes this value into the JavaScript at build time**, and the call is made by the user's browser. It must be a URL the browser can reach, such as `http://<server-ip>:8000`, not `http://backend:8000`. If you change it, rebuild the frontend image.
- The frontend uses port **8081** so it does not clash with Jenkins on 8080.
- Base images use full names (`docker.io/library/...`) because Podman does not assume a default registry for short names.

## 3. Install Podman on Linux

**RHEL / Rocky / Alma 8 or 9, Fedora**

```bash
sudo dnf -y install podman python3-pip curl tar
pip3 install --user podman-compose          # or: sudo dnf -y install podman-compose (EPEL / Fedora)
```

**Ubuntu 22.04+ / Debian 12+**

```bash
sudo apt-get update
sudo apt-get install -y podman podman-compose curl
```

Check it:

```bash
podman --version
podman-compose --version
```

Open the ports if a firewall is on:

```bash
# firewalld (RHEL family)
sudo firewall-cmd --permanent --add-port=8000/tcp --add-port=8081/tcp && sudo firewall-cmd --reload
# ufw (Ubuntu)
sudo ufw allow 8000/tcp && sudo ufw allow 8081/tcp
```

On cloud VMs, also allow those ports in the security group.

## 4. Run it on Linux with podman-compose

```bash
cd medicare-container-main

# 1. Settings. The browser must be able to reach VITE_API_BASE_URL.
cp .env.example .env
#    then edit .env: JWT_SECRET_KEY, VITE_API_BASE_URL=http://<your-server-ip>:8000

# 2. Build and start
podman-compose up -d --build
podman ps

# 3. Load demo data (safe to run more than once)
podman-compose exec -T backend python seed.py

# 4. Check
curl http://localhost:8000/health        # {"status":"healthy"}
```

Open:

- Frontend: `http://<your-server-ip>:8081`
- API docs (Swagger): `http://<your-server-ip>:8000/docs`

Demo logins created by `seed.py`:

| Role    | Email                        | Password     |
|---------|------------------------------|--------------|
| Admin   | `admin@medicareplus.com`     | `Admin@123`  |
| Doctor  | `dr.smith@medicareplus.com`  | `Doctor@123` |
| Patient | `john.doe@example.com`       | `Patient@123`|

`POST /api/auth/login` takes **form data** (`username`, `password`); `POST /api/auth/login-json` takes JSON.

```bash
curl -X POST http://localhost:8000/api/auth/login \
     -d 'username=admin@medicareplus.com&password=Admin@123'
```

Everyday commands:

```bash
podman-compose logs -f backend            # follow API logs
podman-compose restart backend
podman-compose up -d --build frontend     # after changing VITE_API_BASE_URL or frontend code
podman-compose down                       # stop (keeps MongoDB data)
podman-compose down -v                    # stop and DELETE the MongoDB volume
```

Run the backend tests in a throwaway container:

```bash
podman run --rm -v "$PWD/medicare-plus":/src:Z -w /src docker.io/library/python:3.12-slim \
  sh -c "pip install -q -r requirements.txt && pytest tests/ -v"
```

**Keep containers running after you log out (rootless).** Rootless Podman containers stop when the user's session ends unless lingering is on:

```bash
sudo loginctl enable-linger $USER
```

**Startup order.** The compose file uses a plain `depends_on`. Podman runs healthchecks through systemd timers, so `condition: service_healthy` can hang where there is no systemd user session (for example some Jenkins agents). If MongoDB is still starting, the backend exits and `restart: unless-stopped` starts it again.

## 5. Jenkins

### 5.1 Install Jenkins on Linux

Jenkins needs Java 17 or 21.

**RHEL family**

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo dnf -y install fontconfig java-17-openjdk jenkins
```

**Ubuntu / Debian**

```bash
sudo apt-get install -y fontconfig openjdk-17-jre
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt-get update && sudo apt-get install -y jenkins
```

**Both**

```bash
sudo systemctl enable --now jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword   # unlock key for first login
```

Browse to `http://<server-ip>:8080`, paste the key and install the suggested plugins. The pipeline uses **Pipeline**, **Git**, **JUnit**, **Timestamper** and **Workspace Cleanup** (all in the suggested set). For the `Deploy` stage's `branch 'main'` condition, use a **Multibranch Pipeline** job.

### 5.2 Prepare the Jenkins user for rootless Podman

The `jenkins` user runs Podman itself, without sudo or root.

```bash
# 1. Sub-UID/GID ranges for rootless containers (skip if the user already has them)
grep jenkins /etc/subuid /etc/subgid || \
  sudo usermod --add-subuids 200000-265535 --add-subgids 200000-265535 jenkins

# 2. Keep jenkins' containers running after each build ends
sudo loginctl enable-linger jenkins

# 3. podman-compose for the jenkins user (skip if installed system-wide by apt/dnf)
sudo -u jenkins -H pip3 install --user podman-compose
#    and make sure ~jenkins/.local/bin is on Jenkins' PATH:
#    Manage Jenkins > System > Global properties > Environment variables
#    PATH+LOCAL = /var/lib/jenkins/.local/bin

# 4. Check
sudo -u jenkins -H podman info --format '{{.Host.Security.Rootless}}'   # true
sudo -u jenkins -H podman-compose --version
```

### 5.3 Create the job

1. **New Item → Pipeline** (or Multibranch Pipeline) → *Pipeline script from SCM* → Git → your repo URL and branch. Script Path: `Jenkinsfile`.
2. In `Jenkinsfile`, set `VITE_API_BASE_URL` to `http://<server-ip>:8000`.
3. Optional: create a *Secret text* credential with ID `medicare-jwt-secret` and swap in `JWT_SECRET_KEY = credentials('medicare-jwt-secret')` as the comment in the file shows. Without it the default key from `app/config.py` is used.
4. **Build Now**. When it is green, open `http://<server-ip>:8081` and load demo data once:

```bash
cd /var/lib/jenkins/workspace/<job-name>
sudo -u jenkins -H podman-compose -p medicare -f podman-compose.yml exec -T backend python seed.py
```

### 5.4 What the pipeline does

The stages follow the project's original Jenkinsfile. The difference is that everything runs in Podman containers, so the agent needs only `podman`, `podman-compose`, `curl` and `tar`. It does not need Python, Node.js or MongoDB.

| Stage | What it runs |
|-------|--------------|
| Checkout | `checkout scm` |
| Verify Podman Toolchain | prints `podman` / `podman-compose` versions |
| Stop Previous Run | `podman-compose -p medicare down` (keeps the MongoDB volume) |
| Setup Python Environment | creates `medicare-plus/.venv` inside a `python:3.12-slim` container |
| Backend Lint | `flake8` (same flags as before) |
| Backend Compile Check | `python -m py_compile ...` |
| Backend Unit Tests | `pytest` with a JUnit report |
| Frontend Install / Lint / Build | `npm install`, `npm run lint` (oxlint), `npm run build` in a `node:22-alpine` container |
| Frontend Unit Tests | disabled until a `test` script exists (same as before) |
| Package Artifact | `dist/medicare-plus-<build>.tar.gz` (backend source, Containerfiles, frontend `dist/`, compose file), archived |
| Build Images | `podman-compose build` |
| Run Stack | `podman-compose up -d`, then waits for `/health` |
| Diagnose Stack | `podman ps` and the last log lines of each service (saved to `run-logs/`) |
| Deploy | placeholder on `main` (same as before) |

Other differences from the original pipeline:
- MongoDB is the `mongo` container instead of a `mongod` tarball downloaded into the workspace.
- The frontend is the nginx container on **8081** instead of the Vite dev server on 5173.
- The package is a `.tar.gz` instead of a `.zip`, so the agent doesn't need `zip`.
- The workspace is cleaned after the build, because the running containers don't read from it. Only `.venv` and `node_modules` are kept as a cache.

The lint, test, build and compose steps were run outside Jenkins with Podman 4.9.3 and podman-compose 1.6.0, and all passed: flake8 and the compile check were clean, pytest passed 7 tests, oxlint and the Vite build succeeded, `/health` answered, `seed.py` ran and admin login returned a token. The pipeline was not run on a Jenkins server.

## 6. Troubleshooting

| Symptom | Fix |
|---------|-----|
| Frontend loads but every call fails / login spins | `VITE_API_BASE_URL` is wrong for the browser. Set it to `http://<server-ip>:8000` and rebuild the frontend (`podman-compose up -d --build frontend`). |
| `short-name ... did not resolve to an alias` | Use full image names (`docker.io/library/...`), as the Containerfiles and compose file already do. |
| `cannot find newuidmap` / `there might not be enough IDs available` | Install `uidmap` (Ubuntu) or `shadow-utils` (RHEL) and add sub-UIDs (section 5.2), then `podman system migrate`. |
| Containers disappear when the build finishes or you log out | `sudo loginctl enable-linger jenkins` (or your user). |
| `podman-compose: command not found` in Jenkins | Add `/var/lib/jenkins/.local/bin` to Jenkins' PATH (section 5.2). |
| `Permission denied` on mounted files (SELinux) | Keep the `:Z` suffix on volume mounts. |
| Ports below 1024 refused | Rootless Podman can't bind them. Keep 8000/8081 or put a reverse proxy in front. |
| Backend restarts a few times at first start | MongoDB was still starting; it settles by itself. Check `podman-compose logs backend` if it keeps restarting. |
| Refreshing `/patients` returns 404 | `frontend/nginx.conf` is missing the `try_files ... /index.html` line. |

## 7. Before production

- Set a strong `JWT_SECRET_KEY` (Jenkins credential or `.env`).
- Restrict CORS in `medicare-plus/app/main.py` (currently `allow_origins=["*"]`).
- Keep MongoDB's port unpublished, as the compose file does.
- Put the frontend and API behind HTTPS, for example an nginx or Traefik reverse proxy.
- To run the stack as a system service, use `podman generate systemd` or Quadlet `.container` files.
