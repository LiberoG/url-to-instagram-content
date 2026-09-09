# URL to Instagram Content

An AI automation built with **n8n** and the **Claude API**: paste a website URL, get back 5 ready-to-post Instagram captions written in that brand's actual voice — built as a hands-on exercise in Anthropic's [AI Fluency course](https://academy.claude.com/courses/ai-fluency-framework-foundations), specifically the **Description–Discernment loop**.

## What it does

1. You submit a website URL through a simple form.
2. The workflow fetches the page and extracts the readable text.
3. Claude (Sonnet 5) analyzes the text and summarizes the brand's voice — tone, key themes, audience, style.
4. Claude (Opus 5) uses that voice profile to write 5 Instagram posts, each with a caption and hashtags.
5. The results land in your inbox.

## The story behind it

This wasn't built in one pass — it's a real example of the **describe → run → discern → refine** loop the AI Fluency course teaches, including the parts that broke along the way.

### Round 1: the naive description

I started with the simplest version of what I wanted:

> *"I want an n8n flow that takes a website URL, figures out the brand's voice, and emails me 5 Instagram-style posts based on it."*

That's it — no format spec, no tone constraints. First result: a genuine bug, not a design gap. The brand-voice step returned `undefined` further down the pipeline, and rather than making something up, Claude flagged it honestly:

> *"Quick heads-up: the brand voice profile came through as undefined — so I don't have your tone, audience, product, or any voice rules to work from. If you paste it in, I'll rewrite these to match exactly."*

That turned out to be the right behavior surfacing a real problem — not the AI's fault, but a pipeline bug worth tracking down.

### The debugging

A few real issues came up building this, each with an actual root cause:

- **Gmail SMTP + App Passwords hit a dead end.** My Google account uses security-key-only 2FA, which Google doesn't allow App Passwords for. Rather than weaken my account's security to work around it, I swapped the email step entirely to the **Resend API** — no OAuth dance, no app-password maze.
- **A silent parsing bug from adaptive thinking.** Claude Opus 5 and Sonnet 5 return a `thinking` content block *before* the actual answer block by default. My code assumed `content[0]` was always the answer — so it was quietly grabbing the thinking block instead. Fixed by searching the response for the block where `type === "text"` instead of assuming a fixed index.
- **403 Forbidden fetching the target site.** Some sites block plain server-side requests outright. Fixed by sending a real browser User-Agent header.

None of this showed up as a clean "it just worked" build — and that's the point. The debugging is the actual work.

### Round 2: the refined description

Once the pipeline was solid, the real Discernment step started: reading the *actual* output critically, not just checking for errors.

**What Round 1 got right:** the brand voice extraction was genuinely specific — real product details, real neighborhood references, a consistent tone across all 5 posts. Not generic filler.

**What it got wrong:** all 5 posts followed the identical structure — bold hook, two paragraphs, CTA, hashtags. Read back to back, the repetition was obvious even though each post individually looked fine.

The Round 2 description explicitly asked for variation in structure and length across the 5 posts, and light, sparing emoji use — targeting the exact gap Round 1 revealed, without touching what was already working.

### The takeaway

The interesting part of this project isn't the workflow — it's that every fix came from actually reading the output and the errors, not from assuming either the AI or the automation "just works." Description sets the direction; Discernment is what actually catches whether you got there.

## Tech stack

- **n8n** (self-hosted via Docker) — orchestration
- **Claude API** — `claude-sonnet-5` for brand voice analysis, `claude-opus-5` for content generation
- **Resend** — email delivery

## How to use it

1. Download [`url-to-instagram-content.json`](./url-to-instagram-content.json) from this repo.
2. In n8n: **Workflows → Import from File**, select the downloaded JSON.
3. Create two credentials:
   - **Header Auth** named `Anthropic API Key` — Name: `x-api-key`, Value: your Anthropic API key
   - **Header Auth** named `Resend API` — Name: `Authorization`, Value: `Bearer <your Resend API key>`
4. Attach the Anthropic credential to the **Analyze Brand Voice** and **Generate Posts** nodes, and the Resend credential to the **Email Me the Posts** node.
5. In the **Email Me the Posts** node, replace the placeholder `to` address with your own email (Resend's free tier only delivers to the address you signed up with, unless you verify a domain).
6. Run it and submit a website URL through the form.

## Possible next steps

- Support for platform-specific formats beyond Instagram (LinkedIn, TikTok captions)
- A proper hosted front-end instead of the n8n test form
- Public Instagram/LinkedIn profiles as input, via each platform's official Graph/API path (direct scraping isn't reliable or ToS-compliant)
