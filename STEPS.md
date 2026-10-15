# Q3 – Online Hospital Appointment Portal
### CI/CD automation with automated rollback

## Repository layout

```
hospital-portal/
├── index.html            Doctor profiles, specializations, timings, slots, announcements
├── styles.css
├── nginx.conf            Serves the site and exposes /healthz
├── Dockerfile
├── deployment.yaml       Deployment – 3 replicas, rolling update, readiness/liveness probes
├── service.yaml          NodePort Service on 30084
├── Jenkinsfile           Clone Code → Build Docker Image → Test Application → Push Image →
│                         Deploy to Kubernetes → Verify Deployment → Promote Stable Release
└── tests/smoke-test.sh   Post-deployment checks run inside the cluster
```

**Before you start:** replace `YOUR_DOCKERHUB_USERNAME` in `Jenkinsfile` and `deployment.yaml` with your Docker Hub username.

### Pipeline stages

| # | Stage | What it does |
|---|---|---|
| 1 | Clone Code | `checkout scm` from GitHub and prints the commit being built |
| 2 | Build Docker Image | Builds `hospital-portal:v1.0.<BUILD_NUMBER>` |
| 3 | Test Application | Confirms the image exists, validates the nginx config (`nginx -t`) and checks `index.html` is packaged |
| 4 | Push Image | Logs in with `dockerhub-creds` and pushes the versioned tag |
| 5 | Deploy to Kubernetes | Applies the NodePort Service and the Deployment (3 replicas) with the new tag |
| 6 | Verify Deployment | Waits for the rollout; runs the smoke test through the Service; **rolls back automatically on failure** |
| 7 | Promote Stable Release | Tags the verified image as `latest` and pushes it |

---

## Setup (one time)

### 1. Create the GitHub repository

```bash
cd hospital-portal
git init -b main
git add .
git commit -m "Hospital appointment portal v1.0.0"
git remote add origin https://github.com/aditya-raj-arora/hospital-portal.git
git push -u origin main
```

### 2. Build, run and push the first image

```bash
docker build --build-arg APP_VERSION=v1.0.0 -t adityarajarora123/hospital-portal:v1.0.0 -t adityarajarora123/hospital-portal:latest .
docker run -d --name hospital-local -p 8080:80 adityarajarora123/hospital-portal:v1.0.0
```

Open **http://localhost:8080**, then:

```bash
docker rm -f hospital-local
docker login
docker push adityarajarora123/hospital-portal:v1.0.0
docker push adityarajarora123/hospital-portal:latest
```

### 3. Enable Kubernetes and create the Deployment + NodePort Service

Docker Desktop → **Settings → Kubernetes → Enable Kubernetes** (kubeadm) → **Apply & Restart**.

```bash
kubectl config use-context docker-desktop
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl rollout status deployment/hospital-portal
kubectl get deployment,service hospital-portal
```

### 4. Configure Jenkins credentials

Jenkins on the same machine as Docker Desktop, with **Pipeline**, **Git** and **Credentials Binding** plugins; `docker` and `kubectl` on its PATH.

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

| Kind | ID | Value |
|---|---|---|
| Username with password | `dockerhub-creds` | Docker Hub username + access token |
| Secret file | `kubeconfig` | `C:\Users\<you>\.kube\config` or `~/.kube/config` |

### 5. Create the Jenkins pipeline

**New Item** → `hospital-portal-pipeline` → **Pipeline** → *Pipeline script from SCM* → Git → repo URL → Branch `*/main` → Script Path `Jenkinsfile` → Save → **Build Now**.

The first build registers the `pollSCM` trigger (checks GitHub every ~2 minutes). For instant triggers, expose Jenkins with ngrok and add a GitHub webhook to `https://<ngrok-url>/github-webhook/`.

Verify: **http://localhost:30084** shows the portal with `Release: v1.0.1` in the footer.

---

## Step 9 – Modify the webpage in GitHub

*(The question mentions a "placement webpage"; for this portal the equivalent change is adding a new doctor.)*

