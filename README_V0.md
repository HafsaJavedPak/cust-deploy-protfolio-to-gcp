# GCP Tech Portfolio — Full Deployment Guide

> Cloud Resume Challenge · No Terraform · Beginner-friendly · $5 budget · Estimated cost: ~$0.10/week

This guide walks you through deploying a complete tech portfolio on Google Cloud Platform — with a live visitor counter, a Gemini-powered AI chatbot, and a voice agent — all for under $0.10/week using free tiers.

---

## What You're Building

| Component | Description | Cost |
|---|---|---|
| Portfolio site | HTML/CSS hosted on Firebase | Free |
| Visitor counter | JS + Cloud Functions + Firestore | Free tier |
| CI/CD pipeline | GitHub Actions auto-deploy | Free |
| AI chatbot | Vertex AI Agent Builder (Gemini) | GCP native |
| Voice agent | Dialogflow Messenger + Cloud TTS | Showstopper |

---

## Architecture

Everything is deployed via CLI commands and the GCP Console. Two GitHub repos — one for frontend and one for backend — each with their own GitHub Actions workflow.

> **Why skip the Load Balancer?**
> A GCP Load Balancer costs ~$0.60/day ($4.20/week) — nearly your entire $5 budget. Firebase Hosting is actually backed by Google Cloud Storage and gives you free HTTPS + global CDN + a clean `.web.app` URL. For a beginner demo, it's the right call.

---

## Prerequisites

- A Google account with $5 GCP credits activated
- **gcloud CLI** installed — `brew install google-cloud-sdk` on Mac, or download from cloud.google.com/sdk
- **Node.js** installed (for Firebase CLI) — `brew install node`
- A **GitHub account** with two new empty repos: `portfolio-frontend` and `portfolio-backend`
- Python 3.11 installed (for Cloud Functions)
- A text editor (VS Code recommended)

---

## Project Deployment Guide

### STEP 1 — Set Up Your GCP Project

#### 1.1 Create a New Project

1. Go to **console.cloud.google.com** and sign in with your Google account
2. Click the **project dropdown** at the top of the page (it might say "Select a project" or show an existing project name)
3. In the popup, click **"New Project"** (top right of the dialog)
4. Fill in:
   - **Project name:** `Hafsa Portfolio`
   - **Project ID:** `hafsa-portfolio-gcp` (edit it if it auto-generated something different — click the pencil icon next to the ID)
   - Leave **Location** as "No organization" unless you're in a Workspace org
5. Click **Create**
6. Wait ~30 seconds, then click the notification bell and click **"Select Project"** to switch into it

#### 1.2 Link Billing Account

1. In the left sidebar, click the **hamburger menu (☰)** → scroll down to **"Billing"**
2. If your project has no billing account, you'll see a banner — click **"Link a billing account"**
3. Select your billing account from the dropdown (the one with your $5 credits)
4. Click **"Set Account"**

#### 1.3 Enable Required APIs

1. Click **☰ → APIs & Services → Library**
2. Search for and enable each of these one by one — search the name, click the result, click **"Enable"**:
   - `Cloud Functions API`
   - `Cloud Build API`
   - `Cloud Firestore API`
   - `Cloud Run API`
   - `Dialogflow API`
   - `Cloud Text-to-Speech API`
   - `Cloud Storage API`
   - `Vertex AI API`

Each one takes 10–30 seconds to enable. You'll see a green checkmark and an "API Enabled" banner when done.

#### 1.4 Create Firestore Database

1. Click **☰ → Firestore**
2. You'll see a "Select a database mode" screen — choose **"Native mode"** and click **"Continue"**
3. Under **Location**, select **`asia-south1 (Mumbai)`** — closest to Pakistan
4. Click **"Create Database"**
5. Wait about a minute for it to provision

#### 1.5 Create the Visitor Counter Document

Once Firestore is ready:

1. Click **"+ Start collection"**
2. **Collection ID:** `visitors` → click **Next**
3. **Document ID:** `counter` (type it manually — don't use "Auto-ID")
4. Under **"Add a field"**:
   - Field name: `count`
   - Type: **Number**
   - Value: `0`
5. Click **Save**

---

### STEP 2 — Build the Portfolio Site (Files on Your Local Computer)

You do this on your own machine. Open VS Code (or any text editor) and create a folder called `portfolio-frontend`. Inside it, create these files exactly:

#### `index.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Hafsa | Cloud & Solutions Architect</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <header>
    <h1>Hafsa</h1>
    <p class="tagline">Cloud & Solutions Architect · AWS Certified · GCP</p>
    <p class="visitor-count">
      Visitors: <span id="count">...</span>
    </p>
  </header>

  <section id="about">
    <h2>About</h2>
    <p>Computer Systems Engineering graduate, UET Peshawar.
    AWS Cloud Club Captain · Microsoft Learn Student Ambassador Lead.
    Specializing in cloud architecture and data engineering.</p>
  </section>

  <section id="certifications">
    <h2>Certifications</h2>
    <ul>
      <li>AWS Cloud Practitioner</li>
      <li>GCP Cloud Digital Leader (in progress)</li>
    </ul>
  </section>

  <section id="projects">
    <h2>Featured Projects</h2>
    <div class="project-card">
      <h3>Quantum-Classical Portfolio Optimization</h3>
      <p>FYP: Hybrid quantum + classical optimizer for stock portfolios
      using ARIMAX/GARCH forecasting, deployed on Azure Functions.</p>
      <span class="tag">Azure</span>
      <span class="tag">Python</span>
      <span class="tag">XGBoost</span>
    </div>
    <div class="project-card">
      <h3>This Portfolio</h3>
      <p>Deployed on GCP with CI/CD, AI chatbot, and voice agent.</p>
      <span class="tag">GCP</span>
      <span class="tag">Firebase</span>
      <span class="tag">Vertex AI</span>
    </div>
  </section>

  <script src="counter.js"></script>
</body>
</html>
```

#### `style.css`
```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: 'Segoe UI', system-ui, sans-serif;
  max-width: 780px;
  margin: 0 auto;
  padding: 40px 24px;
  color: #1a1a1a;
  line-height: 1.7;
}
header {
  margin-bottom: 48px;
  border-bottom: 1px solid #eee;
  padding-bottom: 24px;
}
h1 { font-size: 36px; font-weight: 700; }
h2 { font-size: 20px; margin: 32px 0 12px; }
h3 { font-size: 16px; margin-bottom: 8px; }
.tagline { color: #666; font-size: 16px; margin-top: 6px; }
.visitor-count {
  margin-top: 12px;
  font-size: 13px;
  color: #999;
  background: #f5f5f5;
  display: inline-block;
  padding: 4px 10px;
  border-radius: 20px;
}
section { margin-bottom: 40px; }
.project-card {
  border: 1px solid #eee;
  border-radius: 10px;
  padding: 20px;
  margin-bottom: 16px;
}
.tag {
  display: inline-block;
  background: #e8f5ee;
  color: #0f6e56;
  padding: 3px 8px;
  border-radius: 4px;
  font-size: 12px;
  margin: 8px 4px 0 0;
}
```

#### `counter.js`
Leave the URL blank for now — you'll fill it in after Step 5:
```javascript
const FUNCTION_URL =
  'https://asia-south1-hafsa-portfolio-gcp.cloudfunctions.net/visitor-counter';

async function updateVisitorCount() {
  try {
    const response = await fetch(FUNCTION_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' }
    });
    if (!response.ok) throw new Error('API call failed');
    const data = await response.json();
    const countEl = document.getElementById('count');
    if (countEl) countEl.textContent = data.count.toLocaleString();
  } catch (error) {
    console.error('Could not fetch visitor count:', error);
    const countEl = document.getElementById('count');
    if (countEl) countEl.textContent = '—';
  }
}

