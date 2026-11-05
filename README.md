# AtriBot

AtriBot is an AI assistant for service businesses: salons, dental clinics, repair shops and the like. It sits on the business's website, answers customer questions from the business's own material, and lets the owner know when someone is ready to buy or book.

Live at [atribotai.com](https://atribotai.com). The agency site [atrigen.rs](https://atrigen.rs) runs it in production.

The code is private. This repository describes what the product does and how it is built.

## What it does

- The business adds one script tag to its site and gets a chat window in its own colors, with its own avatar and greeting.
- The assistant answers only from what the business has given it: service descriptions, price lists, text, or pages imported from its website. If the answer is not there, it says so instead of guessing.
- The conversation carries over as the visitor moves from page to page.
- When a visitor leaves contact details or shows clear intent, the owner gets an email with the whole conversation.
- The owner has a dashboard with the conversations, the knowledge base, and the look and wording of the chat window.
- Answers are in Serbian, the market it was built for.

The website is the main channel today. WhatsApp and Instagram are planned on the same knowledge base and the same conversation logic.

## How answers are found

The knowledge base is split into sections and indexed in two ways: by meaning, with vector search, and by exact words, with keyword search. The two result lists are merged, a reranking model puts the candidates in order, and only the best few go into the prompt. If even the best match is weak, the assistant does not answer from it. This is what keeps it from inventing prices or opening hours.

The prompt keeps the business's material separate from the instructions, and the material is treated as data, never as commands. Filters on the way in and on the way out catch attempts to pull out the instructions or steer the assistant off topic.

Each request goes to a primary language model, with a second provider as a fallback. If the first one is down or too slow, the second one answers, so an outage at one provider does not take the assistant offline.

## Architecture

```
Business website (widget)
        |
        v
API Worker (Hono) ---> D1, KV, R2, Vectorize, Workers AI
        |
        +---> language models (primary and fallback)
        +---> email

Business owner ---> Dashboard Worker (Astro) ---> same database, and the API for files and the knowledge base
```

- The API is a Cloudflare Worker. D1 holds the data, KV handles caching and rate limits, R2 stores files, and Vectorize holds the knowledge base, with a separate index for each business. Keyword search has its own database.
- The widget is plain TypeScript inside a closed Shadow DOM, with no framework, and it stays under 25 KB compressed. The host page's styles cannot break it, and it cannot break the host page.
- The dashboard is Astro, rendered on a Worker. Owners sign in with a link sent by email, with Google, or with a passkey.
- New data stores are created in the EU.
- The main database is exported every day, encrypted, and kept in storage that cannot be overwritten for a fixed period, with a second copy at another provider. Restoring from a backup has been tested.

## How it is built and shipped

- Every change is a pull request. CI runs type-aware linting, type checks, tests, a dependency audit and a secret scan over the whole history before anything can be merged.
- Tests run inside the real Workers runtime, against a local database with every migration applied. Each route has tests for who may call it and who gets refused.
- Every merge deploys to a staging environment with its own database.
- A production release is uploaded next to the running version without traffic, smoke-tested on its own, and only then switched on. If the check fails, the previous version keeps serving.
- Database changes are made in two steps, adding first and removing later, so the version that is running keeps working while the next one goes out.
- Decisions that would otherwise look strange are written down as short records in the repository, along with a glossary of the terms the code uses.

## Stack

TypeScript, Cloudflare Workers, D1, KV, R2, Vectorize, Workers AI, Hono, Astro, Vitest, Resend.
