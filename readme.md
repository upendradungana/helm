# Kubernetes Helm v3: Official Theory & Project-Based Lab Guide

## Project Directory Structure

```text
helm-guide-labs/
├── project-1-off-the-shelf/
│   ├── dev-nginx.yaml
│   └── default-nginx-values.yaml
└── project-2-pulse-web/
    └── pulse-web/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── _helpers.tpl
            ├── configmap.yaml
            ├── deployment.yaml
            ├── service.yaml
            └── NOTES.txt
```

---

## Module 1: Core Theory & Mechanism

### 1.1 What is Helm? (The Restaurant Mental Model)

Helm is both a **Template Engine** (turning dynamic variables into valid Kubernetes YAML) and a **Package Manager** (like `apt`, `yum`, or `homebrew` for Kubernetes).

Students remember the three core building blocks best through a restaurant analogy:

* **Chart (The Recipe Book):** A bundled folder of templated Kubernetes YAML files (`deployment.yaml`, `service.yaml`, `configmap.yaml`) plus package metadata.
* **values.yaml (The Custom Order):** A plain YAML file containing configuration knobs (such as `replicaCount: 3`, `image.tag: "1.27-alpine"`, or custom HTML text).
* **Release (The Served Dish):** A specific, named running instance of a Chart inside a Kubernetes cluster. You can install the exact same `nginx` Chart three times in one cluster as three independent Releases: `frontend-web`, `docs-portal`, and `status-page`.

---

### 1.2 How Helm Works Under the Hood (Helm v3 Architecture)

A common beginner misconception is thinking Helm runs a background controller inside the cluster. Helm v3 is strictly a client-side CLI tool.

```text
+-------------------------------------------------------+       HTTPS (:6443)       +----------------------------------------------------+
|               LOCAL MACHINE (Helm v3 CLI)             |   (Uses ~/.kube/config)   |                 KUBERNETES CLUSTER                 |
|                                                       |                           |                                                    |
|  [Chart Templates] + [values.yaml] + [-f env.yaml]    |                           |  +----------------------------------------------+  |
|                         |                             |                           |  |            Kubernetes API Server             |  |
|                         v                             |                           |  +----------------------------------------------+  |
|           [Go Template Rendering Engine]              |   Pure K8s YAML Manifests |            |                         |             |
|        (Evaluates {{ .Values }} in memory)            | ========================> |            v                         v             |
|                                                       |                           |  [Live NGINX Workloads]   [Helm Release Secrets]   |
|                                                       |                           |  (Deployments, Pods,      (sh.helm.release.v1.*    |
|                                                       |                           |   Services, ConfigMaps)    tracks v1, v2, v3...)   |
+-------------------------------------------------------+                           +----------------------------------------------------+
```

When you execute `helm install` or `helm upgrade`, four steps happen in sequence:

1. **Load & Merge:** The Helm CLI reads the Chart templates from disk (or a remote repo) and merges the default `values.yaml` with your custom override files (`-f custom-values.yaml`).
2. **Client-Side Rendering:** Helm's built-in Go template engine injects those values into every `{{ .Values.xyz }}` placeholder, generating pure Kubernetes YAML manifests in memory.
3. **API Server Submission:** Helm sends the rendered YAML directly to the Kubernetes API Server using your standard `~/.kube/config` credentials and RBAC permissions.
4. **Release State Storage:** How does Helm track history and rollbacks without a server inside the cluster? It stores a compressed snapshot of each Release revision as a native Kubernetes Secret (with label `owner=helm` and type `helm.sh/release.v1`) directly inside the target namespace. Every upgrade creates a new Secret (`v1`, `v2`, `v3`), allowing instant rollbacks.

---

### 1.3 Anatomy of a Chart Directory

