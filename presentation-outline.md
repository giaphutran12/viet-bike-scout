# Vietnam Bike Price Scout — Speaker Notes

**AI Tinkerers HCMC** | "From Prompt to Agent" | 5 min | In-person, projector

---

## Setup (before your slot)

- **Tab 1**: `viet-bike-scout.vercel.app` — live search running (HCMC, all bike types, cache OFF). Start this 1-2 min early.
- **Tab 2**: Same app — cache ON, same city, results already loaded (your backup).
- **Tab 3**: VS Code with `src/app/api/search/route.ts` open.

---

## 0:00 – 0:45 | The Hook

**SHOW: Tab 1 — app with live browser agent iframes running**

- Renting a motorbike in Vietnam — no aggregator, no Kayak, nothing. You're checking 5-10 websites, all different layouts, different currencies.
- So I built an app that sends AI browser agents to all of them at the same time.
- *(point at the iframe grid)* What you're seeing right now are real browser sessions. Not a mockup. Each one is navigating a different rental shop, handling popups, converting VND to USD. All live, all in parallel.

---

## 0:45 – 1:45 | The API — "It's just a fetch call"

**SHOW: Tab 3 — VS Code, `route.ts` lines 185-197**

```typescript
const response = await fetch(TINYFISH_SSE_URL, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Accept: "text/event-stream",
    "X-API-Key": apiKey,
  },
  body: JSON.stringify({
    url,
    goal: GOAL_PROMPT,
  }),
  signal: controller.signal,
});
```

- This is the entire TinyFish integration. 11 lines. Just a fetch call.
- You give it a URL and a goal in plain English. It gives you structured JSON back.
- No SDK, no complex setup. Just `url` and `goal`.

**Scroll to GOAL_PROMPT (lines 41-80)**

- I just tell it in plain English — go to the pricing page, handle any popups, grab all the bikes, click through pages.
- TinyFish figures out the rest.

---

## 1:45 – 2:45 | The Parallelism — "Promise.allSettled, that's it"

**SHOW: `route.ts` lines 341-361**

```typescript
const tasks = uncachedSites.map((url) =>
  (async () => {
    return runTinyFishSseForSite(url, apiKey, siteEnqueue);
  })(),
);

const settled = await Promise.allSettled(tasks);
```

- The parallelism? It's literally just `Promise.allSettled`. Fire all sites at once. No staggering, no rate limiting.
- TinyFish has no concurrency caps — I send 5-6 requests at once and each one spins up its own browser agent.

**Briefly show CITY_SITES config (lines 12-39)**

- That's 18 shops across 4 cities — all niche local sites, none of them have an API — and they all get scraped at the same time.

---

## 2:45 – 3:45 | The Streaming — "Results arrive as they finish"

**SHOW: `route.ts` lines 207-240 — SSE reader**

```typescript
const reader = response.body.getReader();
const decoder = new TextDecoder();
let buffer = "";

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  buffer += decoder.decode(value, { stream: true });
  const lines = buffer.split("\n");
  buffer = lines.pop() ?? "";

  for (const line of lines) {
    if (!line.startsWith("data: ")) continue;
    const event = JSON.parse(line.slice(6));

    if (event.streamingUrl) {
      // forward the live iframe URL to the client
    }
    if (event.status === "COMPLETED") {
      // parse the JSON, stream to client, cache the result
    }
  }
}
```

- TinyFish doesn't send you one big response — it's a stream, like a drip-feed. Data arrives in random-sized chunks over time.
- The buffer is basically a holding area. Network can cut a message in half mid-sentence — buffer saves the incomplete piece, glues it back together when the next chunk arrives.
- Two event types come through: `streamingUrl` gives us the live browser iframe, `COMPLETED` gives us the actual bike data as JSON.
- Results hit the frontend as each shop finishes — you don't wait for the slowest one.

---

## 3:45 – 4:15 | The Output — "Structured data from unstructured websites"

**SHOW: Tab 1 or Tab 2 — results grid with bike cards**

- Every card came from a completely different website with a completely different layout.
- TinyFish returns JSON, but AI output is messy — so we normalize it.

**SHOW: `use-bike-search.ts` lines 44-68 — type normalization**

```typescript
const typeMap: Record<string, Bike['type']> = {
  scooter: 'scooter',
  automatic: 'scooter',
  'semi-automatic': 'semi-auto',
  enduro: 'adventure',
  // ... 20+ mappings → 4 canonical types
};
```

- TinyFish might return "automatic", "semi-automatic", "enduro" — we map all of them to 4 clean types.
- Same for prices — if a price is over 1,000, it's probably VND, so we divide by 25,000.

---

## 4:15 – 4:40 | Error Handling — "What if an agent fails?"

**SHOW: `route.ts` lines 254-259 — catch block**

```typescript
} catch (error) {
  console.error(`[TINYFISH] Failed: ${url}`, error);
  return false;
} finally {
  clearTimeout(timeoutId);
}
```

- Honestly — haven't seen an agent fully fail. Worst case it returns empty results.
- But the code handles it: each site runs independently. If one fails, the other 4 still complete.
- That's why I used `allSettled` instead of `Promise.all` — one site crashing doesn't take down the whole search.
- Supabase cache is also optional — if it's down, the app just skips caching and scrapes live.

---

## 4:40 – 5:00 | Wrap Up

**SHOW: Tab 1 — results should be loaded by now. Scroll through them.**

- So what you're looking at right now — 4 cities, 18 shops, all scraped in parallel, results streaming in as they finish.
- The entire backend is one API route — 389 lines, and half of that is just the cache layer.
- Built this in about 2 weeks using OpenCode and Nia for context.
- If there's one thing to take away from this — going from prompt to agent is way simpler than you think. The hard part isn't the code, it's picking the right problem.

---

## If someone asks...

**Rate limits?** — TinyFish has none. No concurrency caps either. Built for parallel automation at scale.

**Cost?** — API credits are unlimited for this demo. In production, cache aggressively — 6-hour TTL in Supabase.

**Accuracy?** — Surprisingly good. The natural language prompt handles most edge cases. Normalization layer cleans up the rest — type mapping, VND conversion, filtering out bikes with no name.

**Live demo still loading?** — Switch to Tab 2: "Let me show you the cached version — same data, pulled from our 6-hour cache."

---

## Numbers

| | |
|---|---|
| TinyFish integration | 11 lines |
| Parallel agents per city | 5-6 |
| Total shops | 18 across 4 cities |
| Typical search | 15-30 sec |
| Cached search | < 1 sec |
| Backend | 1 route, 389 lines |
| Cache TTL | 6 hours |
| Max live iframes | 5 per search |
