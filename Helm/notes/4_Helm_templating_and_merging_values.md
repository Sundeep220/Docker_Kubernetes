Great — let’s go **deep into Helm templating** and then explain **how Helm merges multiple values files**.
These two topics are the real heart of Helm — once you master them, writing charts becomes very easy.

---

# 🟦 **PART 1: Deep Dive into Helm Templating**

( `{{ }}` syntax, functions, pipelines, `.Values`, `.Chart`, `.Release`, context (`.`), helpers, etc. )

Helm uses **Go Templates** + **Helm-specific functions** to generate YAML.

Let’s break it down step by step.

---

# 🟥 **1. Templating Syntax: {{ }}**

Anything inside:

```
{{ ... }}
```

is evaluated by Helm.

Examples:

```
replicas: {{ .Values.replicaCount }}
```

```
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

```
name: {{ include "myapp.fullname" . }}
```

---

# 🟧 **2. The Dot (.) — The Context Object**

The single dot (`.`) is the **current context**.

From this dot, you can access:

| Accessor        | Meaning                                   |
| --------------- | ----------------------------------------- |
| `.Values`       | All values from values.yaml + overrides   |
| `.Chart`        | Metadata from Chart.yaml                  |
| `.Release`      | Info about the release                    |
| `.Files`        | Access to non-template files              |
| `.Capabilities` | Cluster capabilities (API versions, etc.) |

### Example:

```
{{ .Values.service.type }}
{{ .Chart.Name }}
{{ .Release.Namespace }}
```

---

# 🟩 **3. .Values — The Most Important Object**

`.Values` contains everything from `values.yaml` and any override files.

Example values.yaml:

```yaml
replicaCount: 3

image:
  repository: myapp
  tag: "1.0.0"
```

Template:

```
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

---

# 🟦 **4. .Chart — Information about the chart**

Contains metadata defined in Chart.yaml.

Example:

```
{{ .Chart.Name }}        # chart name
{{ .Chart.Version }}     # chart version
{{ .Chart.AppVersion }}  # app version
```

---

# 🟪 **5. .Release — Information about this deployment**

Contains:

```
{{ .Release.Name }}       # release name
{{ .Release.Namespace }}  # namespace where chart is installed
{{ .Release.Revision }}   # revision number (for rollback)
{{ .Release.Service }}    # "Helm"
```

Useful for naming:

```
name: {{ .Release.Name }}-service
```

---

# 🟫 **6. .Capabilities — Cluster Information**

Tells what APIs cluster supports.

Example:

```
{{ .Capabilities.KubeVersion }}
{{ .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
```

Useful for multi-version compatibility.

---

# 🟧 **7. Functions in Templates**

Helm provides many helper functions.

Examples:

### `default`

```
{{ default "ClusterIP" .Values.service.type }}
```

### `quote`

```
{{ quote .Values.env }}
```

### `toYaml`

Used for indentation:

```
{{ toYaml .Values.env | indent 8 }}
```

### `include`

Used to call helpers:

```
{{ include "myapp.fullname" . }}
```

### `required`

For mandatory values:

```
{{ required "Image tag is required!" .Values.image.tag }}
```

---

# 🟦 **8. Pipelines ( | )**

Like Linux pipes — output of one call becomes input to the next.

```
{{ .Values.env | toYaml | indent 6 }}
```

Meaning:

1. Take `.Values.env`
2. Convert to YAML
3. Indent by 6 spaces

---

# 🟥 **9. Conditionals**

```
{{- if .Values.ingress.enabled }}
# create ingress
{{- end }}
```

---

# 🟩 **10. Loops**

```
{{- range .Values.ports }}
- containerPort: {{ . }}
{{- end }}
```

---

# 🟨 **11. Helper Templates (_helpers.tpl)**

Define reusable code:

```
{{- define "myapp.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}
```

Use it with:

```
name: {{ include "myapp.fullname" . }}
```

---

# 🟫 **12. Accessing Files**

Inside chart:

```
{{ .Files.Get "config/app.json" }}
```

Or load as YAML:

```
{{ .Files.Get "config/settings.yaml" | fromYaml }}
```

---

# 🟪 **13. NOTES.txt Templating**

Whatever you write here prints after installation:

```
Your app is deployed!
kubectl get pods -n {{ .Release.Namespace }}
```

---

# 🟦 **PART 2 — How Helm Merges Multiple Values Files**

This is CRITICAL for dev/stage/prod environments.

### Helm allows multiple values files:

```
helm install myapp . -f values.yaml -f values-prod.yaml
```

Or:

```
helm upgrade myapp . -f base.yaml -f qa.yaml -f prod.yaml
```

---

# 🟥 **1. Merge Order (IMPORTANT)**

**Last file wins.**

Given:

```
-f values.yaml -f stage.yaml -f prod.yaml
```

Helm merges like:

1. Load `values.yaml`
2. Override with `stage.yaml`
3. Override with `prod.yaml` (highest priority)

---

# 🟧 **2. Example of Merge Behavior**

### base values.yaml

```yaml
replicaCount: 2
image:
  repository: myapp
  tag: "1.0"
service:
  type: ClusterIP
```

### stage.yaml

```yaml
replicaCount: 4
```

### prod.yaml

```yaml
image:
  tag: "2.0"
service:
  type: LoadBalancer
```

### Final Merged Output:

```yaml
replicaCount: 4
image:
  repository: myapp
  tag: "2.0"
service:
  type: LoadBalancer
```

Why?

* `stage.yaml` changed replicas
* `prod.yaml` changed image tag & service type

---

# 🟩 **3. Dictionaries merge — not replace**

Helm **merges nested keys**:

Given:

base:

```yaml
service:
  type: ClusterIP
  port: 80
```

override:

```yaml
service:
  port: 8080
```

Final:

```yaml
service:
  type: ClusterIP  # kept
  port: 8080       # overridden
```

---

# 🟧 **4. Lists DO NOT merge — they replace**

Example:

base:

```yaml
ports:
  - 80
  - 443
```

override:

```yaml
ports:
  - 8080
```

Final:

```yaml
ports:
  - 8080
```

Lists = replaced
Maps = merged

---

# 🟥 **5. Inline Overrides (--set) Have Highest Priority**

```
helm install myapp . \
  -f values.yaml \
  -f prod.yaml \
  --set replicaCount=10 \
  --set image.tag=3.0
```

Priority:

1. `--set`
2. `-f prod.yaml`
3. `-f values.yaml`

---

# 🟩 **6. Helm Merge Rules Summary**

| Type                 | Behavior             |
| -------------------- | -------------------- |
| **Primitive values** | replaced             |
| **Maps / objects**   | merged (deep merge)  |
| **Lists / arrays**   | replaced, NOT merged |

---

# 🎯 Summary

## 🔹 Helm Templating:

* `{{ ... }}` → evaluate template
* `.` context = `.Values`, `.Chart`, `.Release`, `.Files`
* Includes loops, conditionals, pipelines
* Helpers in `_helpers.tpl`

## 🔹 Helm Values Merge:

* Multiple values files allowed
* Last file wins
* Maps merge
* Lists replace
* `--set` has highest priority

---

