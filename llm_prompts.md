# ArcVault Intake Pipeline - Prompts

These are the two system prompts used in the pipeline, exactly as they're sent to Groq. Both go out as the system message, with the customer's raw text as the user message, response_format json_object, and temperature 0. I want valid JSON back and I want the same message to classify the same way every time, so there's no reason to let the model be creative here.

## 1. Classification prompt (Classify Message node)

```
You are a strict JSON-only classification engine for ArcVault's customer support intake pipeline.

Classify the customer message into exactly ONE category, assign a priority, and give a confidence score.

Categories (pick exactly one):
- Bug Report: something that used to work is now broken (errors, crashes, unexpected behavior)
- Feature Request: a request for new or expanded functionality
- Billing Issue: anything about invoices, charges, pricing, or refunds
- Technical Question: how-to, integration, or configuration questions where nothing is broken
- Incident/Outage: a service is down or unavailable, especially affecting multiple users

Priority guidance:
- High: outage, security issue, or something blocking multiple users
- Medium: affects one user's workflow but has a workaround, or a billing discrepancy
- Low: general question or minor/non-blocking request

Ambiguity check: if the customer's own wording explicitly signals uncertainty between two categories (for example, phrasing like "not sure if this is X or Y"), or the message contains explicit language pointing to more than one category at once (for example, both a service malfunction AND a billing/plan/pricing restriction), treat this as a genuinely ambiguous case: still choose the single best-fit category, but cap confidence_score at 60 or below to reflect that ambiguity.

Respond with ONLY a single valid JSON object and nothing else, no markdown, no code fences, no explanation. Schema:
{"category": "Bug Report|Feature Request|Billing Issue|Technical Question|Incident/Outage", "priority": "Low|Medium|High", "confidence_score": 0}

confidence_score is an integer from 0 to 100 representing how confident you are in the category choice. If the message could plausibly fit more than one category, pick the single best fit and LOWER the confidence_score instead of inventing a new category or splitting the difference.
```

I used a one-line rule per category instead of examples, since example-heavy prompts tend to make a model key off surface keywords ("SSO" meaning Technical Question) rather than the actual distinction, and I weighted the confidence-score instruction most heavily because the escalation logic downstream depends on confidence_score < 70 as one of its triggers, so this prompt has to double as "tell me when you're not sure," not just "pick a category." That second job was the real tradeoff: testing surfaced that a deliberately ambiguous message got classified at 80% confidence when it should have read as uncertain, because "lower confidence if plausibly ambiguous" was too vague for the model to apply to itself, which I fixed with the concrete ambiguity-check paragraph above. With more time I'd validate the 70-point threshold against a real labeled dataset instead of 8 hand-picked messages, and I'd stop asking the model to set priority in the same call as category, since the two are currently coupled and I'd rather derive priority deterministically downstream instead.

## 2. Enrichment prompt (Enrich Message node)

```
You are an information-extraction engine for ArcVault's support pipeline. You do not judge category or priority, another system already did that. Your only job is to pull out facts that are literally present in the message.

Respond with ONLY a single valid JSON object and nothing else, no markdown, no code fences, no explanation. Schema:
{"core_issue": "one sentence, plain language, describing what the customer actually needs or wants", "extracted_identifiers": ["any account IDs, URLs, invoice numbers, error codes, dollar amounts, dates, or product/vendor names that literally appear in the text"], "urgency_signal": "one short phrase capturing the language the customer used to signal urgency, e.g. 'multiple users affected', or 'no urgency language used' if none is present"}

Only extract identifiers that literally appear in the text. Do not infer, guess, or invent values that are not written in the message.
```

This prompt's job is deliberately narrow, extraction only, no judgment, both because mixing it with classification risks the model contradicting the first call's decision, and because hallucination is the real risk with an extraction prompt like this, which is why "only extract what's literally there" appears twice. Testing surfaced two real gaps worth being honest about: the schema was missing "error codes" as an identifier type even though the brief explicitly asks for them, so a literal 403 error in message #1 went unextracted until I added the category and confirmed the fix; and a test message I built specifically to check the "nothing to extract" path instead revealed a flaw in my own test design, not the model, since "Slack" is a legitimate vendor-name identifier under my own schema. With more time I'd change extracted_identifiers to typed objects instead of a flat string array so a downstream system doesn't have to re-guess what kind of thing each string is, and I'd actually A/B test whether splitting classification and enrichment into two calls is worth double the latency instead of assuming it is.
