# Security Audit Report (Revised) - Cart Rotom

**Last Updated**: February 14, 2026  
**Status**: 🟢 GOOD - Existing Firestore rules are solid

---

## Executive Summary

**Good News**: You already have **solid Firestore security rules in place**. The audit was revised after discovering your actual rules. The main remaining issues are:

1. ✅ **Firestore Security Rules** - Already implemented (better than my suggestions)
2. ⚠️ **Input Validation** - Added in PR #16 (validates all user input)
3. ✅ **Authentication Checks** - Good (all callable functions check auth)
4. ✅ **Data Encryption** - Good (payment cards, webhooks, Telegram IDs)
5. ⚠️ **Rate Limiting** - Not implemented (minor issue)

---

## Your Existing Firestore Rules (GOOD!)

```firestore
match /customers/{uid} {
  allow read: if request.auth.uid == uid;
  match /checkout_sessions/{id} { allow read, write: if request.auth.uid == uid; }
  match /subscriptions/{id} { allow read: if request.auth.uid == uid; }
  match /payments/{id} { allow read: if request.auth.uid == uid; }
}

match /users/{uid} {
  allow read, write: if request.auth.uid == uid;
  match /agents/{agentId} { allow read, write: if request.auth.uid == uid; }
}

match /products/{id} {
  allow read: if true;  // Public product catalog
  match /prices/{id} { allow read: if true; }
  match /tax_rates/{id} { allow read: if true; }
}
```

### What These Rules Protect ✅

| Path | Protection | Details |
|------|-----------|---------|
| `/customers/{uid}` | Private per-user | Only that user can read |
| `/customers/{uid}/checkout_sessions` | Private per-user | Only that user can read/write |
| `/customers/{uid}/subscriptions` | Private per-user | Only that user can read |
| `/customers/{uid}/payments` | Private per-user | Only that user can read |
| `/users/{uid}` | Private per-user | Only that user can read/write |
| `/users/{uid}/agents` | Private per-user | Only that user can read/write agents |
| `/products/*` | Public | Anyone can read products/prices/tax (for pricing page) |

### Security Properties ✅

✅ **User Isolation**: Users cannot access other users' data  
✅ **Agent Protection**: Users can only read/write their own agents  
✅ **Subscription Protection**: Users cannot see other users' subscriptions/payments  
✅ **Public Catalog**: Products are readable (needed for UI)  
✅ **Deny by Default**: Anything not explicitly allowed is denied  

---

## Cloud Functions Security Analysis

All callable functions properly check authentication:

```javascript
exports.saveWebhook = onCall(async (request) => {
    const { auth, data } = request;
    if (!auth) throw new Error("Unauthorized");  // ✅ Auth check
    const userId = auth.uid;  // ✅ Uses real user ID
    // Then accesses db.collection("users").doc(userId)...
    // ✅ Uses userId to scope access
});
```

**Status**: ✅ All functions (`saveWebhook`, `saveTelegram`, `addPaymentMethod`, `testTelegram`) properly check auth and use `auth.uid` for data scoping.

---

## Input Validation (Added in PR #16)

### Before PR #16 ❌
```javascript
saveWebhook({ webhookUrl: "' OR '1'='1" })  // No validation
saveTelegram({ userId: "DROP TABLE users" })  // No validation
addPaymentMethod({ cardNumber: "A".repeat(10000) })  // No validation
```

### After PR #16 ✅
```javascript
// Webhook URLs: Must be valid URLs, max 2048 chars
// Telegram IDs: Must be positive integers (validated)
// Card numbers: Must be 13-19 digits (validated)
// CVC: Must be 3-4 digits (validated)
// Expiry: Must be MM/YY format (validated)
```