document.addEventListener('DOMContentLoaded', updateVisitorCount);
```

---

### STEP 3 — Deploy to Firebase Hosting

Firebase Hosting is configured through the Firebase Console, which is separate from GCP Console but uses the same Google account.

#### 3.1 Open Firebase Console

1. Go to **console.firebase.google.com**
2. Click **"Add project"**
3. Select your existing GCP project: **`hafsa-portfolio-gcp`** from the dropdown
4. Click **Continue** through the steps (you can turn off Google Analytics or leave it on)
5. Click **"Create project"** / **"Continue"**

#### 3.2 Enable Hosting

1. In the Firebase Console left sidebar, click **"Build" → "Hosting"**
2. Click **"Get started"**
3. The wizard will show you CLI commands — **ignore them** (we're using the console approach via GitHub Actions in Step 6). Just click **Next → Next → Continue to console**

#### 3.3 Create `firebase.json` on your local machine

In your `portfolio-frontend` folder, create `firebase.json`:

```json
{
  "hosting": {
    "public": ".",
    "ignore": [
      "firebase.json",
      ".firebaserc",
      ".github/**",
      "node_modules/**"
    ],
    "headers": [
      {
        "source": "**",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "max-age=3600, s-maxage=86400"
          }
        ]
      }
    ]
  }
}
```

Also create `.firebaserc` in the same folder:

```json
{
  "projects": {
    "default": "hafsa-portfolio-gcp"
  }
}
```

#### 3.4 Initial Manual Deploy via Firebase Console (Upload Method)

Since we're avoiding CLI, we'll do the first deploy manually through the console:

1. In Firebase Console → **Hosting** → click **"Add custom domain"** is optional; your default domain is already `hafsa-portfolio-gcp.web.app`
2. In Firebase Console → **Hosting** → click your site name → **"Release history"** tab
3. For the very first deploy, you'll need the Firebase CLI just once (on your local machine). Install it: open a terminal and run:
   ```
   npm install -g firebase-tools
   firebase login
   firebase deploy --only hosting
   ```
   from inside your `portfolio-frontend` folder. After this one-time deploy, all future deploys happen via GitHub Actions (Step 6) and you won't need the CLI again.

Your site will be live at `https://hafsa-portfolio-gcp.web.app`

---

### STEP 4 — Deploy the Cloud Function (Visitor Counter Backend)

This is done entirely in the GCP Console using the inline code editor.

#### 4.1 Create a Cloud Storage Bucket for Function Source

1. Go to **console.cloud.google.com** → **☰ → Cloud Storage → Buckets**
2. Click **"Create"**
3. **Name:** `hafsa-portfolio-functions-source` (must be globally unique — add your initials if needed)
4. **Region:** `asia-south1`
5. Leave all other settings as default
6. Click **Create**

#### 4.2 Upload Function Source Files

You need to upload your backend files. First create them locally:

Create a folder called `portfolio-backend` on your computer with these three files:

**`main.py`**
```python
import functions_framework
from google.cloud import firestore
from flask import jsonify

db = firestore.Client()

@functions_framework.http
def visitor_counter(request):
    cors_headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type',
    }

    if request.method == 'OPTIONS':
        return ('', 204, cors_headers)

    doc_ref = db.collection('visitors').document('counter')
    doc = doc_ref.get()

    if doc.exists:
        current_count = doc.to_dict().get('count', 0)
        new_count = current_count + 1
    else:
        new_count = 1

    doc_ref.set({'count': new_count})

    return jsonify({'count': new_count}), 200, cors_headers
```

**`requirements.txt`**
```
functions-framework==3.*
google-cloud-firestore==2.*
```

Now zip these two files together. On your computer:
- **Windows:** select both files → right-click → "Send to" → "Compressed (zipped) folder" → name it `function-source.zip`
- **Mac:** select both files → right-click → "Compress 2 items" → rename to `function-source.zip`

**Important:** The zip must contain the files directly (not inside a subfolder). Open it and confirm you see `main.py` and `requirements.txt` at the root level of the zip.

Now upload to GCS:
1. Go back to **Cloud Storage → Buckets → `hafsa-portfolio-functions-source`**
2. Click **"Upload files"**
3. Select your `function-source.zip`
4. Wait for upload to complete