| File / Folder | Daily Operational Purpose |
| :--- | :--- |
| **`Chart.yaml`** | Metadata about the chart: name, version (chart package revision), `appVersion` (the underlying NGINX version), and sub-chart dependencies. |
| **`values.yaml`** | Default configuration variables (replicas, image tag, HTML content) that users override per environment. |
| **`templates/`** | Templated Kubernetes manifest skeletons (`deployment.yaml`, `service.yaml`, `configmap.yaml`). |
| **`templates/_helpers.tpl`** | Reusable named template snippets (like standard labels and resource naming macros) shared across manifests. |
| **`templates/NOTES.txt`** | Plain-text usage instructions printed to the terminal immediately after `helm install` completes. |

---

## Module 2: Project 1 — Consuming Off-the-Shelf Charts ("Custom NGINX Web Server")

### Project Goal

Learn the daily operational lifecycle of consuming a community NGINX chart and replacing its default HTML landing page using a ConfigMap:

$$\text{Search} \longrightarrow \text{Inspect} \longrightarrow \text{Override HTML via ConfigMap} \longrightarrow \text{Deploy} \longrightarrow \text{Break} \longrightarrow \text{Rollback}$$

#### Step 1: Add and Update the Bitnami Chart Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

#### Step 2: Inspect the NGINX Chart's Configurable Knobs

Never install a community chart blind. Dump its default `values.yaml` so students can discover how the chart author allows custom HTML ConfigMaps (`staticSiteConfigmap` and `extraDeploy`):

```bash
helm search repo bitnami/nginx
helm show values bitnami/nginx > default-nginx-values.yaml
```

#### Step 3: Create an Override File with a Custom HTML ConfigMap (`dev-nginx.yaml`)

Instead of copying all 1,000+ lines from `default-nginx-values.yaml`, create a clean override file containing only the parameters we want to customize.

Here, we use `extraDeploy` to create a Kubernetes ConfigMap containing our custom `index.html`, and pass its name to `staticSiteConfigmap` so the chart automatically mounts our HTML into NGINX:

```yaml
# dev-nginx.yaml
replicaCount: 2

service:
  type: ClusterIP
  ports:
    http: 80

# Tell bitnami/nginx to mount this ConfigMap as the HTML web root
staticSiteConfigmap: "custom-nginx-html"

# Bundle the ConfigMap directly inside the Helm release
extraDeploy:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: custom-nginx-html
      namespace: web-lab
    data:
      index.html: |
        <!DOCTYPE html>
        <html>
        <head>
          <title>Helm NGINX Lab</title>
          <style>
            body { font-family: sans-serif; background: #0f172a; color: #38bdf8; text-align: center; padding-top: 10%; }
            .card { display: inline-block; background: #1e293b; padding: 2rem; border-radius: 12px; border: 1px solid #334155; }
          </style>
        </head>
        <body>
          <div class="card">
            <h1>Hello from Bitnami NGINX!</h1>
            <p>This custom HTML page is mounted via a <strong>Kubernetes ConfigMap</strong> managed by Helm.</p>
          </div>
        </body>
        </html>
```

#### Step 4: Preview, Deploy, and View the Custom NGINX Page

```bash
# 1. Render locally first to inspect the generated Deployment, Service, and ConfigMap YAML
helm template my-nginx bitnami/nginx -n web-lab -f dev-nginx.yaml

# 2. Install the Release into the cluster
helm install my-nginx bitnami/nginx \
  --namespace web-lab \
  --create-namespace \
  -f dev-nginx.yaml

# 3. Verify the Pods and ConfigMap are created
kubectl get pods,svc,configmap -n web-lab

# 4. Port-forward and view your custom HTML in a browser or via curl!
kubectl port-forward svc/my-nginx 8080:80 -n web-lab
# Open http://localhost:8080 (or run: curl http://localhost:8080)
```

#### Step 5: Lab Exercise — Edit the HTML, Break the Image, & Roll Back

Now let students experience live upgrades and instant rollbacks:

```bash
# 1. Push a bad upgrade with a non-existent NGINX image tag (creates Revision 2)
helm upgrade my-nginx bitnami/nginx -n web-lab -f dev-nginx.yaml --set image.tag=broken-tag-999

# 2. Watch the new Pod fail with ImagePullBackOff / ErrImagePull
kubectl get pods -n web-lab

# 3. Inspect Helm's revision history and underlying Kubernetes Secrets
helm history my-nginx -n web-lab
kubectl get secrets -n web-lab -l owner=helm

# 4. Roll back to Revision 1 (creates Revision 3 with Revision 1's working state)
helm rollback my-nginx 1 -n web-lab

# 5. Verify recovery
helm history my-nginx -n web-lab
kubectl get pods -n web-lab
```

---

## Module 3: Project 2 — Creating NGINX Chart

### Project Goal

Build a custom, multi-environment NGINX microservice chart (`pulse-web`) from an empty folder where `templates/configmap.yaml` dynamically generates an HTML dashboard using Helm variables, `if`/`else` conditionals, `range` loops, and built-in `.Release` metadata.

### The 6 Core Templating Rules Used Daily

Before writing the templates, review the six constructs students will use in real-world charts:

1. **Built-in Objects:** `.Values` (reads `values.yaml`), `.Release` (Name, Namespace, Revision), and `.Chart` (Name, Version, AppVersion).
2. **Chomping Whitespace (`{{-` and `-}}`):** Strips extra blank lines caused by template tags.
3. **Pipelines & Functions (`|`):** Chains functions together, such as `{{ .Values.bannerText | quote }}` or `{{ .Values.environment | upper }}`.
4. **Conditionals (`{{- if ... }}`):** Shows or hides YAML/HTML blocks based on boolean flags.
5. **Loops (`{{- range ... }}`):** Iterates over lists in `values.yaml` to avoid repetitive code.
6. **Smart Indentation (`nindent X`):** Inserts a newline and indents an entire block by $X$ spaces so YAML alignment never breaks.

#### Step 1: Scaffold the Minimal Chart Folder

```bash
mkdir -p pulse-web/templates
cd pulse-web
```

#### Step 2: Create `Chart.yaml`

```yaml
# pulse-web/Chart.yaml
apiVersion: v2
name: pulse-web
description: A custom NGINX web chart with a dynamic ConfigMap HTML dashboard
type: application
version: 0.1.0        # Bump when chart templates change
appVersion: "1.27.0"  # Underlying NGINX version
```

#### Step 3: Create Default `values.yaml`

Notice how we define variables for the NGINX container alongside variables that will control our HTML ConfigMap:

```yaml
# pulse-web/values.yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

# Custom HTML ConfigMap Knobs
environment: "development"
bannerText: "Welcome to NGINX on Kubernetes!"
themeColor: "#0ea5e9"
showReleaseMetadata: true
features:
  - "Templated NGINX Deployment"
  - "Dynamic HTML ConfigMap Injection"
  - "Zero-Downtime SHA-256 ConfigMap Rollouts"
```

#### Step 4: Create Reusable Macros in `templates/_helpers.tpl`

`_helpers.tpl` prevents copy-pasting the same metadata labels and naming conventions across every YAML file:

