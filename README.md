# n8n for developers

**From the basics to a real workflow: a daily tech-news digest**

---

## Part 1: The basics

### What n8n is

A workflow automation tool: you connect nodes on a canvas, and each run passes data from one step to the next.

- **Visual workflows:** Build flows by connecting nodes, and inspect every step's input and output.
- **Self-host or cloud:** Run it on your own server with Docker, or use n8n Cloud.
- **Fair-code license:** The source is available, and self-hosting is free for internal and personal use.

### Five core concepts

| Concept | What it is | In our digest |
| --- | --- | --- |
| Trigger | Starts a run | Schedule: every day at 08:00 |
| Node | One step: fetch, transform, branch or send | HTTP Request, RSS Read, Code, IF, Mailjet |
| Item | One JSON object flowing between nodes | One item per story |
| Expression | Inline JavaScript in any field | `{{ $json.subject }}` |
| Credential | A stored secret, reused by nodes | OpenRouter key, Mailjet login |

### Why developers like it

- **Any API:** The HTTP Request node calls any REST endpoint. Credentials live in n8n, not in the workflow.
- **Real code:** Drop into JavaScript or Python in a Code node when a visual node isn't enough.
- **Your infrastructure:** Self-host with Docker, so data and API keys stay on your own servers.
- **Built-in operations:** Retries, execution history and error workflows come out of the box.

---

## Part 2: Example — a daily tech-news digest

Five sources, smart ranking, a free LLM and an email at 08:00 every morning, built entirely in n8n.

### How the workflow runs

1. **Schedule** — every day at 08:00
2. **Fetch sources** — five feeds in parallel
3. **Merge** — wait for all five
4. **Filter & Rank** — score and keep top 20
5. **Any stories?** — stop if none
6. **LLM summarize** — pick 10, as JSON
7. **Format message** — build the email
8. **Send digest** — via Mailjet

Each source retries three times and continues on error, so one broken feed never stops the digest. Failures branch off to alerts.

### Five sources, two ways in

| Source | How it is fetched | Popularity signal |
| --- | --- | --- |
| Hacker News | HTTP Request to the Algolia API | Upvotes |
| Lobsters | HTTP Request to its JSON feed | Upvotes |
| TechCrunch | RSS Read | None |
| Tech.eu | RSS Read | None |
| The Register | RSS Read (Atom feed) | None |

Dropped along the way: Reddit and Ars Technica, which block requests coming from servers.

### Ranking: four signals, multiplied

```
score = relevance × freshness × popularity × buzz
```

- **Relevance:** Weighted keywords: Docker or CVE count 3, AI counts 1.
- **Freshness:** Halves every 12 hours, so new stories win.
- **Popularity:** Upvotes on a log scale: a boost, not a takeover.
- **Buzz:** ×1.5 for each extra source covering the same story.

### Ranking in practice

| Story | Relevance | Freshness | Popularity | Buzz | Score |
| --- | --- | --- | --- | --- | --- |
| Critical Docker vulnerability (3 sources, 2 h old, 300 points) | 5 | 0.90 | 1.8 | 2.25 | **18.2** |
| Rust 2.0 released (Hacker News, 20 h old, 800 points) | 2 | 0.31 | 1.9 | 1 | **1.2** |
| AI startup raises funding (TechCrunch, 3 h old) | 1 | 0.84 | 1 | 1 | **0.84** |

Illustrative numbers. Sorted by upvotes alone, the 800-point Rust story would have beaten the Docker vulnerability.

### The LLM picks, the code formats

- **A free model via OpenRouter:** `openrouter/free` picks a free model for each request.
- **Structured output only:** It returns story ids and one-line summaries as JSON.
- **Code builds the email:** Titles and links come from our own data. A bad reply falls back to a plain list.

```json
// The model returns only this:
[
  { "id": 3,
    "summary": "Patch now: actively exploited." },
  { "id": 7,
    "summary": "A faster linker for Rust builds." }
]
```

### When things go wrong

| Situation | What happens |
| --- | --- |
| Everything works | Digest email |
| One source fails, stories still found | Digest, plus a separate warning email |
| All sources work, nothing matches | Ends silently: a quiet day is not a failure |

Every source: three retries, continue on error, always output data.

---