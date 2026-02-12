# Telegram Notifications Setup Guide

## Overview
Cart Rotom now supports sending stock alerts directly to your Telegram account via a Cart Rotom Bot.

## Prerequisites
1. A Telegram account
2. Access to Google Cloud Console for your Firebase project
3. Firebase Cloud Functions enabled

## Step 1: Create a Telegram Bot

1. Open Telegram and search for **@BotFather**
2. Click "Start" and send the command `/newbot`
3. Follow the prompts:
   - **Name**: `Cart Rotom Bot`
   - **Username**: `CartRotomBot`
4. BotFather will return a **token** that looks like: `123456789:ABCDEfghijklmnopqrstuvwxyz123456789`
5. **Save this token** — you'll need it in the next step

## Step 2: Store Token in Google Cloud Secret Manager

### Using Google Cloud Console (Recommended for Production)

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Navigate to **Secret Manager** (search in top bar)
3. Click **Create Secret**
4. Fill in:
   - **Name**: `TELEGRAM_BOT_TOKEN`
   - **Secret value**: Paste the token from BotFather
   - **Replication**: Automatic
5. Click **Create Secret**
6. Grant your Cloud Functions service account access:
   - Click the secret you just created
   - Click **Manage Roles** in the sidebar
   - Under "Principal", enter: `<YOUR-PROJECT-ID>@appspot.gserviceaccount.com`
   - Assign role: **Secret Accessor**
   - Click **Save**

### Using Firebase CLI (Development)

```bash
# Set the secret in your Firebase project
firebase functions:secrets:set TELEGRAM_BOT_TOKEN

# When prompted, paste the token from BotFather
# Then deploy functions
firebase deploy --only functions
```

## Step 3: Update and Deploy Code

```bash
cd web
npm install  # Install new dependency (@google-cloud/secret-manager)
firebase deploy --only functions
```

## Step 4: Connect in Cart Rotom UI

1. Open Cart Rotom Settings → Notifications tab
2. Select **Telegram** as your notification method
3. Instructions will tell you to message `@CartRotomBot`
4. Send `/start` to the bot in Telegram
5. The bot will reply with your **Telegram User ID**
6. Paste that ID into Cart Rotom and click "Connect Telegram"
7. Click "Test Telegram" to verify it works

## How It Works

When an item you're monitoring goes in/out of stock:
- Cart Rotom checks the product
- The Cloud Function detects the status change
- If Telegram is configured, it sends you a direct message via @CartRotomBot
- Message format: `✅ Product Name` with URL and timestamp

## Security

- Bot tokens are stored in Google Cloud Secret Manager (encrypted at rest)
- Telegram User IDs are encrypted in Firestore before storage
- Tokens are cached in-memory for 1 hour to reduce API calls
- Cloud Functions authenticate all requests via Firebase Auth
- Secrets are retrieved with proper service account permissions

## Troubleshooting

### "Telegram bot not configured" error
- Verify `TELEGRAM_BOT_TOKEN` secret exists in Google Cloud Secret Manager
- Check the Cloud Functions service account has **Secret Accessor** role
- Redeploy: `firebase deploy --only functions`

### Test button doesn't send message
- Verify your Telegram user ID is correct
- Make sure you've started @CartRotomBot (`/start`)
- Check Cloud Functions logs: `firebase functions:log`

### "Failed to retrieve secret" in logs
- Ensure secret name is exactly `TELEGRAM_BOT_TOKEN`
- Verify service account permissions in Secret Manager
- Check your GCP project ID is correct

## Switching Between Webhook and Telegram

You can switch notification methods anytime in Settings → Notifications without losing your configuration.

---

For issues, check Cloud Functions logs:
```bash
firebase functions:log --follow
```
