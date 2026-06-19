# Complete Step-by-Step Setup Guide
### Local AI Job-Search Automation — n8n + Ollama on Windows 11

This guide assumes you are a **beginner**. Follow it top to bottom. Don't skip Part 0.

> **About Claude Code / Claude Desktop:** You do **not** run this automation inside Claude. Claude can *write and fix* these files and *run commands* for you, but the automation itself runs in **Docker Desktop → n8n → Ollama**. Once set up, it runs on its own (even with Claude closed). The only real prerequisite is **Docker Desktop with WSL2**.

---

## PART 0 — Prerequisites (do this once)

### 0.1 Check your Windows version
1. Press `Windows key`, type **winver**, press Enter.
2. You need **Windows 11** (any edition). Close the box.

### 0.2 Turn on virtualization features (WSL2)
WSL2 is the Linux engine Docker uses. Easiest way to install it:

1. Press `Windows key`, type **PowerShell**.
2. Right-click **Windows PowerShell → Run as administrator**.
3. Paste this and press Enter:
   ```powershell
   wsl --install
   ```
4. If it says WSL is already installed, run:
   ```powershell
   wsl --update
   ```
5. **Restart your laptop** when it asks (or restart anyway to be safe).

> If `wsl --install` errors about virtualization, reboot into BIOS and enable **Virtualization / SVM / VT-x**. Most laptops have it on by default — only do this if you hit the error.

### 0.3 Install Docker Desktop
1. Go to https://www.docker.com/products/docker-desktop/ and download **Docker Desktop for Windows**.
2. Run the installer. On the options screen, keep **"Use WSL 2 instead of Hyper-V"** checked.
3. Finish, then **launch Docker Desktop**. Accept the service agreement.
4. Wait until the whale icon in your system tray (bottom-right) is steady and the dashboard says **"Engine running"**.

### 0.4 Confirm Docker works
1. Open a **normal** PowerShell window (not admin needed now).
2. Run:
   ```powershell
   docker --version
   docker compose version
   ```
   You should see version numbers for both. If you get "command not found," make sure Docker Desktop is actually open and running, then reopen PowerShell.

✅ **Prerequisites done.** You only do Part 0 once, ever.

---

## PART 1 — Put the files in one folder
Your four files are already here:
```
C:\Users\Owner\Job Automation\AI Automation for Job Hubt\
   ├─ docker-compose.yml
   ├─ job-search-workflow.json
   ├─ ollama-system-prompt.txt
   ├─ README.md
   └─ SETUP-GUIDE.md   (this file)
```
Keep them together. You'll run commands from this folder.

### 1.1 Set your timezone (optional but recommended)
1. Open `docker-compose.yml` in Notepad.
2. Find the two lines with `Asia/Kolkata` and change them to your timezone if different (e.g. `America/New_York`, `Europe/London`).
3. Save and close.

---

## PART 2 — Start the stack

### 2.1 Open PowerShell *in this folder*
1. Open File Explorer and go to `C:\Users\Owner\Job Automation\AI Automation for Job Hubt`.
2. Click the address bar, type **powershell**, press Enter. A PowerShell window opens already pointed at this folder.

   *(Alternative: open PowerShell anywhere and run `cd "C:\Users\Owner\Job Automation\AI Automation for Job Hubt"`)*

### 2.2 Launch the containers
Run:
```powershell
docker compose up -d
```
**What this does:** downloads the n8n and Ollama images (first time only — a few minutes) and starts both containers in the background (`-d` = detached).

When it finishes you'll see `Container ollama Started` and `Container n8n Started`.

### 2.3 Confirm both are running
```powershell
docker compose ps
```
Both `n8n` and `ollama` should show **State: running** (or "Up"). You can also see them in the Docker Desktop dashboard under **Containers**.

---

## PART 3 — Download the AI model into Ollama
The Ollama container starts **empty** — it has no model yet. Pull one:

```powershell
docker exec -it ollama ollama pull llama3.1:8b
```
**What this does:** downloads the Llama 3.1 8B model (~4.7 GB) into Ollama. This is a one-time download; it's saved in a Docker volume and survives restarts.

> **On a slower / CPU-only laptop**, use a smaller, faster model instead:
> ```powershell
> docker exec -it ollama ollama pull llama3.2:3b
> ```
> If you do this, remember to set the model name to `llama3.2:3b` in the n8n "Ollama Chat Model" node later (Part 7).

Verify the model is there:
```powershell
docker exec -it ollama ollama list
```

---

## PART 4 — Open n8n and create your account
1. Open your browser and go to **http://localhost:5678**
2. First time only: n8n asks you to **set up an owner account** — enter an email + password (this is local, only for logging into your own n8n). Save these somewhere.
3. You're now in the n8n dashboard.

---

## PART 5 — Create your credentials in n8n
n8n needs permission to talk to each service. Do these four. (Left sidebar → **Credentials** → **Add credential**.)

### 5.1 Ollama (the local AI) — easiest
1. Add credential → search **Ollama** → select **Ollama API**.
2. **Base URL:** `http://ollama:11434`  ← must be exactly this, NOT `localhost`.
3. Save. (It should connect instantly since both containers share the network.)

### 5.2 Google Sheets
1. Add credential → search **Google Sheets** → select **Google Sheets OAuth2 API**.
2. n8n shows a **Sign in with Google** button. Click it, choose your Google account, allow access.
   - *If it doesn't work right away:* Google OAuth from a local n8n sometimes needs a Google Cloud project with the Sheets API enabled and an OAuth client. If you hit that wall, tell me and I'll walk you through creating the Google Cloud credentials — it's a 10-minute one-time setup.
3. Save.

### 5.3 Gmail
1. Add credential → search **Gmail** → select **Gmail OAuth2**.
2. Sign in with the same (or another) Google account, allow access. Save.

