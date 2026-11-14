# docker-optimize-rollback

Docker image optimization, multi-stage build, image tagging, deployment and rollback, using a small Flask API.

**Docker Hub:** https://hub.docker.com/r/jhansi977/flask-api

---

## Summary

I took a small Python Flask API and containerized it. I made the image about **89% smaller**, tagged and pushed versions to Docker Hub, released a broken version on purpose, and **rolled back to the last working version in seconds** without rebuilding.

| What I did | Result |
|---|---|
| Image size, before vs after | 1.72 GB → 190 MB |
| Versions pushed to Docker Hub | `v1`, `v2`, `latest`, 2 Git SHA tags |
| Failure simulated | Wrong port in v2 |
| Rollback | Redeployed `v1` (no rebuild) |

**Tools:** Docker, Docker Hub, Git, GitHub, Ubuntu (WSL), VS Code, Jira.

---

## Project structure

```
docker-optimize-rollback/
├── backend/
│   ├── app.py                  # Flask API (port 5000)
│   ├── Dockerfile              # Optimized: python:3.10-slim
│   ├── Dockerfile.single       # BEFORE: full python:3.10 + extra tools
│   ├── Dockerfile.multistage   # Build stage + small runtime stage
│   └── .dockerignore           # Files kept out of the image
├── Frontend/ and other Dockerfiles   # Trainer's class practice files (not used in this project)
└── README.md
```

---

## The app

`backend/app.py` is a Flask API. It answers `Hello from Backend API` on port 5000.

---

## Part 1: Image optimization and size comparison

I built the same app with three Dockerfiles.

| File | What it does |
|---|---|
| `Dockerfile.single` | BEFORE. Full `python:3.10` image, copies everything (`COPY . .`), installs `gcc`, `vim` and `curl` that the app does not need. |
| `Dockerfile` | AFTER. `python:3.10-slim` image, copies only `app.py`. |
| `Dockerfile.multistage` | Stage 1 installs Flask with the full image. Stage 2 starts from `python:3.10-slim` and copies only the installed packages and `app.py`. |

```bash
cd backend
docker build -f Dockerfile.single     -t flask-api:single .
docker build -f Dockerfile            -t flask-api:optimized .
docker build -f Dockerfile.multistage -t flask-api:multistage .
docker images flask-api
```

### Results

| Image | Size on disk | Compressed (push/pull size) |
|---|---|---|
| `flask-api:single` | **1.72 GB** | 446 MB |
| `flask-api:optimized` | **204 MB** | 49.9 MB |
| `flask-api:multistage` | **190 MB** | 46.4 MB |

### What I learned
- **The base image is the biggest factor.** Changing from `python:3.10` to `python:3.10-slim` cut the size by about 88%.
- Removing tools the app does not need (`gcc`, `vim`, `curl`) means a smaller image and fewer security risks.
- `.dockerignore` keeps unwanted files (like `.git` and `.env`) out of the image.
- **Multi-stage gave only a 14 MB gain here** (190 MB vs 204 MB), because Flask is a tiny app. It saves much more for Java, Node or Go apps, where build tools are large.

---

## Part 2: Tagging and pushing to Docker Hub

One image can have many tags. I tagged each release in three ways.

| Tag | Meaning |
|---|---|
| `v1` | First working release |
| `v2` | Second release (contains a bug) |
| `latest` | The most recent push (moves with every push) |
| `7f38730` | Git commit ID of v1 |
| `5729d8e` | Git commit ID of v2 |

```bash
# v1
docker build -f Dockerfile.multistage -t jhansi977/flask-api:v1 .
docker tag jhansi977/flask-api:v1 jhansi977/flask-api:latest
docker tag jhansi977/flask-api:v1 jhansi977/flask-api:7f38730

docker login -u jhansi977        # used a Docker Hub access token as the password
docker push jhansi977/flask-api:v1
docker push jhansi977/flask-api:latest
docker push jhansi977/flask-api:7f38730
```

`docker tag` does not copy anything. It gives the same image another name, so all three tags point to the same image ID.

The Git commit ID comes from:
```bash
git rev-parse --short HEAD
```
This ties every image to the exact code it was built from.