```yaml
# pulse-web/templates/_helpers.tpl
{{- define "pulse.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}

{{- define "pulse.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

#### Step 5: Create the Dynamic NGINX HTML ConfigMap (`templates/configmap.yaml`)

This single file teaches students almost every major Helm templating feature (`include`, `upper`, `if`/`else`, `range`, `.Values`, and `.Release`) while producing a visual webpage served by NGINX:

```yaml
# pulse-web/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "pulse.fullname" . }}-html
  labels:
    {{- include "pulse.labels" . | nindent 4 }}
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <title>{{ .Chart.Name }} | {{ .Values.environment | upper }}</title>
      <style>
        body { font-family: system-ui, sans-serif; background: #0f172a; color: #f8fafc; padding: 3rem; }
        .banner { border-left: 6px solid {{ .Values.themeColor }}; background: #1e293b; padding: 1.5rem 2rem; border-radius: 8px; max-width: 650px; margin: 0 auto; }
        .badge { background: {{ .Values.themeColor }}; color: #0f172a; padding: 4px 10px; border-radius: 999px; font-weight: bold; font-size: 0.8rem; }
        .meta { margin-top: 1.5rem; padding-top: 1rem; border-top: 1px solid #334155; font-family: monospace; color: #94a3b8; font-size: 0.9rem; }
      </style>
    </head>
    <body>
      <div class="banner">
        <span class="badge">ENV: {{ .Values.environment | upper }}</span>
        <h1>{{ .Values.bannerText }}</h1>

        <h3>Active Helm Capabilities:</h3>
        <ul>
        {{- range .Values.features }}
          <li>{{ . }}</li>
        {{- end }}
        </ul>

        {{- if .Values.showReleaseMetadata }}
        <div class="meta">
          <p><strong>Release Name:</strong> {{ .Release.Name }}</p>
          <p><strong>Namespace:</strong> {{ .Release.Namespace }}</p>
          <p><strong>Revision:</strong> v{{ .Release.Revision }}</p>
          <p><strong>Chart Version:</strong> {{ .Chart.Name }}-{{ .Chart.Version }} (NGINX {{ .Chart.AppVersion }})</p>
        </div>
        {{- end }}
      </div>
    </body>
    </html>
```

#### Step 6: Create the NGINX Deployment and Service

Notice how `templates/deployment.yaml` mounts the ConfigMap directly into NGINX's default document root (`/usr/share/nginx/html`) and includes the `checksum/config` annotation:

```yaml
# pulse-web/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "pulse.fullname" . }}
  labels:
    {{- include "pulse.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      annotations:
        # Forces NGINX Pods to restart whenever configmap.yaml HTML content changes!
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "pulse.labels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
          volumeMounts:
            - name: nginx-html-storage
              mountPath: /usr/share/nginx/html
              readOnly: true
      volumes:
        - name: nginx-html-storage
          configMap:
            name: {{ include "pulse.fullname" . }}-html
```

Create `templates/service.yaml`:

```yaml
# pulse-web/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "pulse.fullname" . }}
  labels:
    {{- include "pulse.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
```

#### Step 7: Lint, Render, and Deploy Multi-Environment NGINX Releases

```bash
# 1. Validate chart syntax and best practices
helm lint .

# 2. Preview the rendered ConfigMap HTML and Deployment YAML in the terminal
helm template preview-app .

# 3. Deploy Release 1: DEV Environment (1 Replica, Green Theme, Debug Metadata ON)
helm upgrade --install web-dev . -n dev --create-namespace \
  --set environment="dev" \
  --set replicaCount=1 \
  --set themeColor="#10b981" \
  --set bannerText="NGINX Development Sandbox" \
  --set showReleaseMetadata=true

# 4. Deploy Release 2: PROD Environment (3 Replicas, Orange Theme, Debug Metadata OFF)
helm upgrade --install web-prod . -n prod --create-namespace \
  --set environment="production" \
  --set replicaCount=3 \
  --set themeColor="#f59e0b" \
  --set bannerText="NGINX Production Storefront" \
  --set showReleaseMetadata=false
```

#### Step 8: Live-Edit the NGINX HTML via `helm upgrade`

Have students port-forward to `web-dev`, view the webpage, and then update `bannerText` with `helm upgrade` to see the page update automatically:

```bash
# Update the HTML banner text in DEV
helm upgrade web-dev . -n dev \
  --reuse-values \
  --set bannerText="Updated Live via Helm Upgrade (Revision 2)!"

# Verify that the Pod rolled automatically and serves the new HTML
kubectl port-forward svc/web-dev-pulse-web 8081:80 -n dev
# Refresh http://localhost:8081 in your browser!
```

---

## Module 4: Production Deep-Dive — Safe ConfigMap Rollouts (`checksum/config`)

### 4.1 The Kubernetes ConfigMap Gotcha

When NGINX mounts an HTML file or `nginx.conf` from a ConfigMap, updating that ConfigMap during a `helm upgrade` does not automatically restart your NGINX Pods. Because the Deployment YAML itself didn't change—only the separate ConfigMap object did—Pods either keep serving cached configuration or fail to reload `nginx.conf`.

To guarantee a clean rollout every time `configmap.yaml` changes, Helm engineers place a dynamic SHA-256 checksum annotation inside the Deployment's Pod template (`spec.template.metadata.annotations`).

---

### 4.2 Where to Put It (`templates/deployment.yaml`)

Add the `checksum/config` line directly under `spec.template.metadata.annotations` (not the top-level Deployment metadata):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "pulse.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      annotations:
        # Forces a Pod rolling restart whenever configmap.yaml changes!
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "pulse.labels" . | nindent 8 }}
```

---

### 4.3 How the Helm Expression Works (Piece by Piece)

Read the expression from the inside out:

$$\text{\{\{ include (print \$.Template.BasePath "/configmap.yaml") . | sha256sum \}\}}$$

1. **`$.Template.BasePath`**: Returns the folder path of the current chart's templates directory (for example, `pulse-web/templates`).
2. **`(print $.Template.BasePath "/configmap.yaml")`**: Combines the path and filename into a single string: `"pulse-web/templates/configmap.yaml"`.
3. **`include (...) .`**: Tells Helm to render `configmap.yaml` (including your custom NGINX HTML) in memory using your current `values.yaml` context (`.`), producing the complete ConfigMap YAML text.
4. **`| sha256sum`**: Pipes that rendered ConfigMap YAML text into a cryptographic SHA-256 hash function, turning the entire file into a single 64-character fingerprint.

---

### 4.4 What Happens During `helm upgrade`

If you run `helm template` to inspect what Helm sends to Kubernetes, that template expression turns into a simple string hash:

#### Release 1 (`bannerText: "Hello Dev"`):

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: 4a8b9c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b
```

#### Release 2 (You change `bannerText: "Updated HTML!"` in `values.yaml`):

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: 9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c3b2a1f0e9d8c7b6a5f4e3d2c1b0a9f8e
```

Because the `checksum/config` string inside `spec.template.metadata.annotations` changed from `4a8b...` to `9f8e...`, Kubernetes detects a modification to the Pod template and immediately triggers a zero-downtime RollingUpdate so new NGINX Pods start up with the updated HTML ConfigMap.

---

## Module 5: Daily Production Cheat Sheet

| Daily Task | Command | Why Engineers Use It |
| :--- | :--- | :--- |
| **Idempotent Deploy (CI/CD)** | `helm upgrade --install [release] [chart] -f values.yaml` | Installs the release if it doesn't exist; upgrades it if it does. |
| **Local YAML Rendering** | `helm template [release] [chart] -f values.yaml` | Debugs template logic and HTML ConfigMap output locally without cluster access. |
| **API Server Validation** | `helm install [release] [chart] --dry-run=server` | Sends rendered YAML to the API server to catch schema/admission errors without creating resources. |
| **Inspect Live Overrides** | `helm get values [release] -n [namespace]` | Shows the exact custom overrides currently active on a running release. |
| **Inspect Live Manifests** | `helm get manifest [release] -n [namespace]` | Outputs the full rendered Kubernetes YAML (Deployment, Service, ConfigMap) deployed in the cluster. |
| **Check Release History** | `helm history [release] -n [namespace]` | Lists all revisions (v1, v2, v3), statuses, and descriptions. |
| **Instant Rollback** | `helm rollback [release] [revision] -n [namespace]` | Reverts the release to a previous revision snapshot and records a new revision number. |
| **Safe ConfigMap Rollouts** | Add `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . \| sha256sum }}` under Pod annotations | Forces NGINX Pods to automatically restart whenever ConfigMap HTML or config values change during `helm upgrade`. |