### 5.4 Decodo Scraper API
1. Log into your Decodo dashboard and copy your **API key / token**.
2. In n8n: Add credential → search **Header Auth** → select **Header Auth**.
3. **Name:** `Authorization` — **Value:** `Basic YOUR_DECODO_TOKEN` (use whatever scheme Decodo's docs specify; many use Basic auth).
4. Save.

> Tip: keep the Decodo docs open — the exact header name and the request body differ by which scraping target you use.

---

## PART 6 — Set up your Google Sheet
1. Create a new Google Sheet (sheets.google.com → blank).
2. Make **two tabs** (rename the tabs at the bottom):
   - Tab named **`Candidate Profile`** — in row 1 put these headers: `skills` | `salary_expectation` | `preferred_industries`. In row 2, fill in YOUR details, e.g.
     `Python, n8n, SQL, automation` | `120000` | `SaaS, AI, fintech`
   - Tab named **`Job Output`** — leave it empty. The workflow fills it automatically.
3. Copy the **Sheet ID** from the URL. In
   `https://docs.google.com/spreadsheets/d/`**`1AbC...XyZ`**`/edit`
   the bold part is your Sheet ID. Keep it handy.

---

## PART 7 — Import the workflow and wire it up
### 7.1 Import
1. n8n → top-left menu → **Workflows** → **Import from File**.
2. Choose `job-search-workflow.json` from this folder. The workflow appears on the canvas.

### 7.2 Fix the placeholders (click each node, set the value, then "Save")
- **Load Candidate Profile** → set **Document/Sheet ID** to your Sheet ID; **Sheet name** = `Candidate Profile`; pick your Google Sheets credential.
- **Fetch Jobs (Decodo)** → select your **Header Auth** credential. Adjust the **URL** and **JSON body** to your actual Decodo target if needed.
- **Ollama Chat Model** → select your Ollama credential; confirm **Model** = `llama3.1:8b` (or `llama3.2:3b` if you pulled the small one).
- **Save to Google Sheets** → same Sheet ID; **Sheet name** = `Job Output`; pick your Google Sheets credential.
- **Gmail Notify** → set **To** to your email; pick your Gmail credential.
- **Filter: fit_score > 50** → already set; change `50` if you want stricter/looser matching.
3. Click **Save** (top-right) to save the whole workflow.

---

## PART 8 — Test run
1. Click **Execute Workflow** (bottom-center button).
2. Watch the nodes light up green one by one. Expected: jobs fetched → each scored by Ollama → filtered → written to your **Job Output** tab → email(s) sent.
3. **If a node turns red**, click it to read the error, then see Troubleshooting below (or send me the error text).

> First scoring run can be slow on CPU — the model "warms up." Later runs are faster.

---

## PART 9 — Turn on the daily schedule
1. With the workflow open, flip the **Active** toggle (top-right) to **ON**.
2. That's it — the **Daily Trigger 8AM** node now runs the whole thing automatically every day at 8 AM (in the timezone you set in Part 0/Part 1).
3. The containers must be running for the schedule to fire. Since `docker-compose.yml` uses `restart: unless-stopped`, they auto-start when Docker Desktop starts. To have it run unattended, set Docker Desktop to **launch on login** (Docker Desktop → Settings → General → "Start Docker Desktop when you sign in").

---

## PART 10 — Everyday commands (cheat sheet)
Run these from the project folder in PowerShell:

```powershell
docker compose ps           # see what's running
docker compose logs -f n8n  # watch n8n logs live (Ctrl+C to stop watching)
docker compose stop         # stop containers (keeps data)
docker compose start        # start them again
docker compose down         # stop AND remove containers (data in volumes is kept)
docker compose pull         # download newer n8n/ollama images
docker compose up -d         # apply updates / restart after editing docker-compose.yml
```
Your workflows, credentials, and downloaded models live in Docker **volumes**, so `stop`/`down` won't lose them. (Only `docker compose down -v` deletes the volumes — don't run that unless you want a clean wipe.)

---

## Troubleshooting (top 3)

**1. n8n can't reach Ollama ("ECONNREFUSED" / "fetch failed").**
The Ollama credential's Base URL must be `http://ollama:11434`, not `localhost`. Inside Docker, `localhost` points at the n8n container itself. Test the link:
```powershell
docker exec -it n8n wget -qO- http://ollama:11434/api/tags
```
It should return JSON listing your model.

**2. "model not found" in the LLM node.**
You forgot Part 3, or the node's model name doesn't match. Re-pull and check the exact tag:
```powershell
docker exec -it ollama ollama pull llama3.1:8b
docker exec -it ollama ollama list
```
The name in the **Ollama Chat Model** node must match a name from `ollama list` exactly.

**3. Slow scoring, timeouts, or Docker crashing (out of memory).**
An 8B model is heavy on a CPU-only laptop. Fixes: (a) switch to `llama3.2:3b` or `phi3:mini`; (b) keep the 25-job cap in the "Prepare Jobs" node; (c) give WSL2 more RAM — create the file `C:\Users\Owner\.wslconfig` with:
```ini
[wsl2]
memory=8GB
processors=4
```
then run `wsl --shutdown` in PowerShell and restart Docker Desktop. If you have an NVIDIA GPU, uncomment the GPU block in `docker-compose.yml`, install the NVIDIA Container Toolkit in WSL2, and `docker compose up -d` again.

---

## Quick recap of the flow
`Daily 8AM` → `Load profile (Sheets)` → `Fetch jobs (Decodo)` → `Prepare jobs (Code)` → `Score with Ollama` → `Parse JSON` → `Filter fit_score > 50` → `Save to Sheets` → `Email me`.

Everything is local. No OpenAI key, no data leaving your laptop except the Decodo job fetch and the Google/Gmail sync you authorize.