#### 4.3 Create the Cloud Function

1. Go to **☰ → Cloud Functions**
2. Click **"Create Function"**
3. Fill in **Configuration** tab:
   - **Environment:** `2nd gen`
   - **Function name:** `visitor-counter`
   - **Region:** `asia-south1`
   - **Trigger type:** `HTTPS`
   - **Authentication:** `Allow unauthenticated invocations` ← important, tick this
   - **Require HTTPS:** leave checked
4. Expand **"Runtime, build, connections and security settings"**:
   - **Memory:** `256 MiB`
   - **CPU:** `1`
   - **Maximum number of instances:** `5`
5. Click **"Next"**
6. On the **Code** tab:
   - **Runtime:** `Python 3.11`
   - **Entry point:** `visitor_counter`
   - **Source code:** Change dropdown to `Cloud Storage`
   - Paste the GCS path or click Browse and select your `function-source.zip` from `hafsa-portfolio-functions-source`
7. Click **"Deploy"**

Wait 2–3 minutes. You'll see a green checkmark when it's done.

#### 4.4 Get the Function URL

1. Click on `visitor-counter` in the functions list
2. Go to the **"Trigger"** tab
3. Copy the **Trigger URL** — it looks like:
   `https://asia-south1-hafsa-portfolio-gcp.cloudfunctions.net/visitor-counter`

#### 4.5 Update `counter.js`

Go back to your `portfolio-frontend/counter.js` and confirm/update the `FUNCTION_URL` at the top with the URL you just copied. The URL in the template already matches the expected format, so if your project ID is `hafsa-portfolio-gcp` it should be correct.

#### 4.6 Fix Permissions if You Get a 403 Error

If the function returns a 403 when called from the browser:

1. Go to **Cloud Functions → visitor-counter → Permissions tab**
2. Click **"Add principal"**
3. **New principal:** `allUsers`
4. **Role:** `Cloud Functions Invoker`
5. Click **Save** (you may see a warning about public access — click "Allow public access")

---

### STEP 5 — Set Up GitHub Repos and CI/CD

#### 5.1 Create Both GitHub Repos

1. Go to **github.com** → click the **+** icon → **"New repository"**
2. Create `portfolio-frontend` — Public, no README
3. Repeat for `portfolio-backend` — Public, no README

#### 5.2 Push Your Code to GitHub

On your local machine, open a terminal in each folder:

```bash
## Frontend
cd portfolio-frontend
git init
git add .
git commit -m "Initial portfolio site"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/portfolio-frontend.git
git push -u origin main

## Backend
cd ../portfolio-backend
git init
git add .
git commit -m "Add visitor counter Cloud Function"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/portfolio-backend.git
git push -u origin main
```

#### 5.3 Create a Service Account in GCP Console

1. Go to **GCP Console → ☰ → IAM & Admin → Service Accounts**
2. Click **"+ Create Service Account"**
3. **Name:** `github-deployer`
4. **ID:** will auto-fill as `github-deployer`
5. Click **"Create and Continue"**
6. **Grant roles** — click "Add another role" for each:
   - `Cloud Functions Developer`
   - `Cloud Run Developer`
   - `Storage Object Admin`
   - `Service Account User`
7. Click **"Continue"** → **"Done"**

#### 5.4 Create and Download Service Account Key

1. Click on `github-deployer` in the service accounts list
2. Go to the **"Keys"** tab
3. Click **"Add Key" → "Create new key"**
4. Select **JSON** → click **"Create"**
5. A file `hafsa-portfolio-gcp-xxxxxxxx.json` downloads automatically — this is your `GCP_SA_KEY`

**Keep this file secure — never commit it to Git.**

#### 5.5 Get Firebase Token

In your terminal:
```bash
firebase login:ci
```
A browser window opens. Authenticate with your Google account. Copy the token printed in the terminal — this is your `FIREBASE_TOKEN`.

#### 5.6 Add Secrets to GitHub Repos

**For `portfolio-backend`:**

