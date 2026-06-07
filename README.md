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

## Architecture

Everything is deployed via CLI commands and the GCP Console. Two GitHub repos — one for frontend and one for backend — each with their own GitHub Actions workflow.

> **Why skip the Load Balancer?**
> A GCP Load Balancer costs ~$0.60/day ($4.20/week) — nearly your entire $5 budget. Firebase Hosting is actually backed by Google Cloud Storage and gives you free HTTPS + global CDN + a clean `.web.app` URL. For a beginner demo, it's the right call.

## Prerequisites

- A Google account with $5 GCP credits activated
- **gcloud CLI** installed — `brew install google-cloud-sdk` on Mac, or download from cloud.google.com/sdk
- **Node.js** installed (for Firebase CLI) — `brew install node`
- A **GitHub account** with two new empty repos: `portfolio-frontend` and `portfolio-backend`
- Python 3.11 installed (for Cloud Functions)
- A text editor (VS Code recommended)

---

## Step 1 — Set Up GCP Project

*~10 minutes · One-time setup*

Everything in GCP lives inside a **project**. Think of it as a folder that groups all your services and tracks costs together.

### 1.1 Login and create project

```bash
# Login to Google Cloud
gcloud auth login

# Create a new project (change the name to something unique)
gcloud projects create hafsa-portfolio-gcp --name="Hafsa Portfolio"

# Set this as your active project
gcloud config set project hafsa-portfolio-gcp

# Check your billing account ID (you'll need this)
gcloud billing accounts list
```

### 1.2 Link billing and enable APIs

Copy the `ACCOUNT_ID` from the command above (looks like `01A2B3-CD4E56-789F0A`), then run:

```bash
# Link your billing account (paste your actual ID)
gcloud billing projects link hafsa-portfolio-gcp \
  --billing-account=YOUR_BILLING_ACCOUNT_ID

# Enable all the services you'll use
gcloud services enable \
  cloudfunctions.googleapis.com \
  cloudbuild.googleapis.com \
  firestore.googleapis.com \
  run.googleapis.com \
  dialogflow.googleapis.com \
  texttospeech.googleapis.com
```

> Each API needs to be enabled before you can use it. This takes about 2 minutes and only needs to be done once per project.

### 1.3 Create Firestore database

```bash
# Create Firestore in Native mode (choose region closest to Pakistan)
# asia-south1 = Mumbai, best for Pakistan
gcloud firestore databases create --location=asia-south1
```

Go to **console.cloud.google.com → Firestore** and create:

1. Collection: `visitors`
2. Document ID: `counter`
3. Field: `count` (Number) = `0`

---

## Step 2 — Build the Portfolio Site

*~20 minutes · HTML + CSS*

### Folder structure

```
portfolio-frontend/
├── index.html
├── style.css
├── counter.js            ← visitor counter (added in Step 4)
├── .github/
│   └── workflows/
│       └── deploy.yml    ← CI/CD (added in Step 6)
└── firebase.json         ← Firebase config (added in Step 3)
```

### `index.html`

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

  <!-- Visitor counter script -->
  <script src="counter.js"></script>
  <!-- Dialogflow Messenger (chatbot + voice) - added in Step 7 -->
</body>
</html>
```

### `style.css`

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

---

## Step 3 — Deploy to Firebase Hosting

*~15 minutes · Free HTTPS + CDN*

Firebase Hosting gives you free HTTPS, a global CDN, and a `.web.app` URL — no Load Balancer needed.

### 3.1 Install Firebase CLI and login

```bash
npm install -g firebase-tools
firebase login
```

### 3.2 Initialize Firebase in your frontend folder

```bash
cd portfolio-frontend
firebase init hosting
```

When prompted:

1. Select **Use an existing project** → choose `hafsa-portfolio-gcp`
2. Public directory: **`.`** (just a dot — deploy the current folder)
3. Configure as single-page app? **No**
4. Overwrite index.html? **No**

### 3.3 Create `firebase.json`

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

### 3.4 Deploy

```bash
firebase deploy --only hosting
```

> **Your site is live!** Firebase gives you a URL like `hafsa-portfolio-gcp.web.app`. Open it in your browser — your portfolio is live with HTTPS, on Google's global CDN, for free.

---

## Step 4 — Visitor Counter (Firestore + JS)

*~15 minutes · Shows live visitor count*

The visitor counter calls your Cloud Function (built in Step 5), which reads and writes to Firestore. The JS in your HTML fetches the count and displays it.

### `counter.js`

```javascript
// Replace this URL after you deploy the Cloud Function in Step 5
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

