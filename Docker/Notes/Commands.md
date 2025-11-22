Perfect — let’s go over **every important Docker command**, grouped by purpose, with **short explanations + examples**.
This will give you a complete reference like a cheat sheet but with clarity.

---

# 🐳 **Docker CLI Commands — Full Guide (Most Useful to Least Used)**

---

# ⭐ 1. **Basic Commands**

### ✔️ Check Docker installation

```
docker --version
docker info
```

### ✔️ Login to a registry (Docker Hub, ACR, ECR)

```
docker login
```

---

# ⭐ 2. **Container Commands**

## 🔹 List containers

```
docker ps          # running containers
docker ps -a       # all containers (including stopped)
```

## 🔹 Run a container

```
docker run <image>
docker run -d <image>        # run in detached mode
docker run -p 8080:80 <image> # map ports
docker run -e KEY=value <image> # set environment variable
```

Examples:

```
docker run --name mydb -e POSTGRES_PASSWORD=pass -d postgres
```

## 🔹 Stop & start a container

```
docker stop <container>
docker start <container>
```

## 🔹 Restart a container

```
docker restart <container>
```

## 🔹 Remove (delete) a container

```
docker rm <container>
docker rm -f <container>    # force remove
```

## 🔹 View container logs

```
docker logs <container>
docker logs -f <container>  # follow logs like tail -f
```

## 🔹 Execute inside a container (VERY IMPORTANT)

```
docker exec -it <container> bash
docker exec -it <container> sh
```

## 🔹 Copy files between host and container

```
docker cp file.txt <container>:/app/file.txt
docker cp <container>:/app/output.txt .
```

---

# ⭐ 3. **Image Commands**

## 🔹 List images

```
docker images
```

## 🔹 Build an image from Dockerfile

```
docker build -t myapp .
docker build -f Dockerfile.dev -t myapp:dev .
```

## 🔹 Remove images

```
docker rmi <image>
docker rmi -f <image>
```

## 🔹 Pull image from registry

```
docker pull nginx
```

## 🔹 Push image to registry

```
docker push myapp:latest
```

## 🔹 Tag image

```
docker tag myapp:latest myrepo/myapp:v1
```

---

# ⭐ 4. **Docker Volumes**

## 🔹 List volumes

```
docker volume ls
```

## 🔹 Create volume

```
docker volume create myvol
```

## 🔹 Inspect volume

```
docker volume inspect myvol
```

## 🔹 Remove volume

```
docker volume rm myvol
```

## 🔹 Remove all unused volumes

```
docker volume prune
```

---

# ⭐ 5. **Docker Networks**

## 🔹 List networks

```
docker network ls
```

## 🔹 Create network

```
docker network create mynet
```

## 🔹 Inspect network

```
docker network inspect mynet
```

## 🔹 Remove network

```
docker network rm mynet
```

---

# ⭐ 6. **Docker System Commands (Clean up resources)**

## 🔹 Remove unused images, containers, networks

```
docker system prune
docker system prune -a   # very aggressive – removes all unused images
```

## 🔹 Show disk usage

```
docker system df
```

---

# ⭐ 7. **Docker Compose Commands (Important)**

> File: `docker-compose.yml`

## 🔹 Start services

```
docker-compose up
docker-compose up -d      # detached mode
```

## 🔹 Stop services

```
docker-compose down
```

## 🔹 Rebuild and start containers

```
docker-compose up --build
```

## 🔹 View logs

```
docker-compose logs
docker-compose logs -f     # live logs
```

## 🔹 Run a specific service

```
docker-compose up redis
```

## 🔹 Execute inside a service container

```
docker-compose exec backend bash
```

---

# ⭐ 8. **Docker Inspect Commands**

## 🔹 Inspect container

```
docker inspect <container>
```

## 🔹 Inspect image

```
docker inspect <image>
```

## 🔹 Get container IP

```
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>
```

---

# ⭐ 9. **Docker Stats & Monitoring**

## 🔹 Live CPU/RAM usage

```
docker stats
```

## 🔹 Container processes

```
docker top <container>
```

---

# ⭐ 10. **Docker Events & History**

## 🔹 View Docker events (real-time)

```
docker events
```

## 🔹 View image history (what layers were created)

```
docker history <image>
```

---

# ⭐ 11. **Advanced Commands**

## 🔹 Save image to tar file (export)

```
docker save -o myapp.tar myapp:latest
```

## 🔹 Load image from tar

```
docker load -i myapp.tar
```

## 🔹 Export container filesystem

```
docker export <container> > container.tar
```

## 🔹 Import container filesystem

```
cat container.tar | docker import -
```