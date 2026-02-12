# GitHub Actions CI/CD Setup Guide

This guide walks you through setting up automatic Cloud Functions deployment via GitHub Actions.

## Overview

When you merge a PR to `main`, GitHub Actions will automatically:
1. Check out the code
2. Install dependencies
3. Authenticate with Firebase
4. Deploy Cloud Functions to production

## Setup Instructions

### Step 1: Get Your Firebase CI Token

Run this command in your terminal (you must be authenticated with Firebase):

```bash
firebase login:ci
```

This will:
- Open a browser window
- Ask you to log in to Firebase
- Generate a **CI token** (looks like a long string)
- Display the token in your terminal

**⚠️ Important**: This token has production access. Treat it like a password!

### Step 2: Add GitHub Secrets

1. Go to your GitHub repo: **Settings → Secrets and variables → Actions**
2. Click **New repository secret**
3. Add two secrets:

#### Secret 1: FIREBASE_CI_TOKEN
- **Name**: `FIREBASE_CI_TOKEN`
- **Value**: Paste the token from Step 1
- Click **Add secret**

#### Secret 2: FIREBASE_PROJECT_ID
- **Name**: `FIREBASE_PROJECT_ID`
- **Value**: Your Firebase project ID (found in Firebase Console → Project Settings)
- Click **Add secret**

### Step 3: Verify Setup

1. Make a small change to any file in `web/functions/`
2. Create a PR to `main`
3. After merge, GitHub Actions should automatically trigger
4. Go to **Actions** tab to see the deployment in progress
5. Check the logs to confirm success

## How It Works

### Workflow Trigger
The workflow runs when:
- Code is **pushed to main** branch
- Changes are in `web/functions/` directory or the workflow file itself

### Workflow Steps
1. **Checkout**: Pull the latest code
2. **Setup Node.js**: Install Node 20 with dependency caching
3. **Install Firebase CLI**: Get the deployment tool
4. **Install Dependencies**: Run `npm install` in `web/functions/`
5. **Deploy**: Authenticate with token and deploy functions

### Automatic Deployments
- Merging a PR to `main` automatically deploys
- No manual `firebase deploy` commands needed
- All team members benefit from the same deployment pipeline

## Security Best Practices

✅ **Token Storage**
- CI token stored as GitHub secret (encrypted)
- Only accessible to GitHub Actions
- Never commit tokens to git

✅ **Project ID**
- Stored separately for easy rotation
- Not sensitive, but good practice to use secrets

✅ **Access Control**
- Only runs on pushes to `main`
- Cannot be triggered by PRs from forks
- Deployment logs visible to team members

## Troubleshooting

### "Unauthorized" Error in Deployment
**Problem**: Token is invalid or expired  
**Solution**: 
1. Go to GitHub **Settings → Secrets**
2. Delete `FIREBASE_CI_TOKEN`
3. Run `firebase login:ci` again
4. Add the new token as a secret

### Deployment Not Triggering
**Problem**: Workflow doesn't run on merge  
**Solution**:
1. Verify you're pushing to `main` (not `master`)
2. Confirm changes are in `web/functions/` directory
3. Check **Actions** tab for workflow runs
4. Verify `FIREBASE_CI_TOKEN` and `FIREBASE_PROJECT_ID` are set

### "Project ID not set" Error
**Problem**: Missing `FIREBASE_PROJECT_ID` secret  
**Solution**:
1. Find your project ID in [Firebase Console](https://console.firebase.google.com/)
2. Go to **Project Settings** (gear icon)
3. Copy the **Project ID**
4. Add it as a GitHub secret named `FIREBASE_PROJECT_ID`

### Deployment Timeout
**Problem**: Firebase deploy takes too long  
**Solution**:
1. GitHub has a default 360-minute timeout (should be fine)
2. If truly hanging, check Firebase Console for stuck operations
3. Can manually cancel the workflow in the **Actions** tab

## Manual Deployment (Fallback)

If GitHub Actions fails or you need to deploy immediately:

```bash
cd web/functions
firebase deploy --only functions
```

This will use your local Firebase authentication.

## Monitoring Deployments

### View Deployment Logs
1. Go to **Actions** tab on GitHub
2. Click the workflow run
3. Click **deploy** job
4. Expand each step to see detailed logs

### Firebase Console
Check [Firebase Console](https://console.firebase.google.com/) under:
- **Functions** → See deployed functions
- **Cloud Functions** logs to verify execution

## Advanced: Environment-Specific Deployments

To deploy to different Firebase projects (staging vs production), you can:

1. Create multiple secrets:
   - `FIREBASE_CI_TOKEN_PROD`
   - `FIREBASE_CI_TOKEN_STAGING`

2. Use conditional workflow steps:
   ```yaml
   - name: Deploy to Production
     if: github.ref == 'refs/heads/main'
     run: firebase deploy --only functions --token "${{ secrets.FIREBASE_CI_TOKEN_PROD }}"
   ```

## What Gets Deployed

The workflow deploys:
- ✅ All files in `web/functions/`
- ✅ Updated `package.json` dependencies
- ✅ New callable functions (e.g., `saveTelegram`, `testTelegram`)
- ✅ Updated `fireWebhook()` logic
- ✅ New secrets helper (reads from Google Cloud Secret Manager)

---

**Questions?** Check the deployment logs in the Actions tab for detailed error messages.
