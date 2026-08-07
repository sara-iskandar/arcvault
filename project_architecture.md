ArcVault Intake and Triage Pipeline. Architecture Write-Up.

## System Design

The workflow can start in two ways. One is a Manual Trigger, which loads a fixed set of 8 test messages I built myself. The other is a real Webhook Trigger, which starts the workflow automatically the moment someone sends it a message over the internet. Both ways lead into the same chain of steps after that: a step called Ingested Message, then Classify Message (asks Groq's AI a question), then Enrich Message (a second, separate question to Groq), then Route and Escalate (plain code, no AI involved), then Build Final Record, then Review Results.

The brief asks for the workflow to start automatically when a new message arrives, not when someone clicks a button. That is why the real Webhook Trigger exists. It listens at a specific web address (POST /arcvault-intake). The moment a message is sent to that address, the whole workflow runs, and a setting called responseMode: lastNode means whoever sent the message gets the finished, fully processed result sent straight back to them. Both the Manual Trigger and the Webhook Trigger feed into one shared step called Ingested Message. That step's only job is making sure the message looks the same shape no matter which way it came in, since a real webhook message arrives in a different shape than my fixed test data. I tested this for real: published the workflow and sent it a live request using curl (a command line tool for sending web requests), and got back a correct, fully processed result.

I kept the Manual Trigger and fixed test data even after adding the real Webhook Trigger, because running the same 8 messages every time is the only way to tell if a change actually helped or made things worse, without waiting for real traffic to show up. 8 test messages run through it: the 5 given in the brief, plus 3 I added myself, each one built to test a specific weak spot (classification confusion, a message with nothing to extract, and the security escalation rule).

Nothing in this workflow is saved anywhere outside of it. There is no database and no queue. Each step reads directly from an earlier step by name, using a built-in n8n feature that lets one step see another step's data, instead of every step manually passing everything forward. This keeps each step's job small and clear. The downside is that if something crashes partway through, that message's progress is lost completely. Fine for a demo, not something you would want in a real, high-volume system. More on that below.

Classifying and enriching a message are two separate calls to Groq, not one combined request. I split them so each request has exactly one job and stays simple to reason about. The real cost is that two calls take twice as long and cost twice as much per message as one call would, and I have not actually proven that split is worth it. That is something worth testing properly with more time.

## Model Choice

I used Groq's llama-3.3-70b-versatile model for both calls. Groq's free tier meant I could test as much as I wanted at no cost, and it is fast, which mattered a lot since every small prompt change meant running the whole thing again to check it. Groq's API is also built the same way as OpenAI's, so setting it up in n8n's HTTP Request node was simple and standard. One honest gap: this model is probably bigger and more capable than most of these 8 messages actually need. A smaller, cheaper model would likely handle something like "invoice shows the wrong amount" just fine. But I did not have a real set of labeled test data (messages with the correct answers already marked) to safely check whether a cheaper model would still get the harder, more confusing messages right. So I used one model everywhere instead of guessing.

## Routing Logic

Deciding which team gets a message is handled by plain code (the Route and Escalate step), not by asking the AI. Here is the fixed table it uses:

| Category | Queue | Reasoning |
|---|---|---|
| Bug Report | Engineering | Something is broken, engineers fix broken things |
| Incident/Outage | Engineering | Same team, outages are usually acute bugs |
| Billing Issue | Billing | Invoice and pricing questions belong to billing |
| Feature Request | Product | Goes to the product backlog, not an active fire |
| Technical Question | IT/Security | How-to and login/auth questions fit here (e.g. SSO) |
| unrecognized category | Escalation | An unknown category is itself a warning sign, default to a human |

I used code instead of another AI call here because this decision is a fixed business rule, not something that needs judgment. If someone asks why a message went to Engineering, I can point to the exact line of code that decided it, instead of trying to explain an AI's reasoning after the fact.

## Escalation Logic

A message gets flagged for a human to review, which overrides the normal routing above, if any of these four things are true. The AI's confidence score is below 70. The message contains the exact words "outage" or "down for all users". It is a Billing Issue and mentions a dollar amount over $500. Or the message contains a security related word like "breach", "unauthorized access", or "data leak". That last rule was not one of the examples given in the brief. I added it myself, because ArcVault's whole product is audit logs, and its customers clearly care a lot about security and compliance, so a security incident deserves its own check instead of hoping one of the other three rules happens to catch it. I built a specific test message (#8) just to prove this rule works on its own, and confirmed that it does. All four checks are plain code, for the same reason as the routing table: a decision like this needs a clear, checkable reason.

Worth admitting: the first rule (confidence below 70) did not catch what I expected the first time I tested it. I wrote a message on purpose to be confusing. It hints at both a bug and a billing problem, and the customer literally says "not sure if...". I built it specifically to see if it would get flagged. It did not. The AI came back 80% confident anyway, even though the message was genuinely unclear. The code did exactly what it was told to do. The real problem was one step earlier, in the instructions I gave the AI for classifying messages, which told it to lower its confidence on unclear messages but never explained what "unclear" actually looks like. I fixed the instructions, not the 70% threshold. Good lesson from this: "the rule did not trigger" and "the rule was given bad information to work with" are two different problems, and it is easy to fix the wrong one if you do not stop to figure out which one you actually have.

## What I Would Do Differently at Production Scale

Handling failures is the biggest gap right now. If a call to Groq fails, or comes back broken, that message just stops being processed and nothing else happens. At real scale, I would want failed messages to automatically go to the Escalation queue instead of disappearing, I would want the workflow to automatically try again if a call fails for a temporary reason, and I would want a way to make sure the same incoming message never accidentally gets processed twice.

The AI is less consistent than I expected. I ran the same 8 messages through twice, using the exact setting meant to make results repeatable (temperature 0, which is supposed to reduce randomness). The results were not identical both times. One message's priority changed from Medium to High between runs. Another message's confidence score changed by 5 points. Nothing about the actual routing decisions changed because of this, but it is a real thing to know: you cannot assume a past decision could be exactly recreated later, even with the same message and the same instructions. A real system would need to save the actual decision made at the time, not assume it could be regenerated identically for review later.

Speed could improve almost for free. The Enrich Message step does not actually need anything from the Classify Message step, only the routing step later does. Right now they run one after another. Running them at the same time instead would cut the time each message takes roughly in half.

Cost grows in a straight line with how many messages come in, and every single message currently uses the same large, more expensive AI model, whether it needs to or not. I would add a cheaper, faster first check to handle the obviously simple messages, and only use the full process for messages that actually need it.

There is currently no record keeping at all. Nothing gets logged anywhere. Without that, there is no way to notice if the system is slowly getting worse over time, and no responsible way to adjust the 70% confidence threshold or the escalation keyword list based on real evidence instead of a guess.

## Phase 2, Given Another Week

- A webhook already exists for real ingestion, but I would also add a proper form or email option.
- Replace the current "just show me the result" output step with somewhere records actually need to go, like Google Sheets or a real support ticket system.
- Instead of just asking the AI nicely for JSON and hoping it is formatted correctly, use a stricter method that forces the AI's answer into the exact shape needed, so a broken answer becomes an obvious error instead of quietly crashing something downstream.
- Build a simple review screen for escalated messages, and actually record it whenever a human overrides what the AI decided. That correction data is the real path to eventually having proper labeled test data, to tune thresholds against real evidence instead of guesses.
- A basic dashboard: how often messages get escalated over time, how confident the AI usually is by category, and how many messages come in overall. These are the kinds of signals that would show something going wrong before a customer notices their message went nowhere.
