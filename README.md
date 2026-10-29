# Chatham-Kent Living Opportunity Atlas — GitHub Pages + live publishing API

An interactive, **synthetic** Chatham-Kent simulation adapted from ten uploaded atlas PDFs. The existing `index.html` visualization is preserved and extended with a browser-to-API publisher. **No actual transactions or observed live economic measurements are implied.**

## Architecture

`GitHub Pages index.html` → **POST** simulation snapshot with private bearer token → `Cloudflare Worker` → `Cloudflare KV` → **GET** public JSON feed → monitoring apps.

GitHub Pages is free static hosting; it **cannot receive POSTs or persist shared live state**. Cloudflare Worker + KV is the small additional backend. Free plans have limits, which can change. The browser publishes every **120 seconds** and on control changes, throttled to **30 seconds**. Monitoring clients can poll every **10 seconds**; polling faster does **not** produce faster updates. When the publishing tab closes, the feed retains its last snapshot and becomes **stale after 5 minutes**. KV is eventually consistent; updates may take time to appear globally. Multiple simultaneous publishers overwrite the same `latest` snapshot: use one authorized publisher.

## 1. Publish the website on GitHub Pages

1. Create a **public** GitHub repository.
2. Upload the **contents** of this ZIP to the repository root (not the enclosing folder).
3. Go to **Settings → Pages → Build and deployment → Deploy from a branch → main → /(root) → Save**.
4. Open `https://YOUR-USERNAME.github.io/YOUR-REPO/` after deployment.

`api/baseline.json` is a static, publicly readable **reference** dataset. It is **not** the live feed.

## 2. Deploy the free-tier API with Cloudflare

1. Create a free Cloudflare account and install Node.js locally.
2. From this repository run `cd worker` and `npx wrangler login`.
3. Create KV storage: `npx wrangler kv namespace create SIM_STATE`. Copy the returned namespace ID.
4. Edit `worker/wrangler.toml`: uncomment the `[[kv_namespaces]]` section and insert that ID. The binding name must remain `SIM_STATE`.
5. Set a long, random **publisher secret**: `npx wrangler secret put PUBLISH_TOKEN` (paste secret when prompted). Do **not** commit it. If the Worker is not deployed yet, deploy first and then set the secret.
6. Deploy: `npx wrangler deploy` from the `worker/` directory. Save the Worker URL shown, e.g. `https://chatham-kent-simulation-api.YOUR-SUBDOMAIN.workers.dev`.
7. Check `https://YOUR-WORKER.workers.dev/api/health` in a browser.

Cloudflare's free KV write allowance may be exceeded if you publish too frequently or from many tabs. Keep one publishing browser and the default 120-second interval. Review current plan quotas before production use.

## 3. Start publishing

1. Open your GitHub Pages website.
2. Scroll to **Live API publishing**.
3. Paste your Worker base URL and private `PUBLISH_TOKEN`.
4. Click **Start live publishing** and leave that browser tab open.
5. Click **Copy live feed URL**. Your monitoring app can now `GET` that URL without the secret.

The secret is held in browser session storage, **not** in the public GitHub files. Anyone who obtains it can overwrite your feed. Use a private publishing device and rotate the secret if exposed. The Worker permits cross-origin requests; publication is protected by the bearer token, while reads are intentionally public.

## API

- `GET /api/health` — health check.
- `GET /api/live` — latest published snapshot, freshness status, age in seconds. No token.
- `POST /api/live` — publish synthetic snapshot. Requires `Authorization: Bearer YOUR_SECRET` and JSON payload of the documented schema.

Example response:

```json
{
  "status": "fresh",
  "live": true,
  "ageSeconds": 10,
  "refreshRecommendedSeconds": 10,
  "data": {
    "schemaVersion": 1,
    "dataType": "synthetic",
    "observed": false,
    "receivedAt": "2026-10-29T15:00:00.000Z",
    "metrics": {
      "expectedPaidTasksPerDay": 45,
      "conditionalGrossCadPerDay": 26781
    }
  }
}
```

This is an **illustrative abbreviated response**, not an actual deployed response.

## App integration

See `examples/monitor.html` for a complete browser-based 10-second monitoring app. JavaScript example:

```js
const feed = 'https://YOUR-WORKER.workers.dev/api/live';
setInterval(async () => {
  const response = await fetch(feed, { cache: 'no-store' });
  const { status, ageSeconds, data } = await response.json();
  if (status !== 'fresh') console.warn('Publisher offline or stale', ageSeconds);
  else console.log(data.metrics, data.simulation);
}, 10000);
```

## Provenance and limitations

The 14 prices and 6,053 per-tier inferred links are drawn from the uploaded Chatham-Kent atlases. The baseline uses 25,000 synthetic residents, 12% daily participation, 1.5% paid conversion, 140 task concepts, and 1,189 business records. Business-task links are inferred, **not** confirmed transactions. The existing visual terrain uses illustrative sector allocation, not the full source crosswalk. The Canadian capital sector totals are a historical August 2025 snapshot, not a real-time exchange feed. Simulated sphere rotation, trajectory animations and simulated clock do not represent real-world speed or movement.

## Security and production notes

This is a simple single-publisher proof of concept, not a hardened multi-tenant service. Use HTTPS, keep the secret out of source control, monitor quotas, and consider origin restrictions, schema validation improvements, abuse controls and separate publisher identities before scaling. `GET /api/live` is public by design. Worker KV is eventually consistent, so this is **near-live periodic snapshot monitoring**, not guaranteed instantaneous streaming. The publisher is browser-dependent and stops on close/sleep/background throttling.
