# 🚀 **Docker Command Cheat Sheet (2025 Edition)**

---

## 🔹 **1. Docker System & Info**

| Purpose                  | Command                  |
| ------------------------ | ------------------------ |
| Show Docker version      | `docker version`         |
| Show system-wide info    | `docker info`            |
| Check Docker disk usage  | `docker system df`       |
| Remove unused data       | `docker system prune`    |
| Remove everything unused | `docker system prune -a` |
| View Docker events       | `docker events`          |

---

## 🔹 **2. Images Commands**

| Purpose                      | Command                                      |
| ---------------------------- | -------------------------------------------- |
| List images                  | `docker images`                              |
| Pull an image                | `docker pull <image>`                        |
| Build image                  | `docker build -t <name> .`                   |
| Build with custom Dockerfile | `docker build -f Dockerfile.dev -t <name> .` |
| Build without cache          | `docker build --no-cache -t <name> .`        |
| Push image to registry       | `docker push <image>`                        |
| Remove image                 | `docker rmi <image>`                         |
| Save image to tar            | `docker save -o image.tar <image>`           |
| Load image from tar          | `docker load -i image.tar`                   |
| Tag an image                 | `docker tag <source> <target>`               |
| Inspect image                | `docker image inspect <image>`               |

---

## 🔹 **3. Container Lifecycle Commands**

| Purpose                  | Command                              |
| ------------------------ | ------------------------------------ |
| Create container         | `docker create <image>`              |
| Run container            | `docker run <image>`                 |
| Run in background        | `docker run -d <image>`              |
| Run with name            | `docker run --name myapp <image>`    |
| Run with ports           | `docker run -p 8080:80 <image>`      |
| Attach to container      | `docker attach <container>`          |
| Start a container        | `docker start <container>`           |
| Stop a container         | `docker stop <container>`            |
| Restart                  | `docker restart <container>`         |
| Pause                    | `docker pause <container>`           |
| Unpause                  | `docker unpause <container>`         |
| Remove container         | `docker rm <container>`              |
| Force remove             | `docker rm -f <container>`           |
| Inspect container        | `docker inspect <container>`         |
| View logs                | `docker logs <container>`            |
| Live logs                | `docker logs -f <container>`         |
| Copy files to/from       | `docker cp <src> <container>:<dest>` |
| Execute inside container | `docker exec -it <container> sh`     |
| Exec bash                | `docker exec -it <container> bash`   |

---

## 🔹 **4. Container Listing & Stats**

| Purpose                    | Command                   |
| -------------------------- | ------------------------- |
| List running containers    | `docker ps`               |
| List all containers        | `docker ps -a`            |
| Container stats            | `docker stats`            |
| Top processes in container | `docker top <container>`  |
| Check port mappings        | `docker port <container>` |

---

## 🔹 **5. Networks Commands**

| Purpose                      | Command                                           |
| ---------------------------- | ------------------------------------------------- |
| List networks                | `docker network ls`                               |
| Create network               | `docker network create <name>`                    |
| Remove network               | `docker network rm <name>`                        |
| Connect container to network | `docker network connect <network> <container>`    |
| Disconnect from network      | `docker network disconnect <network> <container>` |
| Inspect network              | `docker network inspect <network>`                |

---

## 🔹 **6. Volumes Commands**

| Purpose               | Command                        |
| --------------------- | ------------------------------ |
| List volumes          | `docker volume ls`             |
| Create volume         | `docker volume create <name>`  |
| Remove volume         | `docker volume rm <name>`      |
| Inspect volume        | `docker volume inspect <name>` |
| Remove unused volumes | `docker volume prune`          |

---

## 🔹 **7. Docker Compose Commands**

| Purpose              | Command                            |
| -------------------- | ---------------------------------- |
| Start services       | `docker compose up`                |
| Start in background  | `docker compose up -d`             |
| Stop services        | `docker compose down`              |
| Rebuild              | `docker compose up --build`        |
| View logs            | `docker compose logs`              |
| Live logs            | `docker compose logs -f`           |
| List services        | `docker compose ps`                |
| Execute in container | `docker compose exec <service> sh` |

---

## 🔹 **8. Docker Registry Commands**

| Purpose           | Command                           |
| ----------------- | --------------------------------- |
| Login to registry | `docker login`                    |
| Logout            | `docker logout`                   |
| Tag image         | `docker tag app:v1 myrepo/app:v1` |
| Push image        | `docker push myrepo/app:v1`       |

---

## 🔹 **9. Debugging Commands**

| Purpose                | Command                                                     |
| ---------------------- | ----------------------------------------------------------- |
| Check container logs   | `docker logs <container>`                                   |
| Inspect container      | `docker inspect <container>`                                |
| Check low-level events | `docker events`                                             |
| Container filesystem   | `docker exec -it <container> sh`                            |
| Check exit code        | `docker inspect <container> --format='{{.State.ExitCode}}'` |

---

## 🔹 **10. Cleanup Commands**

| Purpose                   | Command                                     |
| ------------------------- | ------------------------------------------- |
| Remove stopped containers | `docker rm $(docker ps -aq)`                |
| Remove all images         | `docker rmi $(docker images -q)`            |
| Remove all volumes        | `docker volume rm $(docker volume ls -q)`   |
| Remove networks           | `docker network rm $(docker network ls -q)` |
| Clean everything unused   | `docker system prune -a --volumes`          |

---

## 🔹 **11. Handy One-Liners**

| Purpose                 | Command                               |
| ----------------------- | ------------------------------------- |
| Run temporary container | `docker run --rm -it <image> sh`      |
| Run with env vars       | `docker run -e KEY=value <image>`     |
| Map env file            | `docker run --env-file=.env <image>`  |
| Bind mount directory    | `docker run -v $(pwd):/app <image>`   |
| Assign network          | `docker run --network=my-net <image>` |


## 🔹 **12. Shutdown Commands**
| Command                  | Signal            | Behavior                                     |
| ------------------------ | ----------------- | -------------------------------------------- |
| `docker stop`            | SIGTERM → SIGKILL | Graceful shutdown, then forced after timeout |
| `docker kill`            | SIGKILL           | Immediate kill, no cleanup                   |
| `docker kill -s SIGTERM` | SIGTERM           | Graceful kill without stop timeout           |


---

