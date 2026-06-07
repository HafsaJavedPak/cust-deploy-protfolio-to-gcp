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

### Step 1 — Set Up GCP Project & Enable APIs

*Estimated time: 10 minutes*

Everything in GCP lives inside a project. We will create the project, link billing, and activate the necessary services manually.

**1.1 Create the Project**

1. Open the [Google Cloud Console](https://console.cloud.google.com/).
2. Click the **Project Selector** dropdown at the top left of the screen, then click **New Project**.
3. Name your project (e.g., `Hafsa Portfolio`) and click **Create**.
4. Once created, ensure your new project is selected in the top dropdown.

**1.2 Link Billing**

1. Open the left navigation menu (hamburger icon) and select **Billing**.
2. Click **Link a Billing Account** and select the account with your active $5 credits.

**1.3 Enable Required APIs**

1. Go to **APIs & Services** > **Library**.
2. Use the search bar to find and click **Enable** for each of the following:
* Cloud Functions API
* Cloud Build API
* Cloud Run API
* Firestore API
* Dialogflow API
* Cloud Text-to-Speech API



---

### Step 2 — Create the Firestore Database

*Estimated time: 5 minutes*

This will act as the backend storage for your visitor counter.

1. In the GCP Console search bar, type **Firestore** and select it.
2. Click **Create Database**.
3. Select **Native mode**.
4. For the location, select **asia-south1** (Mumbai) to ensure the lowest latency for users accessing the site from Pakistan. Click **Create Database**.
5. Once initialized, click **Start Collection**.
6. Set the Collection ID to `visitors`.
7. Set the Document ID to `counter`.
8. Add a field: Name it `count`, change the Type to `Number`, and set the Value to `0`.
9. Click **Save**.

---

### Step 3 — Deploy the Cloud Function (Python API)

*Estimated time: 15 minutes*

Instead of deploying via `gcloud`, we will use the inline editor in the Console.

1. Search for **Cloud Functions** in the top bar and select it.
2. Click **Create Function**.
3. **Configuration:**
* Environment: **2nd gen**
* Function name: `visitor-counter`
* Region: **asia-south1**


4. **Trigger:**
* Type: **HTTPS**
* Authentication: Select **Allow unauthenticated invocations** (crucial so your public website can trigger it).
* Click **Save**, then click **Next**.


5. **Code Setup:**
* Runtime: **Python 3.11**
* Entry point: `visitor_counter`
* In the inline editor, replace the contents of `main.py` with your provided backend Python code.
* Switch to `requirements.txt` in the editor and paste your dependencies (`functions-framework==3.*` and `google-cloud-firestore==2.*`).


6. Click **Deploy**.
7. Once the deployment finishes (indicated by a green checkmark), click on the function name, go to the **Trigger** tab, and **copy the Trigger URL**. You will need this for your frontend code.

---

### Step 4 — Build & Push Frontend Code via GitHub Web UI

*Estimated time: 10 minutes*

1. Go to your `portfolio-frontend` repository on GitHub.
2. Click **Add file** > **Create new file**.
3. Create `index.html` and `style.css` using the exact code you provided.
4. Create `counter.js`. Paste your JS code, but **replace the `FUNCTION_URL` variable** with the Cloud Function Trigger URL you copied in Step 3.
5. Create a file named `firebase.json` and paste your provided JSON configuration.
6. Commit these files directly to your `main` branch.

---

### Step 5 — CI/CD with GitHub Actions (No Local Setup)

*Estimated time: 15 minutes*

We will create a Service Account in the GCP Console to allow GitHub to deploy on your behalf.

**5.1 Create the Service Account**

1. In the GCP Console, go to **IAM & Admin** > **Service Accounts**.
2. Click **Create Service Account**. Name it `github-deployer` and click **Create and Continue**.
3. Grant the following roles:
* **Cloud Functions Developer** (for backend updates)
* **Service Account User** (required to attach the service account to functions)
* **Firebase Hosting Admin** (for frontend updates)


4. Click **Done**.
5. Click on the newly created service account, go to the **Keys** tab, click **Add Key** > **Create new key**. Choose **JSON** and download it to your computer.

**5.2 Configure GitHub Secrets**

1. Go to your frontend GitHub repository > **Settings** > **Secrets and variables** > **Actions**.
2. Click **New repository secret**.
3. Name it `GCP_SA_KEY` and paste the *entire* contents of the downloaded JSON file. Save it.
4. Repeat this exact process for your backend repository.

**5.3 Create the CI/CD Workflows**

1. In your `portfolio-frontend` repo, click **Add file** > **Create new file**.
2. Name the path exactly: `.github/workflows/deploy.yml`.
3. Paste your frontend workflow code. **Crucial modification:** Because we bypassed the local Firebase CLI, update the deploy step in your YAML to use the Service Account directly instead of a Firebase token:
```yaml
   - name: Deploy to Firebase Hosting
     uses: FirebaseExtended/action-hosting-deploy@v0
     with:
       repoToken: ${{ secrets.GITHUB_TOKEN }}
       firebaseServiceAccount: ${{ secrets.GCP_SA_KEY }}
       channelId: live
       projectId: hafsa-portfolio-gcp

```
4. Commit the file. Click the **Actions** tab in GitHub to watch your site deploy automatically.
5. (Optional) Repeat the workflow creation for your `portfolio-backend` repo using your provided backend YAML, ensuring the `credentials_json` points to `${{ secrets.GCP_SA_KEY }}`.

---

### Step 6 — Set up Firebase Hosting
*Estimated time: 5 minutes*

Your GitHub Action just attempted to deploy to Firebase, but we need to activate it in the console first.

1. Go to the [Firebase Console](https://console.firebase.google.com/).
2. Click **Add project** and select your existing `hafsa-portfolio-gcp` project. Confirm the billing plan.
3. On the left sidebar, click **Build** > **Hosting**.
4. Click **Get Started**. 
5. You can click "Next" through the CLI instruction screens since your GitHub Action is already handling the deployment process. 
6. Re-run your GitHub Action in the `portfolio-frontend` repo. It will now succeed and output your live `.web.app` URL.

---

### Step 7 — AI Chatbot & Voice Agent Setup
*Estimated time: 20 minutes*

This step is natively designed for the console. 

**7.1 Create the Data Store**
1. In the GCP Console, search for **Vertex AI** and navigate to **Agent Builder**.
2. Click **Create data store** and select **Cloud Storage** (or Website if preferred).
3. Open a new tab, go to **Cloud Storage** > **Buckets**, and click **Create**. Name it something like `hafsa-portfolio-data`.
4. Upload your `portfolio-info.txt` to this bucket.
5. Back in Agent Builder, select the bucket you just created, name the data store `portfolio-knowledge`, and wait a few minutes for indexing.

**7.2 Create the Agent**
1. In Agent Builder, click **Create app** > **Chat**.
2. Name it `Portfolio Assistant` and link your `portfolio-knowledge` data store.
3. In **Agent settings**, paste your provided system prompt regarding your FYP, cloud certifications, and teaching roles.
4. Save and test the agent in the Preview window.

**7.3 Enable Voice & Embed**
1. In the Agent Builder, go to **Integrations** > **Dialogflow Messenger**.
2. Toggle on **Allow speech**.
3. Select a neural voice (e.g., `en-US-Neural2-F`).
4. Copy the generated HTML embed code.
5. Go back to your GitHub Web UI, open `index.html` in the frontend repo, click the pencil icon to edit, and paste the embed code just before the closing `</body>` tag along with your provided custom `<style>` block.
6. Commit the changes. Your GitHub Action will automatically rebuild and deploy the site.

Your serverless portfolio, complete with a live backend database, auto-deployment pipelines, and an interactive AI voice agent, is now fully functional and accessible via your Firebase URL.

```
