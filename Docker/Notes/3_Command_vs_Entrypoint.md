Great — **CMD vs ENTRYPOINT** is one of the most misunderstood but *most powerful* concepts in Docker.
Let’s go beyond definitions and understand **how Docker executes processes**, what gets overridden, and how images should be designed for production.

---

# 🚀 Big Picture: CMD vs ENTRYPOINT

Every Docker container starts by running **exactly one process**, defined by:

* **ENTRYPOINT** (the executable)
* **CMD** (its default arguments)

Think of ENTRYPOINT as **the program**, and CMD as **the default options**.

```
ENTRYPOINT ["python"]
CMD ["app.py"]
```

When container runs → executes:

```
python app.py
```

---

# ⚡ Real Difference (Deep Explanation)

| Feature                      | ENTRYPOINT             | CMD                      |
| ---------------------------- | ---------------------- | ------------------------ |
| Purpose                      | Fixed executable       | Default arguments        |
| Overridable by `docker run`? | ❌ No                   | ✔ Yes                    |
| Designed for                 | Production images      | Development convenience  |
| Behaves like                 | Always executed        | Optional                 |
| Typical usage                | `ENTRYPOINT ["nginx"]` | `CMD ["-g daemon off;"]` |

---

# 🧩 How Docker merges them

Docker creates the **final command** like this:

```
ENTRYPOINT + CMD
```

Example:

```
ENTRYPOINT ["python", "-m"]
CMD ["http.server"]
```

→ Final command:

```
python -m http.server
```

---

# 🔥 What happens if CMD is overridden?

Command-line overrides CMD:

```sh
docker run myimage custom.py
```

Final command:

```
python -m custom.py
```

But ENTRYPOINT stays the same.

---

# 🔥 What happens if ENTRYPOINT is overridden?

You must explicitly override it:

```sh
docker run --entrypoint ls myimage
```

Final command:

```
ls http.server   # CMD is still passed unless replaced
```

Behavior:

* ENTRYPOINT replaced
* CMD still passed automatically (unless you add your own args)

---

# 🧪 3 Execution Modes: exec vs shell

## 1️⃣ Exec Form (Recommended)

```
ENTRYPOINT ["python"]
CMD ["app.py"]
```

Exec form:
✔ No shell involved
✔ Signals handled correctly
✔ Container behaves like the process
✔ Best for production

This is why **PID 1** signal-forwarding works (SIGTERM, SIGINT).

---

## 2️⃣ Shell Form

```
ENTRYPOINT python app.py
```

Runs as:

```
/bin/sh -c "python app.py"
```

Problems:

* Shell eats signals → graceful shutdown fails
* Harder to override
* Unexpected quoting

**Avoid for production containers.**

---

# 💡 When to use ENTRYPOINT?

### ✔ When your image is a *single-purpose executable*

Examples:

* `nginx`
* `postgres`
* `redis`
* `vault`
* `docker-entrypoint.sh`

Example:

```
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

This ensures nginx **always runs** even if user accidentally provides arguments.

---

# 💡 When to use CMD?

### ✔ When your container is configurable

Examples:

* Scripts
* Startup commands
* Local development
* Debug images

Example:

```
CMD ["npm", "start"]
```

Users can easily override:

```sh
docker run my-node-image npm test
```

---

# 💡 When to combine ENTRYPOINT + CMD?

Use this pattern **almost always** for production apps:

```
ENTRYPOINT ["java", "-jar", "/app.jar"]
CMD ["--server.port=8080"]
```

Users can override port:

```sh
docker run myapp --server.port=9090
```

---

# 🚫 Bad Patterns (Avoid)

### ❌ Putting everything in CMD

```
CMD ["python", "app.py"]
```

Why bad?

* Overriding command breaks container intention
* No fixed executable

### ❌ Using ENTRYPOINT with shell form

```
ENTRYPOINT python app.py
```

Breaks signal handling.

---

# 🚀 Advanced: ENTRYPOINT Scripts

Most real-world images wrap their binary with a script:

```
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
CMD ["redis-server"]
```

Why?

✔ Configure environment
✔ Initialize volumes
✔ Prepare config files
✔ Then exec the main app

Inside script:

```sh
#!/bin/sh
set -e
# some setup
exec "$@"
```

`exec "$@"` replaces the script with the real process → crucial for signals.

---

# 🎯 Summary Table

| Scenario                       | Use                    |
| ------------------------------ | ---------------------- |
| Always run a specific app      | ENTRYPOINT             |
| Provide default arguments      | CMD                    |
| User should override program   | CMD                    |
| User should override args only | ENTRYPOINT + CMD       |
| Production images              | ENTRYPOINT (exec form) |
| Signals needed                 | Use exec form only     |

---