1. Go to **github.com/YOUR_USERNAME/portfolio-backend**
2. Click **Settings → Secrets and variables → Actions**
3. Click **"New repository secret"**
4. **Name:** `GCP_SA_KEY`
5. **Value:** open the JSON key file you downloaded, select all the text, paste it here
6. Click **"Add secret"**

**For `portfolio-frontend`:**

1. Go to **github.com/YOUR_USERNAME/portfolio-frontend → Settings → Secrets → Actions**
2. Add `GCP_SA_KEY` (same JSON content)
3. Add another secret:
   - **Name:** `FIREBASE_TOKEN`
   - **Value:** the token from `firebase login:ci`

#### 5.7 Create CI/CD Workflow Files

**In `portfolio-backend`**, create `.github/workflows/deploy.yml`:

```yaml
name: Deploy backend
on:
  push:
    branches: [ main ]
jobs:
  test-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install functions-framework google-cloud-firestore pytest

      - name: Run tests
        run: python -m pytest test_main.py -v

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Deploy Cloud Function
        uses: google-github-actions/deploy-cloud-functions@v2
        with:
          name: visitor-counter
          runtime: python311
          region: asia-south1
          entry_point: visitor_counter
```

Also add `test_main.py` to your `portfolio-backend` folder:

```python
import unittest
from unittest.mock import patch, MagicMock
import main

class TestVisitorCounter(unittest.TestCase):

    @patch('main.db')
    def test_increments_existing_count(self, mock_db):
        mock_doc = MagicMock()
        mock_doc.exists = True
        mock_doc.to_dict.return_value = {'count': 42}
        mock_db.collection.return_value.document.return_value.get.return_value = mock_doc
        request = MagicMock()
        request.method = 'POST'
        response = main.visitor_counter(request)
        response_data = response[0].get_json()
        self.assertEqual(response_data['count'], 43)

    @patch('main.db')
    def test_starts_at_one_when_no_document(self, mock_db):
        mock_doc = MagicMock()
        mock_doc.exists = False
        mock_db.collection.return_value.document.return_value.get.return_value = mock_doc
        request = MagicMock()
        request.method = 'POST'
        response = main.visitor_counter(request)
        response_data = response[0].get_json()
        self.assertEqual(response_data['count'], 1)

    @patch('main.db')
    def test_handles_preflight_cors(self, mock_db):
        request = MagicMock()
        request.method = 'OPTIONS'
        response = main.visitor_counter(request)
        self.assertEqual(response[1], 204)

if __name__ == '__main__':
    unittest.main()
```

**In `portfolio-frontend`**, create `.github/workflows/deploy.yml`:

```yaml
name: Deploy frontend
on:
  push:
    branches: [ main ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Firebase CLI
        run: npm install -g firebase-tools

      - name: Deploy to Firebase Hosting
        run: firebase deploy --only hosting --token ${{ secrets.FIREBASE_TOKEN }}
        env:
          FIREBASE_CLI_EXPERIMENTS: webframeworks
```

Commit and push both workflow files. Go to **github.com/YOUR_USERNAME/portfolio-backend/actions** — you'll see the workflow run automatically. Green checkmark = success.

---

### STEP 6 — AI Chatbot with Vertex AI Agent Builder

#### 6.1 Create a Knowledge Base Document

On your local machine, create a plain text file called `portfolio-info.txt`:

```
Name: Hafsa
Role: Cloud and Solutions Architecture Engineer
Education: Computer Systems Engineering, UET Peshawar (final year)

Certifications:
- AWS Cloud Practitioner
- GCP Cloud Digital Leader (in progress)

Leadership Roles:
- AWS Cloud Club Captain
- Microsoft Learn Student Ambassador Lead
- Community Manager at AtomCamp
- Data Engineering Instructor (weekend course)

Featured Projects:
1. Quantum-Classical Portfolio Optimization (FYP)
   - Built a hybrid quantum + classical stock portfolio optimizer
   - Uses ARIMAX/GARCH forecasting + XGBoost pre-selection
   - Deployed on Azure Functions with Blob Storage
   - Stack: Python, Azure ML, FinBERT, SLSQP, simulated annealing

2. GCP Tech Portfolio (this website)
   - Deployed using Firebase Hosting, Cloud Functions, Firestore
   - CI/CD with GitHub Actions
   - AI chatbot powered by Vertex AI Agent Builder
   - Voice agent using Dialogflow Messenger + Cloud TTS

Skills: AWS, Azure, GCP, Python, Cloud Functions, Firestore,
Docker, SQL, data engineering, machine learning, portfolio optimization

Contact: [your LinkedIn URL]
```