---

## Part 3: Deploy v1

```bash
docker run -d --name api -p 5000:5000 jhansi977/flask-api:v1
docker ps
curl localhost:5000
```
Output:
```
Hello from Backend API
```

- `-d` runs the container in the background.
- `-p 5000:5000` maps host port 5000 to container port 5000.

---

## Part 4: Release v2 (the failure)

I changed the code: I added "v2" to the message, and I **changed the port from 5000 to 5001 by mistake**.

```bash
sed -i 's/Hello from Backend API/Hello from Backend API v2/; s/port=5000/port=5001/' app.py
git add . && git commit -m "v2: new message, port changed by mistake"
docker build -f Dockerfile.multistage -t jhansi977/flask-api:v2 .
docker tag jhansi977/flask-api:v2 jhansi977/flask-api:latest
docker tag jhansi977/flask-api:v2 jhansi977/flask-api:5729d8e
docker push jhansi977/flask-api:v2
docker push jhansi977/flask-api:latest
docker push jhansi977/flask-api:5729d8e
```

Deploy v2:
```bash
docker stop api && docker rm api
docker run -d --name api -p 5000:5000 jhansi977/flask-api:v2
docker ps
curl localhost:5000
docker logs api
```

### What happened
- `docker ps` showed the container as **Up**, so it looked healthy.
- `curl` **failed**:
  ```
  curl: (56) Recv failure: Connection reset by peer
  ```
- `docker logs api` showed the real problem:
  ```
  Running on http://172.17.0.2:5001
  ```

**Root cause:** the app listened on port **5001** inside the container, but Docker forwarded host port 5000 to container port **5000**. Nothing was listening there. A running container is not always a working application, so checking `docker ps` alone is not enough. Check the logs and test the endpoint.

---

## Part 5: Rollback to v1

Rule: **roll back first, then find the root cause.** Fixing live can take hours and cost money.

```bash
docker stop api && docker rm api
docker run -d --name api -p 5000:5000 jhansi977/flask-api:v1
docker ps
curl localhost:5000
docker logs api
```
Output:
```
Hello from Backend API
```
The logs showed `Running on ...:5000` and `"GET / HTTP/1.1" 200`.

The service was working again in seconds. I did **not** rebuild anything and did **not** edit code. I only ran the older tag.

---

## Part 6: Fix the code

After the rollback, I fixed the port in the code and committed it.

```bash
sed -i 's/port=5001/port=5000/' app.py
git add . && git commit -m "fix: restore app port to 5000 after v2 rollback"
```

### Git history

| Commit | Meaning |
|---|---|
| `7f38730` | v1: working release |
| `5729d8e` | v2: port bug |
| `064e9c6` | Fix: port back to 5000 |

I did not push a fixed image. The `v2` image on Docker Hub stays as it was, as proof of the failure. In a real team I would publish `v3` with the fix.

---

## Problems I faced and how I solved them

| Problem | Cause | Fix |
|---|---|---|
| `docker push` failed: `push access denied, repository does not exist or may require authorization` | My shell variable held the wrong Docker Hub username (`your_jhansi977` instead of `jhansi977`), so the image tags had the wrong name | Checked `docker info`, set the right username, re-tagged the image (no rebuild needed), logged in again and pushed |
| v2 container was running, but the app did not answer | Port mismatch: app on 5001, mapping to 5000 | Used `docker logs` to find the port, rolled back to v1, then fixed the code |
| `latest` points to the broken v2 on Docker Hub | `latest` moves with every push | Lesson: never deploy only `latest`. Deploy pinned tags like `v1` or a Git SHA |

---

## Skills shown
Dockerfile writing, image optimization, multi-stage builds, Docker Hub, image tagging strategy, Git commit-based tagging, container deployment, port mapping, log-based troubleshooting, rollback, Git, Jira.

---

## Next improvements (planned)
- GitHub Actions workflow to build and push images automatically, tagged with the Git commit SHA.
- Image vulnerability scan and SBOM.
- Run as a non-root user.
- Docker networks and volumes labs from the trainer's repo.

---

## Credit
Sample app and base Dockerfiles from my trainer's repo: https://github.com/ankit20000/Docker