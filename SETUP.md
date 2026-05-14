# DeployBoard — Complete Setup Guide
## Google Cloud Shell + Docker + Cloudflare Wildcard Subdomains

--------------------

##  CURRENT STATUS (from screenshots)

✅ Build pipeline works (clone → install → build → cleanup all succeed)  
✅ Wildcard A record `*` → `35.205.83.73` already in Cloudflare  
✅ Google Cloud Shell running at `35.205.83.73`  
❌ Auth error fixed — was trying to create DNS records unnecessarily  
❌ Wrong domain fixed — was showing `deployboard.app` instead of `joytreehostingserver.dpdns.org`  
❌ Status stuck on "building" fixed — deployId mismatch between server and frontend  
✅ Start Command field added to deployment form  

---

## PART 1 — CLOUDFLARE WILDCARD SETUP (you already did this right)

Your Cloudflare DNS already has:
```
Type: A
Name: *
Content: 35.205.83.73
Proxied: ON (orange cloud)
```

This single record means EVERY subdomain of `joytreehostingserver.dpdns.org`
automatically resolves to your GCP server. You do NOT need to create DNS records
per deployment. That is why we set `CF_WILDCARD_MODE=true`.

### What "wildcard" means in practice:
- You deploy `myapp` → DeployBoard saves files to `/var/www/user-sites/myapp/dist`
- Someone visits `myapp.joytreehostingserver.dpdns.org`
- Cloudflare sees the `*` record → routes to `35.205.83.73`
- Nginx on your server sees the subdomain in the Host header → serves from `/var/www/user-sites/myapp/dist`
- Done — no API calls needed

### One thing to verify in Cloudflare:
Make sure your ROOT domain also has an A record:
```
Type: A
Name: @   (or joytreehostingserver.dpdns.org)
Content: 35.205.83.73
Proxied: ON
```
This makes the dashboard itself (at `joytreehostingserver.dpdns.org`) accessible.

---

## PART 2 — GOOGLE CLOUD SHELL SETUP

### Step 1 — Open Google Cloud Shell

Go to: https://shell.cloud.google.com  
Click the terminal icon in the top right, or go to console.cloud.google.com and click the `>_` icon.

You get a free VM. Your external IP is `35.205.83.73`.

> **Important:** Cloud Shell VMs are temporary. For permanent hosting, you need a
> GCP Compute Engine VM (e2-micro is free tier). See Part 5 for that.
> For testing, Cloud Shell is fine.

### Step 2 — Install Docker (if not already installed)

```bash
# Check if Docker is installed
docker --version

# If not installed, run:
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker run hello-world
```

### Step 3 — Install Docker Compose

```bash
# Check version
docker compose version

# If not available:
sudo apt-get update
sudo apt-get install -y docker-compose-plugin

# Verify
docker compose version
```

### Step 4 — Clone your DeployBoard repo

```bash
# Navigate to home directory
cd ~

# Clone your repo (replace with your actual GitHub username)
git clone https://github.com/jamesgoodbusiness123/deployboard
cd deployboard

# Verify files are there
ls -la
```

You should see: `server.js`, `buildRunner.js`, `index.html`, `Dockerfile`,
`docker-compose.yml`, `nginx/`, `package.json`, `.env.example`

### Step 5 — Create your .env file

```bash
cp .env.example .env
nano .env
```

Set these values (press Ctrl+X then Y then Enter to save in nano):

```env
PORT=3001
NODE_ENV=production
RUNNER=docker
SITES_DIR=/var/www/user-sites
TMP_DIR=/tmp/deployboard-builds
MONGODB_URI=mongodb://mongo:27017/deployboard
GITHUB_TOKEN=ghp_your_actual_token_here
BASE_DOMAIN=joytreehostingserver.dpdns.org
CF_WILDCARD_MODE=true
CF_API_TOKEN=
CF_ZONE_ID=
CF_TUNNEL_ID=
CF_ACCOUNT_ID=
VPS_IP=35.205.83.73
```

