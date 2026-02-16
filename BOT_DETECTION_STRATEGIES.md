# Bot Detection & Bypass Strategies

This document outlines the bot detection challenges sites use and strategies to bypass them.

## Problem: Why Keyword Checks Fail

**Example: Pokemon Center (Incapsula WAF)**

1. **Your Request** → Axios/Puppeteer hits pokemon center
2. **WAF Response** → Incapsula intercepts, serves fake page with incident ID
3. **Your Check** → Looks for "Unavailable" in fake page
4. **Result** → Wrong stock status (page doesn't contain real content)

## Detection: What We Check For Now

### ✅ Implemented (PR #28)
- Cloudflare Challenge
- Incapsula WAF
- AWS WAF
- F5 BIG-IP

Status shows **BOT_DETECTED** when triggered.

---

## Bypass Strategies

### Strategy 1: Enhanced Puppeteer (Easy) ⭐ RECOMMENDED FIRST
**What**: Add stealth headers and longer timeouts
**Difficulty**: Easy
**Cost**: Free
**Success Rate**: 60-70% on Incapsula sites

```javascript
// In fetchWithPuppeteer():
await page.setViewport({ width: 1920, height: 1080 });
await page.setUserAgent('Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...');

// Masquerade as real browser
await page.evaluateOnNewDocument(() => {
  Object.defineProperty(navigator, 'webdriver', { get: () => false });
  Object.defineProperty(navigator, 'vendor', { get: () => 'Google Inc.' });
});

// Wait for JavaScript to render
await page.waitForNavigation({ waitUntil: 'networkidle2' });
```

**Pros**: No extra dependencies  
**Cons**: Still vulnerable to advanced WAF detection  
**Try this first before more expensive options**

---

### Strategy 2: Puppeteer Extra + Stealth Plugin (Medium) 🛡️
**What**: Use `puppeteer-extra-plugin-stealth` to mask Puppeteer
**Difficulty**: Medium
**Cost**: Free
**Success Rate**: 75-85% on Incapsula sites

```bash
npm install puppeteer-extra puppeteer-extra-plugin-stealth
```

```javascript
const PuppeteerExtra = require('puppeteer-extra');
const StealthPlugin = require('puppeteer-extra-plugin-stealth');

PuppeteerExtra.use(StealthPlugin());
const browser = await PuppeteerExtra.launch({...});
```

**Pros**: Blocks common detection vectors  
**Cons**: Slightly slower, larger bundle  
**When to use**: After Strategy 1 fails

---

### Strategy 3: Residential Proxy Rotation (Hard) 💰
**What**: Route requests through residential IPs (appear as real users)
**Difficulty**: Hard
**Cost**: $$$$ (BrightData, Oxylabs, etc.)
**Success Rate**: 95%+ on all sites

**Services**:
- BrightData: $300-500/month
- Oxylabs: $400-600/month
- Smartproxy: $50-100/month (lower quality)

**Would need**:
```javascript
const axiosInstance = axios.create({
  httpAgent: new HttpProxyAgent(`http://${proxy}:port`),
  httpsAgent: new HttpsProxyAgent(`http://${proxy}:port`),
  proxy: false
});
```

**Pros**: Works on almost any site  
**Cons**: Expensive, slow, requires API key management  
**When to use**: For high-value products (like limited Pokemon cards)

---

### Strategy 4: Headless Chrome DevTools Protocol (Hard)
**What**: Use Chrome DevTools Protocol directly instead of Puppeteer
**Difficulty**: Hard
**Cost**: Free
**Success Rate**: 50-60%

More low-level control but steeper learning curve.

---

### Strategy 5: Site-Specific Handlers (Medium)
**What**: Detect site URL and use custom scraping logic
**Difficulty**: Medium
**Cost**: Free
**Success Rate**: Varies by site

```javascript
if (agent.url.includes('pokemoncenter.com')) {
  // Custom logic for Pokemon Center
  // Maybe use API endpoints instead of scraping
  html = await fetchFromPokemonCenterAPI();
}
```

**Pros**: Can work around site-specific quirks  
**Cons**: Needs per-site maintenance  
**When to use**: For your top 5-10 sites

---

## Recommended Implementation Order

1. **Now (PR #28)**: Show `BOT_DETECTED` status ✅
2. **Next**: Try Enhanced Puppeteer (Strategy 1) - free, easy win
3. **Then**: Consider Puppeteer Stealth (Strategy 2) - if needed
4. **Future**: Site-specific handlers for high-value items (Strategy 5)
5. **Last Resort**: Residential proxies (Strategy 3) - only if spending $$$

---

## Testing Sites

### Free to Test With:
- **Amazon** (moderate bot detection) - good test
- **eBay** (medium protection) - good test
- **Pokemon Center** (Incapsula) - hardest case

### Your Current Status:
- Pokemon Center: **BOT_DETECTED** ✅ (correct!)
- Needs bypass strategy to get real stock status

---

## What's Next?

**Option A**: Implement Strategy 1 (Enhanced Puppeteer) yourself
```
Effort: 30 min
Benefit: Maybe 60-70% success rate
```

**Option B**: I can create PR with Strategy 1 + 2 combined
```
Effort: I handle it
Benefit: 75-85% success rate  
Timeline: By tomorrow
```

**Option C**: Add proxy support (costs $$$)
```
Effort: 2-3 hours
Benefit: 95%+ success
Cost: Proxy service subscription
```

**Which approach interests you?**