#### 6.2 Upload the Document to Cloud Storage

1. Go to **GCP Console → Cloud Storage → Buckets**
2. Click **"Create"**
3. **Name:** `hafsa-portfolio-knowledge`
4. **Region:** `asia-south1`
5. Click **Create**
6. Open the bucket → click **"Upload files"**
7. Upload `portfolio-info.txt`

#### 6.3 Create Vertex AI Agent Builder Data Store

1. Go to **☰ → Vertex AI → Agent Builder**
   - If you don't see it, search "Agent Builder" in the top search bar
2. If prompted, click **"Continue and activate the API"**
3. Click **"Create data store"**
4. Select **"Cloud Storage"** as the source type
5. Click **"Browse"** and navigate to your `hafsa-portfolio-knowledge` bucket
6. Select `portfolio-info.txt`
7. Click **"Continue"**
8. **Data store name:** `portfolio-knowledge`
9. Keep the ID as auto-generated
10. Click **"Create"**
11. Wait 5–10 minutes for indexing. The status will show "Importing" then change to show document count.

#### 6.4 Create the AI Agent (Chat App)

1. Still in **Agent Builder**, click **"Create app"**
2. Select **"Chat"**
3. **App name:** `Portfolio Assistant`
4. **Company name:** `Hafsa Portfolio`
5. **Agent location:** `global`
6. Click **"Continue"**
7. On the next screen, check the box next to **`portfolio-knowledge`** to link your data store
8. Click **"Create"**

#### 6.5 Configure the System Prompt

1. Once the app is created, click on it to open the agent settings
2. Click **"Agent"** in the left sidebar (inside Agent Builder)
3. Find the **"Instructions"** or **"Agent goal"** field
4. Paste this system prompt:
   ```
   You are a helpful portfolio assistant for Hafsa, a cloud and solutions
   architecture engineer. Your job is to answer questions from visitors
   about Hafsa's experience, projects, skills, and background.

   Be friendly, concise, and professional. Only answer based on the
   information in the knowledge base. If you don't know something,
   say so honestly and suggest the visitor contact Hafsa directly.

   Do not make up certifications, projects, or experience that is not
   in the knowledge base.
   ```
5. Click **"Save"**

#### 6.6 Test the Agent

1. In Agent Builder, click **"Preview"** (top right or in the left sidebar)
2. Type: `What projects has Hafsa worked on?`
3. You should get an answer based on your `portfolio-info.txt` file
4. Try: `What certifications does she have?`

#### 6.7 Get the Dialogflow Messenger Embed Code

1. In Agent Builder → your app → click **"Integrations"** in the left sidebar
2. Find **"Dialogflow Messenger"** and click it
3. Copy the **Agent ID** shown (looks like a long UUID)
4. Also copy the full embed code snippet shown on that page

#### 6.8 Add Chatbot to Your Portfolio

Open `index.html` and paste just before `</body>`:

```html
<link rel="stylesheet"
  href="https://www.gstatic.com/dialogflow-console/fast/df-messenger/prod/v1/themes/df-messenger-default.css">
<script src="https://www.gstatic.com/dialogflow-console/fast/df-messenger/prod/v1/df-messenger.js"></script>

<df-messenger
  project-id="hafsa-portfolio-gcp"
  agent-id="YOUR_AGENT_ID_FROM_CONSOLE"
  language-code="en"
  max-query-length="-1">
  <df-messenger-chat-bubble
    chat-title="Ask about Hafsa"
    chat-icon="https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/smart_toy/default/24px.svg">
  </df-messenger-chat-bubble>
</df-messenger>

<style>
  df-messenger {
    z-index: 999;
    position: fixed;
    --df-messenger-font-color: #000;
    --df-messenger-font-family: 'Segoe UI', system-ui;
    --df-messenger-chat-background: #f3f6fc;
    --df-messenger-message-user-background: #d3e3fd;
    --df-messenger-message-bot-background: #fff;
    bottom: 24px;
    right: 24px;
  }
</style>
```

Replace `YOUR_AGENT_ID_FROM_CONSOLE` with the actual Agent ID you copied.

Commit and push — GitHub Actions will auto-deploy the updated frontend.

---

### STEP 7 — Voice Agent

#### 7.1 Enable Voice in Dialogflow Messenger

1. Go back to **Agent Builder → your Portfolio Assistant app → Integrations → Dialogflow Messenger**
2. Find the **"Speech"** or **"Allow speech"** toggle and turn it **ON**
3. Under **Text-to-speech voice**, select:
   - `en-US-Neural2-F` (professional, clear) — recommended
4. Click **Save**

#### 7.2 Update the Embed Code

The embed code doesn't change — the mic icon appears automatically in the chat bubble once speech is enabled in the console. Visitors just click the mic icon to speak.

One important note: microphone access **only works over HTTPS**. Firebase Hosting already gives you HTTPS, so your live site at `hafsa-portfolio-gcp.web.app` will work perfectly.

---

### STEP 8 — Final Checks

#### Verify Everything in the Console

Go through each service and confirm:

**Firebase Hosting:**
- **console.firebase.google.com → Hosting** → should show your site URL and latest deployment timestamp

**Cloud Functions:**
- **GCP Console → Cloud Functions → visitor-counter** → Status should show a green checkmark
- Click the function → **"Testing" tab** → click "Test the function" with method POST → you should get `{"count": 1}` back

**Firestore:**
- **GCP Console → Firestore** → `visitors` collection → `counter` document → `count` field should be incrementing each time the function is tested

**GitHub Actions:**
- **github.com/YOUR_USERNAME/portfolio-backend/actions** → latest run should show green
- **github.com/YOUR_USERNAME/portfolio-frontend/actions** → same

**Agent Builder:**
- Go to your agent's **Preview** and ask it a question — confirm it responds from your data

**Live site:**
- Open `https://hafsa-portfolio-gcp.web.app`
- Visitor counter should show a number (not `—`)
- Chat bubble should appear in the bottom right
- Click chat, ask a question — AI should respond
- Click the mic icon, speak — AI should respond in voice

---

## Quick Reference — Where Everything Lives in the Console

| What you need to do | Where to go |
|---|---|
| Check function logs | Cloud Functions → visitor-counter → Logs tab |
| See Firestore data | Firestore → visitors → counter |
| See hosting deployments | Firebase Console → Hosting |
| Edit AI agent | Vertex AI → Agent Builder → your app |
| Check billing | Billing → Reports |
| Add IAM permissions | IAM & Admin → IAM |
| Service account keys | IAM & Admin → Service Accounts |

---

## Common Issues and Fixes

**Visitor counter shows `—`:** The Cloud Function URL in `counter.js` is wrong or the function isn't deployed yet. Go to Cloud Functions → visitor-counter → Trigger tab to verify the exact URL.

**Function returns 403:** The `allUsers` invoker permission wasn't set. Go to Cloud Functions → visitor-counter → Permissions → Add principal → `allUsers` → Cloud Functions Invoker.

**GitHub Actions failing:** Check that the `GCP_SA_KEY` secret contains the complete JSON (including the opening `{` and closing `}`), and that the service account has the right roles.

**Chatbot not appearing:** The `agent-id` in the embed code is wrong. Go back to Agent Builder → Integrations → Dialogflow Messenger and re-copy the exact agent ID.

**Voice not working on localhost:** Expected — it only works on HTTPS. Test on your live `web.app` URL.