**Status**: ⚠️ Input validation needed (PR #16 adds it)

---

## Encryption (Already Good) ✅

```javascript
// Payment cards encrypted with AES-256-GCM
const encryptedCard = encryptCardData(cardData);

// Webhook URLs encrypted with AES-256-GCM
const encryptedWebhookUrl = encrypt(webhookUrl);

// Secrets in Google Cloud Secret Manager (not hardcoded)
const telegramBotToken = await getSecret('TELEGRAM_BOT_TOKEN');
```

**Status**: ✅ Sensitive data properly encrypted

---

## Real Security Assessment

### Attack Scenarios

#### Scenario 1: Attacker with Database Access
**Can they read User A's agents?**
- BEFORE (without rules): ✅ Yes (database open)
- AFTER (with your rules): ❌ No (Firestore rules block it)
- **Result**: Protected ✅

**Can they read products/prices?**
- BEFORE: ✅ Yes
- AFTER: ✅ Yes (intentionally public)
- **Result**: Expected behavior ✅

#### Scenario 2: Attacker with Cloud Function Access
**Can they send malicious input to saveWebhook()?**
- BEFORE: ✅ Yes, could send invalid data
- AFTER (with PR #16): ❌ No, input validation rejects it
- **Result**: Protected ✅

**Can they access another user's data?**
- BEFORE: ❌ No (auth.uid check)
- AFTER: ❌ No (auth.uid + Firestore rules)
- **Result**: Double protection ✅

#### Scenario 3: XSS/Client-Side Compromise
**If attacker gets user's auth token, what can they access?**
- **Agent data**: Only that user's agents (protected by `/users/{uid}/agents` rule)
- **Payment methods**: Only their own (protected by auth check in function)
- **Subscriptions**: Only their own (protected by `/customers/{uid}` rule)
- **Other users' data**: Blocked by Firestore rules
- **Result**: Damage limited to that user ✅

---

## What Still Needs Work

### 1. ⚠️ Input Validation (PR #16 Fixes This)
**Current**: Cloud functions accept any input  
**Risk**: DOS attacks, invalid data, injection attempts  
**Fix**: PR #16 adds strict validation  
**Status**: In progress

### 2. ⚠️ Rate Limiting (Post-MVP)
**Current**: No rate limiting on Cloud Functions  
**Risk**: DOS attacks, spam  
**Fix**: Add Cloud Functions rate limiting  
**Timeline**: Post-MVP improvement

### 3. ⚠️ Audit Logging (Post-MVP)
**Current**: No logging of sensitive operations  
**Risk**: Can't track who modified what  
**Fix**: Add Cloud Audit Logs for sensitive operations  
**Timeline**: Post-MVP improvement

### 4. ⚠️ Telegram ID Encryption (Enhancement)
**Current**: Telegram IDs stored but validation needed  
**Risk**: Invalid data, potential injection  
**Fix**: PR #16 adds validation  
**Status**: In progress

---

## Recommended Security Checklist

### Already Done ✅
- [x] Firestore Security Rules (users isolated)
- [x] Authentication checks on Cloud Functions
- [x] Payment card encryption (AES-256-GCM)
- [x] Webhook URL encryption
- [x] Secrets in Google Cloud
- [x] GitHub Actions secure secret handling

### In Progress (PR #16) 🔄
- [ ] Input validation on Cloud Functions
- [ ] Telegram ID validation
- [ ] Webhook URL format validation
- [ ] Card data validation (number, CVC, expiry format)

### Future Improvements (Post-MVP) 🟡
- [ ] Rate limiting on Cloud Functions
- [ ] Audit logging for sensitive operations
- [ ] IP whitelisting (optional)
- [ ] 2FA for accounts
- [ ] PCI compliance audit
- [ ] Penetration testing

---

## Conclusion

### Security Posture: 🟢 GOOD

Your existing Firestore rules are **solid and well-designed**. They properly:
- ✅ Isolate user data
- ✅ Protect subscriptions and payments
- ✅ Allow public access to products (intentional)
- ✅ Deny everything else by default

The main gap is **input validation on Cloud Functions** (being fixed in PR #16). After PR #16 merges, your security will be:

- ✅ Backend protected (Firestore rules enforce user isolation)
- ✅ Functions protected (input validation prevents bad data)
- ✅ Data protected (encryption for sensitive fields)
- ✅ Secrets protected (in Google Cloud, not code)

### Recommendation

1. **Merge PR #16** for input validation
2. **Deploy the changes** to production
3. **Post-MVP**: Add rate limiting and audit logging

You're in good shape! 🔒

---

## Appendix: How The Rules Work

### User A tries to read User B's agents:
```javascript
// Firestore evaluates this rule:
match /users/{uid} {
  allow read, write: if request.auth.uid == uid;
}

// For User A reading User B's data:
// request.auth.uid = "user-a-id"
// uid = "user-b-id"
// "user-a-id" == "user-b-id" ??? NO
// Result: ❌ DENIED
```

### User A reads their own agents:
```javascript
// request.auth.uid = "user-a-id"  
// uid = "user-a-id"
// "user-a-id" == "user-a-id" ??? YES
// Result: ✅ ALLOWED
```

### Anyone reads products:
```javascript
match /products/{id} {
  allow read: if true;
}

// No auth check needed - always true
// Result: ✅ ALLOWED (for anyone, even unauthenticated)
```

---

**Questions?** Review the actual rules above and let me know if you'd like any adjustments.