// Run when page loads
document.addEventListener('DOMContentLoaded', updateVisitorCount);
```

> **Note:** The `FUNCTION_URL` will be blank until Step 5. Deploy the Cloud Function first, get the URL, then paste it here and redeploy the frontend.

---

## Step 5 — Cloud Functions (Python API)

*~20 minutes · Serverless backend*

Cloud Functions is serverless Python — you write a function, deploy it, and Google runs it on demand. Free tier covers 2 million requests per month.

### 5.1 Backend folder structure

```
portfolio-backend/
├── main.py           ← your Cloud Function
├── requirements.txt  ← Python dependencies
└── test_main.py      ← unit tests (required by challenge)
```

### 5.2 `main.py`

```python
import functions_framework
from google.cloud import firestore
from flask import jsonify

# Create Firestore client (uses your project automatically)
db = firestore.Client()

@functions_framework.http
def visitor_counter(request):
    """Increments visitor count in Firestore and returns the new count."""
    # Handle CORS so your website can call this function
    cors_headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type',
    }

    # Browser sends OPTIONS first (a preflight check) — just say OK
    if request.method == 'OPTIONS':
        return ('', 204, cors_headers)

    # Get the counter document from Firestore
    doc_ref = db.collection('visitors').document('counter')
    doc = doc_ref.get()

    if doc.exists:
        current_count = doc.to_dict().get('count', 0)
        new_count = current_count + 1
    else:
        new_count = 1

    # Save the updated count back to Firestore
    doc_ref.set({'count': new_count})

    return jsonify({'count': new_count}), 200, cors_headers
```

### 5.3 `requirements.txt`

```
functions-framework==3.*
google-cloud-firestore==2.*
```

### 5.4 `test_main.py`

The Cloud Resume Challenge requires tests. Here's a simple but real test suite:

```python
import unittest
from unittest.mock import patch, MagicMock
import main

class TestVisitorCounter(unittest.TestCase):

    @patch('main.db')
    def test_increments_existing_count(self, mock_db):
        """When document exists, count should go up by 1."""
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
        """When no document exists yet, count starts at 1."""
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
        """OPTIONS request (CORS preflight) should return 204."""
        request = MagicMock()
        request.method = 'OPTIONS'
        response = main.visitor_counter(request)
        self.assertEqual(response[1], 204)

if __name__ == '__main__':
    unittest.main()
```

### 5.5 Run tests locally first

```bash
cd portfolio-backend
pip install functions-framework google-cloud-firestore
python -m pytest test_main.py -v
```

### 5.6 Deploy the function

```bash
gcloud functions deploy visitor-counter \
  --gen2 \
  --runtime=python311 \
  --region=asia-south1 \
  --source=. \
  --entry-point=visitor_counter \
  --trigger-http \
  --allow-unauthenticated \
  --max-instances=5
```

**If you get a permissions error** like `Permission 'iam.serviceaccounts.actAs' denied`, run:

```bash
gcloud projects add-iam-policy-binding <project-id> \
  --member="user:EMAIL@gmail.com" \
  --role="roles/iam.serviceAccountUser"
```

After deploying, copy the **URL** shown in the output (e.g. `https://asia-south1-hafsa-portfolio-gcp.cloudfunctions.net/visitor-counter`) and paste it into `counter.js` in your frontend.

---

## Step 6 — CI/CD with GitHub Actions

*~20 minutes · Auto-deploy on every push*

CI/CD means: whenever you push code to GitHub, it automatically tests and deploys for you.

**How it works:**
1. Push to `portfolio-backend` → GitHub Actions runs Python tests → if tests pass, deploys to Cloud Functions
2. Push to `portfolio-frontend` → GitHub Actions deploys to Firebase Hosting

### 6.1 Create a GCP service account for GitHub