> **GitHub Token:** Go to github.com → Settings → Developer Settings →
> Personal access tokens (classic) → Generate new token.
> Select scope: `repo` (full). Copy the token (starts with `ghp_`).

### Step 6 — Create required directories

```bash
sudo mkdir -p /var/www/user-sites /tmp/deployboard-builds
sudo chmod 777 /var/www/user-sites /tmp/deployboard-builds
```

### Step 7 — Create the nginx directory and verify config

```bash
# Verify nginx config is in the right place
ls nginx/
cat nginx/deployboard.conf
```

The Nginx config uses `joytreehostingserver.dpdns.org` as the domain.

### Step 8 — Open firewall ports

In Google Cloud Console:
1. Go to **VPC Network → Firewall rules**
2. Click **Create Firewall Rule**
3. Name: `allow-web`
4. Targets: All instances
5. Source IP: `0.0.0.0/0`
6. TCP ports: `80, 443, 3001`
7. Click Create

OR run this in Cloud Shell:

```bash
gcloud compute firewall-rules create allow-web \
  --allow tcp:80,tcp:443,tcp:3001 \
  --source-ranges 0.0.0.0/0 \
  --description "Allow web traffic"
```

### Step 9 — Build and start everything with Docker Compose

```bash
# From your deployboard directory:
cd ~/deployboard

# Build the Docker image
docker compose build

# Start all services (app + mongo + nginx)
docker compose up -d

# Watch logs to confirm everything started
docker compose logs -f
```

Expected output:
```
deployboard  | [DeployBoard] Running on http://localhost:3001
deployboard  | [DeployBoard] Mode:        docker
deployboard  | [DeployBoard] Base domain: joytreehostingserver.dpdns.org
deployboard  | [DeployBoard] DNS mode:    WILDCARD (no CF API needed)
deployboard  | [DB] MongoDB connected
```

Press Ctrl+C to stop watching logs (services keep running).

### Step 10 — Verify everything works

```bash
# Check all containers are running
docker compose ps

# Test the health endpoint
curl http://localhost:3001/api/health

# Expected response:
# {"ok":true,"mode":"docker","baseDomain":"joytreehostingserver.dpdns.org",...}
```

### Step 11 — Open the dashboard in your browser

Go to: `https://joytreehostingserver.dpdns.org`

You should see the DeployBoard landing page.

> **If it shows "Connection refused":** Check the firewall rules (Step 8)
> and verify Nginx is running: `docker compose ps nginx`

---

## PART 3 — DEPLOYING YOUR FIRST SITE

### From the Dashboard:

1. Click **New Deployment** (or the `+` button in topbar)
2. Fill in:
   - **Project Name:** `myapp`
   - **Subdomain:** `myapp` → preview shows `myapp.joytreehostingserver.dpdns.org`
   - **GitHub Repo URL:** `https://github.com/username/repo`
   - **Branch:** `main`
   - **Site Type:** `Static Site` for React/Vite/HTML, `Server App` for Node.js
   - **Install Command:** `npm install` (default)
   - **Build Command:** `npm run build` (for static sites)
   - **Start Command:** `npm start` (only shown for Server Apps)
   - **Output Directory:** `dist` (or `build` for Create React App)
3. Click **Deploy Now**

You will be taken to **Live Logs** automatically where you can watch the build.

### Build pipeline steps:
```
Step 1/5 — Clone Repository     (git clone from GitHub)
Step 2/5 — Install Dependencies (npm install)
Step 3/5 — Build Project        (npm run build)
Step 4/5 — Copy to Hosting      (dist/ → /var/www/user-sites/myapp/dist)
Step 5/5 — Cleanup              (remove temp files)
```

### After success:
- Status shows **success** (green badge)
- URL `myapp.joytreehostingserver.dpdns.org` goes live immediately
- Cloudflare wildcard routes it to your server automatically
- Nginx serves the files from `/var/www/user-sites/myapp/dist`

---

## PART 4 — TROUBLESHOOTING

### "DNS not registered: Authentication error"
**Fixed in this update.** Set `CF_WILDCARD_MODE=true` in your `.env`.
You have a wildcard A record so individual DNS creation is unnecessary.

