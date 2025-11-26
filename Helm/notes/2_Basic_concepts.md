Great! Before we build charts and write templates, you must understand the **core foundational concepts** of Helm. These are the building blocks you will see everywhere — charts, templates, releases, values, repositories, hooks, and more.

Let’s break them down in a clean, beginner-friendly but deep way.

---

# 🟦 **📌 1. Helm Chart**

A **Helm Chart** is the fundamental unit of packaging in Helm.

Think of it like a:

* **Docker image** → container package
* **npm package** → JS package
* **maven artifact** → Java package

A Chart contains **everything needed to deploy an application** on Kubernetes.

A typical chart looks like:

```
myapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
```

➡️ It packages reusable Kubernetes configs.

---

# 🟧 **📌 2. Chart.yaml**

This file **describes the chart** itself.

Includes:

* name
* version
* appVersion
* dependencies
* description

Example:

```yaml
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "v3.2.1"
description: A sample Helm chart
```

➡️ This is like **package.json** or **pom.xml** for Kubernetes applications.

---

# 🟩 **📌 3. values.yaml**

This is the **default configuration** for the chart.

Example:

```yaml
replicaCount: 3

image:
  repository: myapp
  tag: "v1.0.0"

service:
  type: ClusterIP
  port: 80
```

Values from here are injected into templates using:

```
{{ .Values.replicaCount }}
{{ .Values.image.tag }}
```

➡️ Central place for app configuration.

---

# 🟪 **📌 4. templates/ directory**

This folder contains Kubernetes manifests written using **Go Templates**.

For example:

deployment.yaml:

```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Helm **renders these templates** using values during installation.

➡️ This is what makes Helm dynamic.

---

# 🟥 **📌 5. Release**

A **release** = an installed instance of a Helm chart.

Example:

```
helm install backend-chart .
```

Here:

* `backend-chart` → release name
* `.` → chart path

If you install the same chart twice with different names, you get two different releases.

➡️ A chart is a blueprint; a **release** is the deployed, live version.

---

# 🟦 **📌 6. Helm Repository**

A repository is where charts are stored and shared.

Examples:

Bitnami repo:

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-redis bitnami/redis
```

➡️ Like Docker Hub for images
➡️ Like Maven Central for Java libraries

---

# 🟧 **📌 7. Helm Template Rendering**

At install or upgrade, Helm:

1️⃣ Reads templates
2️⃣ Applies `.Values`
3️⃣ Generates final YAML
4️⃣ Sends it to Kubernetes API

You can see the rendered YAML using:

```
helm template .
```

This does NOT install it — just renders.

---

# 🟩 **📌 8. values override (hierarchy)**

Helm allows overriding values in multiple ways.

**Priority (highest → lowest):**

1. `--set key=value`
2. `-f custom.yaml`
3. `values.yaml` (default)

Example:

```
helm install myapp . -f values-prod.yaml --set replicaCount=10
```

➡️ Very useful for dev/stage/prod separation.

---

# 🟪 **📌 9. Subcharts (Dependencies)**

Charts can depend on other charts.

Example in Chart.yaml:

```yaml
dependencies:
  - name: redis
    version: 14.8.8
    repository: https://charts.bitnami.com/bitnami
```

Then install:

```
helm dependency update
```

➡️ Helm automatically pulls and installs redis along with your chart.

---

# 🟥 **📌 10. Release Management Commands**

### Install

```
helm install myapp .
```

### Upgrade

```
helm upgrade myapp .
```

### Rollback

```
helm rollback myapp 2
```

### Uninstall

```
helm uninstall myapp
```

➡️ Helm keeps a release history for rollback.

---

# 🟦 **📌 11. Helm Hooks**

Hooks allow tasks to run:

* before installation
* after installation
* before deletion
* before upgrade
* after upgrade

Example use cases:

* Create DB schema before app starts
* Run a migration job during upgrade
* Cleanup resources before deletion

In annotation inside YAML:

```yaml
annotations:
  "helm.sh/hook": pre-install
```

➡️ Hooks make deployments more lifecycle-aware.

---

# 🟧 **📌 12. Helm Charts vs Application Configuration**

Helm chart config is NOT runtime config.

Example:

ReplicaCount, image tag → Helm
DB password, secrets → Kubernetes Secret

➡️ Clear separation between **deployment config** and **application config**.

---

# 🟥 **📌 13. Notes.txt**

A file inside templates/ called:

```
templates/NOTES.txt
```

This prints helpful info after install:

* how to access service
* which URLs to use

Example:

```
Your application is now deployed!
URL: {{ .Values.ingress.host }}
```

➡️ Pure usability feature.

---

# 🟦 **📌 14. Helm Chart Version vs appVersion**

| Field          | Meaning                                          |
| -------------- | ------------------------------------------------ |
| **version**    | Version of the Helm chart (like package version) |
| **appVersion** | Version of your actual application               |

Example:

```
version: 1.2.0
appVersion: "v3.4.1"
```

➡️ Helm chart may evolve independent of app version.

---

# 🟩 **📌 15. Chart Linting**

To validate chart structure:

```
helm lint .
```

Ensures templates, values, and chart structure follow best practices.

---

# 🟨 **📌 16. Helm Template Functions**

Go template functions let you format data:

Examples:

```
{{ toYaml .Values }}
{{ quote .Values.env }}
{{ default 80 .Values.service.port }}
{{ include "mychart.fullname" . }}
```

➡️ Makes templates powerful and clean.

---

# 🟦 **📌 17. Named Templates / Helpers**

A file named `_helpers.tpl` lets you define reusable template snippets:

Example:

```
{{- define "app.fullname" -}}
{{ .Chart.Name }}-{{ .Release.Name }}
{{- end -}}
```

Use it like:

```
{{ include "app.fullname" . }}
```

➡️ Encourages DRY (Don’t Repeat Yourself).

---

# 🟩 **📌 18. The Dot (.) Object**

The `.` in Helm templates represents the **current context**.

Examples:

* `.Values` → values from values.yaml
* `.Chart` → chart metadata
* `.Release` → release name, namespace
* `.Files` → access non-template files

➡️ Understanding `.` is key to mastering Helm.

---

# 🟫 **📌 19. Files Access**

You can include files inside charts:

```
{{ .Files.Get "config/app-config.json" }}
```

➡ Important for:

* Config files
* SQL scripts
* Embedded templates

---

# 🟪 **📌 20. Helm Lifecycle**

The order in which events occur during deployment:

1. chart loaded
2. values applied
3. templates rendered
4. manifest sent to API server
5. hooks executed
6. release tracked

➡️ Understanding lifecycle helps in debugging issues.

---