```bash
# Create the service account
gcloud iam service-accounts create github-deployer \
  --display-name="GitHub Actions Deployer"

# Give it permission to deploy Cloud Functions
gcloud projects add-iam-policy-binding hafsa-portfolio-gcp \
  --member="serviceAccount:github-deployer@hafsa-portfolio-gcp.iam.gserviceaccount.com" \
  --role="roles/cloudfunctions.developer"

# Also allow it to use the App Engine default service account
gcloud iam service-accounts add-iam-policy-binding \
  hafsa-portfolio-gcp@appspot.gserviceaccount.com \
  --member="serviceAccount:github-deployer@hafsa-portfolio-gcp.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# Download a key file for this service account
gcloud iam service-accounts keys create github-key.json \
  --iam-account=github-deployer@hafsa-portfolio-gcp.iam.gserviceaccount.com
```

> ⚠️ **Never commit `github-key.json` to Git!** Add it to `.gitignore` immediately.

```bash
echo "github-key.json" >> .gitignore
```

### 6.2 Add secrets to GitHub

Go to each GitHub repo → **Settings → Secrets and variables → Actions → New repository secret**:

- In `portfolio-backend`: add secret `GCP_SA_KEY` = paste the entire contents of `github-key.json`
- In `portfolio-frontend`: add secret `GCP_SA_KEY` = same key. Also add `FIREBASE_TOKEN` = output of `firebase login:ci`

### 6.3 Backend CI/CD workflow

```yaml
# portfolio-backend/.github/workflows/deploy.yml
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

      # If any test fails, deployment stops here
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

### 6.4 Frontend CI/CD workflow

```yaml
# portfolio-frontend/.github/workflows/deploy.yml
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

### 6.5 Test it

```bash
cd portfolio-backend
git add .
git commit -m "Add visitor counter Cloud Function"
git push origin main
# Watch it deploy automatically at:
# github.com/YOUR_USERNAME/portfolio-backend/actions
```

> **Demo tip:** During your presentation, make a small edit to `index.html`, push to GitHub, then show the GitHub Actions tab auto-deploying. Takes ~90 seconds and always impresses.

---

## Step 7 — AI Chatbot with Vertex AI Agent Builder

*~25 minutes · Gemini-powered, GCP-native*

Vertex AI Agent Builder is Google's platform for building AI agents powered by Gemini. You give it information about yourself and it creates a chatbot that can answer any question about you — without writing any ML code. It uses RAG (Retrieval-Augmented Generation) to ground answers in your actual data.

### 7.1 Create a data store

1. Go to **console.cloud.google.com → Vertex AI → Agent Builder**
2. Click **Create data store**
3. Choose **Website** if your LinkedIn is public, OR **Cloud Storage** to upload a document
4. If using a document: create `portfolio-info.txt` (see template below), upload to a GCS bucket, and link that bucket
5. Name the data store: `portfolio-knowledge`
6. Wait ~5 minutes for indexing

### `portfolio-info.txt` template

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

Contact: [your LinkedIn or email]
```

### 7.2 Create the AI agent

1. In Agent Builder → **Create app** → choose **Chat**
2. Name it: `Portfolio Assistant`
3. Link the data store (`portfolio-knowledge`)
4. In **Agent settings**, set this system prompt:

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

5. Click **Save**, then **Preview** the agent — try asking "What projects has Hafsa worked on?"
6. Go to **Integrations → Dialogflow Messenger** and copy the embed code

### 7.3 Add the chatbot to your portfolio

Paste just before `</body>` in `index.html`:

```html
<!-- Paste YOUR embed code from Dialogflow Messenger here -->
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

---

## Step 8 — Voice Agent (Dialogflow Messenger + Cloud TTS)

*~15 minutes · The showstopper feature*

Dialogflow Messenger has a built-in voice mode. When a visitor clicks the mic, the browser captures their voice (Web Speech API), sends it as text to your agent, and the response is read aloud using Cloud Text-to-Speech — all built into the same widget from Step 7.

### 8.1 Enable voice in Dialogflow Messenger

1. Go to **Agent Builder → your agent → Integrations → Dialogflow Messenger**
2. Toggle on **Allow speech**
3. Under **Text-to-speech voice**, select a neural voice — recommended: `en-US-Neural2-F` (clear, professional) or `en-US-Chirp3-HD-Aoede` (HD quality)
4. Click **Save**