Add this card inside `<section id="doctors">`, after the last doctor card:

```html
        <article class="card doctor">
          <div class="avatar">PN</div>
          <div>
            <h3>Dr. Priya Nair</h3>
            <p class="speciality-name">Dermatology</p>
            <p>MBBS, MD (Dermatology) &middot; 7 years</p>
            <p class="lang">English, Malayalam, Tamil</p>
          </div>
        </article>
```

And this row in the consultation timings table (`<tbody>` of `#timings`):

```html
            <tr><td>Dr. Priya Nair</td><td>Dermatology</td><td>Tue &ndash; Sat</td><td>11:00 AM &ndash; 03:00 PM</td><td>&#8377;650</td></tr>
```

Commit directly on GitHub (pencil icon → *Commit changes*) or locally:

```bash
git commit -am "Add Dr. Priya Nair - Dermatology"
git push
```

## Step 10 – Trigger the pipeline

The commit is detected automatically by the SCM poll (or click **Build Now**).

## Step 11 – Verify Jenkins did everything automatically

In the build's **Console Output** confirm:

- **Retrieves the updated code** – *Clone Code* prints the new commit: `Add Dr. Priya Nair - Dermatology`
- **Builds the new image** – *Build Docker Image* creates `hospital-portal:v1.0.<N>`
- **Pushes to Docker Hub** – *Push Image* shows the pushed digest; the tag appears under *Tags* on hub.docker.com
- **Deploys to Kubernetes** – `deployment.apps/hospital-portal configured` and `successfully rolled out`

```bash
kubectl rollout history deployment/hospital-portal
```

## Step 12 – Verify through the NodePort

Open **http://localhost:30084** – Dr. Priya Nair is listed and the footer shows the new release number.

## Step 13 – Show 3 Pods running

```bash
kubectl get pods -l app=hospital-portal -o wide
kubectl get deployment hospital-portal
```

Expected: `READY 3/3`, three pods `Running`.

---

## Faulty release and automated rollback

### 1. Introduce the faulty version

In `nginx.conf`, change the listening port:

```nginx
    listen 8080;
```

```bash
git commit -am "Faulty release - nginx listens on wrong port"
git push
```

### 2. What the pipeline does

- **Build** and **Test Application** pass – the image builds and `nginx -t` reports a valid configuration.
- **Deploy to Kubernetes** applies the new tag. Kubernetes probes port 80, which is now closed, so the new pod never becomes Ready (and is restarted by the liveness probe). With `maxUnavailable: 0`, the 3 old pods continue serving.
- **Verify Deployment** sees the rollout exceed its progress deadline, prints pod status and warning events, runs `kubectl rollout undo`, waits for the previous revision and fails the build. The faulty tag is never promoted to `latest`.

### 3. Identify the failed rollout

Jenkins console:

```
error: deployment "hospital-portal" exceeded its progress deadline
deployment.apps/hospital-portal rolled back
```

Terminal:

```bash
kubectl get pods -l app=hospital-portal
kubectl describe pod <new-pod-name>        # Readiness probe failed: ... connect: connection refused
kubectl get events --field-selector type=Warning --sort-by=.lastTimestamp
kubectl rollout history deployment/hospital-portal
```

### 4. Demonstrate the restored application

```bash
kubectl rollout status deployment/hospital-portal
kubectl get pods -l app=hospital-portal                 # 3/3 Running on the previous image
kubectl get deployment hospital-portal -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Open **http://localhost:30084** – the portal (including Dr. Priya Nair) loads and the footer shows the previous stable release.

### 5. Restore the repository

```bash
git revert --no-edit HEAD
git push
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `docker` / `kubectl` not found in Jenkins | Add them to the system PATH and restart Jenkins. |
| Jenkins Windows service cannot reach Docker | `services.msc` → Jenkins → **Log On** → your Windows account → restart. |
| `docker login` credential-store error | Remove `"credsStore"` from the Jenkins user's `~/.docker/config.json`. |
| NodePort not reachable | Use the kubeadm cluster type in Docker Desktop. |