### "Status stuck on building"
**Fixed in this update.** Was a deployId mismatch between server and frontend.

### "Wrong domain showing (deployboard.app)"
**Fixed in this update.** All hardcoded `deployboard.app` references replaced.
Go to Settings → Domain and make sure it shows `joytreehostingserver.dpdns.org`.

### Build fails with "Missing script: build"
Your repo's `package.json` doesn't have a `"build"` script.
- Check your `package.json` and set Build Command to an existing script
- For plain HTML sites with no build step, set Build Command to: `echo "static"` and Output Directory to `.` (the root)

### Site shows 404 after deploy
- Check the Output Directory matches where your build puts files
  (Vite → `dist`, Create React App → `build`, Next.js export → `out`)
- Verify files exist: `ls /var/www/user-sites/myapp/dist/`

### Nginx not routing subdomains
Verify the Nginx config has `joytreehostingserver.dpdns.org` (not `deployboard.app`):
```bash
docker compose exec nginx nginx -t    # test config
docker compose exec nginx cat /etc/nginx/conf.d/deployboard.conf
```

### Docker can't clone private repos
Make sure `GITHUB_TOKEN` is set correctly in `.env`.
Test manually:
```bash
git clone https://YOUR_TOKEN@github.com/username/repo /tmp/test
```

### Containers won't start
```bash
docker compose logs app     # check app errors
docker compose logs mongo   # check MongoDB errors
docker compose logs nginx   # check Nginx errors
```

---

## PART 5 — PERMANENT GCP VM (when you're done testing)

Cloud Shell is temporary (shuts down after inactivity). For permanent hosting:

### Step 1 — Create a GCP Compute Engine VM

```bash
# In Cloud Shell:
gcloud compute instances create deployboard-vm \
  --machine-type=e2-small \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --tags=http-server,https-server \
  --zone=us-central1-a
```

> `e2-micro` is always-free tier (limited RAM). Use `e2-small` for better performance.

### Step 2 — SSH into the VM

```bash
gcloud compute ssh deployboard-vm --zone=us-central1-a
```

### Step 3 — Get the external IP

```bash
gcloud compute instances describe deployboard-vm \
  --zone=us-central1-a \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

Update your Cloudflare wildcard A record to this new IP:
- Go to Cloudflare DNS → edit the `*` record → change IP to new one
- Also edit the `@` record (root domain)

### Step 4 — Run the same setup steps (2–11) from Part 2

The VM persists permanently. Your sites and database survive restarts.

---

## PART 6 — SSL/HTTPS WITH CERTBOT (optional but recommended)

Once you have a permanent VM:

```bash
# Inside your VM (not Cloud Shell)
sudo apt install certbot python3-certbot-nginx -y

# Stop Nginx temporarily
docker compose stop nginx

# Get a wildcard certificate (requires DNS challenge)
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d joytreehostingserver.dpdns.org \
  -d "*.joytreehostingserver.dpdns.org"

# Follow the prompts — it will ask you to add a TXT record in Cloudflare
# After cert is issued, update docker-compose.yml to mount /etc/letsencrypt
# and uncomment the HTTPS server block in nginx/deployboard.conf
```

---

## PART 7 — QUICK REFERENCE COMMANDS

```bash
# Start everything
docker compose up -d

# Stop everything
docker compose down

# Restart just the app (after code changes)
docker compose up -d --build app

# View live logs
docker compose logs -f app

# View Nginx logs
docker compose logs -f nginx

# Check running containers
docker compose ps

# Update DeployBoard (pull latest code)
git pull
docker compose up -d --build app

# Remove ALL data and start fresh (WARNING: deletes all sites)
docker compose down -v
docker compose up -d

# SSH into the app container for debugging
docker compose exec app sh

# List deployed sites
ls /var/www/user-sites/

# View a specific site's files
ls /var/www/user-sites/SUBDOMAIN/dist/

# Check health
curl http://localhost:3001/api/health
curl http://localhost:3001/api/projects
```