### 8.2 Updated embed code with voice support

```html
<df-messenger
  project-id="hafsa-portfolio-gcp"
  agent-id="YOUR_AGENT_ID"
  language-code="en"
  max-query-length="-1">
  <df-messenger-chat-bubble
    chat-title="Ask about Hafsa"
    chat-icon="https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/smart_toy/default/24px.svg">
  </df-messenger-chat-bubble>
</df-messenger>
```

> ⚠️ **Microphone access:** Visitors must grant microphone permission in their browser. This only works over HTTPS (which Firebase Hosting provides) — it won't work on plain `http://` localhost.

### 8.3 Test voice locally with ngrok

```bash
# Install ngrok from ngrok.com, then:
npx serve .       # serve your HTML on port 3000
ngrok http 3000   # get a https://xxxx.ngrok.io URL
```

### 8.4 What visitors experience

1. They open your portfolio at `hafsa-portfolio-gcp.web.app`
2. They see a chat bubble in the bottom right: "Ask about Hafsa"
3. They can **type** a question — the AI answers in text
4. They can click the **mic icon**, speak their question, and the AI responds in a natural-sounding voice (powered by Cloud TTS Neural2/Chirp voices)

> **Demo script:** Open your portfolio live. Ask the voice agent: *"Tell me about Hafsa's final year project."* Watch the audience react when the AI speaks back in a clear, professional voice — explaining quantum computing and portfolio optimization — from a chatbot grounded in your own data.

---

## Step 9 — Demo Checklist & Cost Summary

### Pre-demo checklist

- [ ] Portfolio site live at `.web.app` URL
- [ ] Visitor counter showing a real number
- [ ] Chat bubble visible bottom-right
- [ ] Typed chat question works (test it)
- [ ] Voice input works (mic grants permission)
- [ ] Voice output speaks the response
- [ ] GitHub Actions workflow showing green (passed)
- [ ] You can make a live push and show CI/CD deploying

### Demo flow (10 minutes)

1. **Show the live portfolio** — open `hafsa-portfolio-gcp.web.app`. Point out HTTPS, global CDN, free hosting.
2. **Visitor counter** — refresh the page, show the number incrementing. Explain the Firestore → Cloud Functions → JavaScript chain.
3. **The chatbot** — type "What is Hafsa's FYP about?" Show the AI answer grounded in real data (not hallucinated).
4. **Voice agent** — click the mic, ask "What certifications does she have?" — the AI responds in voice. This is your showstopper.
5. **CI/CD** — make a tiny edit to `index.html`, push to GitHub, open the Actions tab and show the workflow running live.
6. **Architecture** — explain which GCP service handles which part. Mention Firestore free tier, Cloud Functions scale-to-zero, Firebase CDN.

### Cost breakdown

| Service | Free tier | 1-week demo cost |
|---|---|---|
| Firebase Hosting | 10 GB/month, 360 MB/day transfer | $0.00 |
| Cloud Functions | 2M invocations/month | $0.00 |
| Firestore | 50K reads, 20K writes/day | $0.00 |
| GitHub Actions | 2,000 min/month (public repos) | $0.00 |
| Vertex AI Agent Builder | ~500 chat queries/month | $0.00 for demo |
| Dialogflow Messenger | Text: $0.007/session, Audio: $0.065/min | ~$0.07 |
| Cloud TTS | 4M chars/month (Standard), 1M (Neural2) | $0.00 |
| GCS (function source) | 5 GB free storage | $0.00 |
| **Total estimated** | | **~$0.07–$0.10** |

> Your $5 covers ~7 months of this setup running live. The only ongoing cost is Dialogflow voice queries — a few cents per week if visitors actually use the voice agent.

### Blog post ideas (Challenge Step 16)

- Why I chose Firebase Hosting over a Load Balancer — and what the tradeoffs are
- How Vertex AI Agent Builder uses RAG to ground the chatbot in your own data
- What CORS is and why my Cloud Function returns `Access-Control-Allow-Origin: *`
- What happened when tests failed in CI/CD and blocked the deploy
- The difference between Cloud Functions Gen1 and Gen2
