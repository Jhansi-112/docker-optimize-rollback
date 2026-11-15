# Part A: Two containers from one image + docker exec

## Commands
```bash
docker run -d --name api  -p 5000:5000 jhansi977/flask-api:v1
docker run -d --name api2 -p 5001:5000 jhansi977/flask-api:v1
docker ps
curl localhost:5000
curl localhost:5001
docker exec -it api /bin/bash
docker exec api cat app.py
docker exec api2 ls /app
```

## What I learned
- Two containers can use the same container port, but not the same host port.
- `-p 5001:5000` means host port 5001 goes to container port 5000.
- `docker exec -it` opens a shell inside a running container.
- Containers hold only what the app needs, so `ping` is not installed.
- The image runs as root because the Dockerfile has no `USER` line.
