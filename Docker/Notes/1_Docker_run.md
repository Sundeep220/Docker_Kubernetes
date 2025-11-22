Absolutely! Let's dive **deep** into `docker run` — one of the most important commands in Docker. 🚀

---

## 🔹 What `docker run` *actually* does

When you execute:

```sh
docker run IMAGE
```

Docker performs **four** actions internally:

| Step | Action                   | Description                                     |
| ---- | ------------------------ | ----------------------------------------------- |
| 1️⃣  | **Pull image**           | If the image is not already downloaded locally  |
| 2️⃣  | **Create container**     | Prepares a writable layer on top of the image   |
| 3️⃣  | **Configure networking** | Assign IP, virtual network, ports (if mapped)   |
| 4️⃣  | **Execute command**      | Starts the container’s process defined in `CMD` |

Equivalent to:

```sh
docker pull IMAGE
docker create IMAGE
docker start CONTAINER
```

---

## 🔸 General syntax of `docker run`

```sh
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

Where:

* **IMAGE** → e.g., `nginx`, `redis:7-alpine`
* **COMMAND** → overrides default CMD
* **OPTIONS** → control networking, volumes, environment, etc.

---

## 🔥 Most important `docker run` options

### ➤ 1️⃣ Run in background (daemon mode)

```sh
docker run -d nginx
```

### ➤ 2️⃣ Name the container

```sh
docker run --name my-nginx -d nginx
```

### ➤ 3️⃣ Map ports (host:container)

```sh
docker run -d -p 8080:80 nginx
```

### ➤ 4️⃣ Mount volumes (persistent data)

```sh
docker run -d -v /my/data:/var/lib/mysql mysql
```

### ➤ 5️⃣ Set environment variables

```sh
docker run -d -e MYSQL_ROOT_PASSWORD=pass mysql
```

### ➤ 6️⃣ Attach interactive terminal

```sh
docker run -it ubuntu bash
```

### ➤ 7️⃣ Auto-restart policy

```sh
docker run -d --restart unless-stopped nginx
```

Restart options:

* `no` (default)
* `always`
* `unless-stopped`
* `on-failure[:max-retries]`

---

## 🧠 Overriding ENTRYPOINT and CMD

### Override CMD:

```sh
docker run ubuntu echo "Hello"
```

### Override ENTRYPOINT:

```sh
docker run --entrypoint ls ubuntu -l
```

---

## 📌 Example: Putting it all together

```sh
docker run -d \
  --name my-app \
  -p 5000:80 \
  -e APP_ENV=prod \
  -v appdata:/app/data \
  --restart always \
  nginx:1.27
```

---

## 🛑 What happens if the main process exits?

The container **stops** — Docker containers are meant to run **one main process**.

Log:

```sh
docker logs my-app
```

Restart:

```sh
docker restart my-app
```

---

## 🎯 Summary Table

| Use Case                   | Option      |
| -------------------------- | ----------- |
| Background run             | `-d`        |
| Custom container name      | `--name`    |
| Expose ports               | `-p`        |
| Persist data               | `-v`        |
| Pass environment variables | `-e`        |
| Keep alive                 | `--restart` |
| Run commands interactively | `-it`       |

