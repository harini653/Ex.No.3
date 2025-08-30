# Exno.3-Scenario-Based Report Development Utilizing Diverse Prompting Techniques

### DATE:30/08/2025                                                                         
### REGISTER NUMBER : 212222050017

## Aim

Design an AI‑powered customer‑support chatbot that handles (1) product troubleshooting, (2) order tracking, and (3) general inquiries. The bot must be fast, accurate, polite, and safe, with a conversational tone. This report shows **how to build it using multiple prompt patterns** and how to test, evaluate, and iterate.

---

## 1) Problem Context & Scope

**Support Channels:** web widget, mobile app, WhatsApp/Telegram.

**User Types:**

* **Guest:** has only order ID/email.
* **Signed‑in customer:** has profile, past orders.
* **Tier‑1 agent:** human fallback.

**Task Categories:**

1. **Troubleshooting:** device/app errors, connectivity, setup.
2. **Order Tracking:** status, delivery ETA, returns/exchanges.
3. **General Queries:** pricing, features, warranty, store hours, policies.

**Non‑Goals:** deep account changes (handled by secure portal), refunds without verification, medical/legal advice.

---

## 2) High‑Level Architecture (RAG + Tools)

```
User → Orchestrator → Router → {Retrieval QA, Tools, Policy Guardrails}
                               ↘ Logging/Analytics ↘ Eval Harness
```

* **System Prompt (global):** defines voice, boundaries, safety.
* **Router:** classifies intent → picks prompt template.
* **RAG (Retrieval‑Augmented Generation):** pulls snippets from KB/FAQs/manuals.
* **Tools/Functions:** `getOrderStatus`, `createTicket`, `scheduleCallback`, etc.
* **Guardrails:** PII masking, policy refusal templates, escalation flow.
* **Observability:** conversation store, red‑team traces, prompt version tags.

---

## 3) Core Data & Tools

**Entities:** order\_id, email, phone, product\_sku, serial\_no, issue\_code, address.

**APIs (function signatures):**

```json
{
  "getOrderStatus": {"order_id": "string"},
  "getReturnWindow": {"order_id": "string"},
  "getProductGuide": {"product_sku": "string"},
  "logIssue": {"order_id": "string", "issue_code": "string", "notes": "string"},
  "scheduleCallback": {"phone": "string", "timeslot": "ISO8601"}
}
```

**KB sources:** product manuals, troubleshooting trees, policies (return/warranty), shipping partner statuses.

---

## 4) Prompting Building Blocks

Prompts are split into **System**, **Developer**, and **User** layers.

### 4.1 Global System Prompt (Tone + Safety)

```
You are a customer‑support assistant. Be concise, friendly, and solution‑oriented.
Always verify identity before sharing sensitive info. Never invent facts; if unsure,
ask a brief clarifying question. Offer steps, not internal reasoning. Escalate when
account action is required. Use the company style: warm, direct, no jargon. Output
in the user’s language.
```

### 4.2 Developer Guardrails (Policies + Tools)

```
- If the request needs order data, collect order_id or email + last 4 digits of phone.
- Use functions for any data lookup. Do not guess. If the tool fails, inform the user and offer escalation.
- Redact PII in logs beyond last 4 digits.
- For prohibited content or unsafe requests, refuse with a short explanation and offer safe alternatives.
```

---

## 5) Prompt Patterns (with scenario examples)

Below we apply **diverse prompting techniques** to the three key tasks. For each pattern: purpose → template → example.

### 5.1 Straightforward (Direct Instruction) Prompting

**Purpose:** Fast, unambiguous actions when intent is already clear.

**Template:**

```
Task: <tracking | troubleshooting | info>
Goal: <exact outcome>
Constraints: <tone, length, don’ts>
Output form: <steps | short answer | JSON>
```

**Example — Order Tracking (User is signed in):**

```
Task: order tracking
Goal: Provide current status and ETA for order {order_id}. If delayed, give reason and next steps.
Constraints: friendly, ≤3 sentences + single CTA.
Output form: short answer.
```

**Expected Response (model):**
“Your order #48219 is in transit and due on 03 Sep, before 7 PM. If you won’t be home, I can request a doorstep drop or reschedule. Want me to arrange that?”

**Example — Troubleshooting:**

```
Task: troubleshoot product_sku={sku}, issue={issue_code}
Goal: Give 3 numbered steps. Stop after 3; then ask to confirm result.
Output form: steps.
```

### 5.2 Tabular‑Format Prompting

**Purpose:** Structured comparisons, status dashboards.

**Template:**

```
Present the answer as a table with columns: <C1, C2, C3>.
If data is missing, show “—”. No extra text below the table.
```

**Example — Shipment Milestones:**

```
Show a table: [Milestone | Timestamp | Notes] for order {order_id}.
```

**Example — Troubleshooting Options:**

```
List likely causes vs. quick checks vs. fix time as a 3‑column table.
```

### 5.3 Missing‑Word Prompting (Gap‑Filling / Slot Elicitation)

**Purpose:** Gently obtain missing required fields without overwhelming users.

**Template:**

```
We’re missing: <slot_names>. Ask for only one item at a time.
If user stalls, offer examples.
```

**Example — Order Lookup:**

```
I can check your status—could you share your order ID (e.g., 48‑219‑AX)?
```

If user shares email instead, bot adapts: “Thanks! I can use your email. For security, what’s the last 4 digits of your phone?”

### 5.4 Preceding‑Question Prompting (Mini‑Triage)

**Purpose:** Route to the right flow before answering.

**Template:**

```
Ask one brief question that determines the correct path: <Q>.
If answer=A → Flow 1; if B → Flow 2; else → fallback.
```

**Example — Returns vs. Exchange:**

```
Quick check: would you like a return or an exchange? (Return / Exchange)
```

### 5.5 Few‑Shot Prompting (Pattern Demonstration)

**Purpose:** Teach the model your format with 2–5 curated examples.

**Template:**

```
You are given examples of correct answers. Follow their format strictly.
[Example 1: <input> → <output>]
[Example 2: <input> → <output>]
Now answer the new case: <input>
```

**Example — Warranty Q\&A Format:**

* *Input:* “How long is the battery warranty?” → *Output:* “12 months || Coverage: manufacturing defects || Claim: via support portal.”
* *Input:* “Is accidental damage covered?” → *Output:* “No || Offer: protection plan add‑on.”

**New Case:**
“Can I transfer warranty if I gift the product?” →
*Expected:* “Yes || Transfer within 30 days || Need: proof of purchase + new owner email.”

### 5.6 Role‑Based (Persona) Prompting

**Purpose:** Align tone and expertise with scenario.

**Template:**

```
Act as a <role>. Keep replies <concise/friendly/empathetic>. Prioritize: <order>.
```

**Examples:**

* **Care Guide:** empathetic tone for broken items; offers apology + solution.
* **Logistics Agent:** precise, timestamped status updates.
* **Tech Specialist:** stepwise troubleshooting; asks targeted follow‑ups.

### 5.7 Tool‑Use / Function‑Calling Prompting

**Purpose:** Force the model to fetch or act via APIs.

**Developer Prompt Snippet:**

```
If the user asks about order status and order_id is present, call getOrderStatus.
If missing, ask for order_id. Never fabricate statuses.
Return tool results to the user in 2 sentences max.
```

**Example (model → tool):**

* Detects: “Where’s my package #48219?” → Calls `getOrderStatus({"order_id":"48219"})` → formats concise reply.

### 5.8 RAG Prompting (Grounded Answers)

**Purpose:** Cite from knowledge base to avoid hallucinations.

**Template:**

```
Use only these passages to answer. If not covered, say “I don’t have that info.”
Quote short phrases in quotes and link the article title.
```

**Example — Feature Clarification:**
User: “Does Model X support Bluetooth 5.3?”
Bot (grounded): “The spec sheet lists ‘Bluetooth 5.3’. Here’s the setup guide for pairing.”

### 5.9 Error‑Budget / Fallback Prompting

**Purpose:** Handle tool failures gracefully.

**Template:**

```
If a tool call errors, apologize once, offer two alternatives (retry or escalate),
and provide a safe next step.
```

**Example:**
“Sorry—our tracking service isn’t responding. I can try again now or create a support ticket with a callback. Which do you prefer?”

### 5.10 Safe‑Completion Prompting (Refusals & Boundaries)

**Purpose:** Keep replies safe, on‑policy, and helpful.

**Template:**

```
If the user requests disallowed actions (e.g., bypass verification), refuse briefly and offer compliant alternatives.
```

**Example:**
“I can’t share order details without verification, but I can email a one‑time link to confirm your identity. Shall I send it?”

### 5.11 Multilingual Prompting

**Purpose:** Mirror the user’s language.

**Template:**

```
Detect the user language and respond in it. Keep English terms where common in support.
```

---

## 6) Scenario Walkthroughs (End‑to‑End)

### A) Troubleshooting Flow (Bluetooth Pairing Fails)

1. **Preceding Question:** “Are you pairing with Android, iOS, or a laptop?”
2. **Direct Steps (Straightforward):** Provide 3 checks.
3. **RAG:** Pull device‑specific steps from KB.
4. **Missing‑Word:** Ask for serial\_no if warranty is needed.
5. **Outcome:** Confirm fix or escalate with `logIssue`.

**Prompt Template (Developer):**

```
For troubleshooting, give up to 3 steps and stop. Ask if the issue is resolved.
If not resolved after two cycles, escalate with a ticket.
```

### B) Order Tracking Flow (Delay Case)

1. **Direct Instruction:** Fetch status via tool.
2. **Tabular:** Show milestones as a table.
3. **Persona (Logistics):** Explain delay reason + options.
4. **Fallback:** Offer reschedule or pickup point.

### C) General Inquiry (Warranty Clause)

1. **Few‑Shot:** Enforce compact “X || Y || Z” format.
2. **RAG:** Cite policy passage.
3. **Safe Completion:** Avoid legal interpretations; provide link to policy.

---

## 7) Prompt Library (Ready‑to‑Use)

> Replace `{like_this}` with runtime values from the orchestrator.

**7.1 System (global)**

```
You are the Customer Care Assistant for {brand}. Be concise, kind, and factual.
Always verify identity for account data. Use tools for lookups; never invent values.
Give practical steps and one clear next action.
```

**7.2 Router (classification)**

```
Classify the user’s message into one of: troubleshooting, order_tracking, general_info, small_talk, billing, return_exchange, escalation.
Return JSON: {"intent":"…","slots":[…],"confidence":0-1}
```

**7.3 Troubleshooting Template**

```
Act as a Tech Specialist. Issue: {issue}. Product: {sku}.
Provide 3 numbered steps. After steps, ask: “Did this fix it?” If no, ask one targeted follow‑up only.
```

**7.4 Order Tracking Template**

```
Act as a Logistics Agent. If {order_id} is present, call getOrderStatus.
Reply with status + ETA in ≤2 sentences. Offer 1 next step (reschedule, pickup, or support ticket).
```

**7.5 General Info (RAG)**

```
Use only the provided passages to answer. If not present, say you don’t know and offer to escalate.
```

**7.6 Missing‑Word Elicitation**

```
We need {slot}. Ask for one item only, with an example format.
```

**7.7 Tabular Output**

```
Return a table with columns: {cols}. No extra prose.
```

**7.8 Safe Refusal**

```
When a request violates policy, apologize once, decline briefly, and offer a safe alternative.
```

---

## 8) Evaluation Plan

**Quality Metrics:** accuracy (grounding vs. KB/tools), completeness, brevity, tone, safety.

**Task‑Specific KPIs:**

* Troubleshooting: first‑contact resolution (FCR), average steps to fix.
* Order tracking: tool success rate, time‑to‑answer (TTA), deflection from human agents.
* General info: citation coverage, hallucination rate.

**Eval Harness:**

* **Golden Set:** 100 real tickets per intent with expected outputs.
* **Adversarial Set:** ambiguous orders, missing/incorrect IDs, tool outages.
* **Automated Checks:** JSON schema validity, link presence, PII redaction.

**Human Review Rubric (1–5):** clarity, empathy, actionability, policy compliance.

---

## 9) Safety, Privacy, and Compliance

* Verify identity before revealing order details.
* Redact PII in logs; store only necessary fields with rotation.
* Provide accessible alternatives (alt text, readable steps).
* Refuse unsafe or out‑of‑scope requests; suggest compliant paths.
* Maintain an audit trail of tool calls and decisions.

---

## 10) Deployment & Ops

* **Versioning:** tag prompts as `vX.Y`; log which version answered each turn.
* **A/B Testing:** compare template variants (e.g., 2 vs. 3 troubleshooting steps).
* **Caching:** cache static answers (store hours) with TTL.
* **Latency Budget:** tool calls < 900 ms; generation < 700 ms.
* **Monitoring:** alert on tool error rate > 3% or hallucination flags.

---

## 11) Sample Conversations (Condensed)

**Troubleshooting**

* User: “Headset won’t pair.”
* Bot: “Are you pairing with Android, iOS, or laptop?” → (user: Android)
* Bot: “1) Toggle Bluetooth. 2) Reset headset (hold 10s). 3) Re‑add device. Did this fix it?”

**Order Tracking**

* User: “Track #48219.”
* Bot: “It’s in transit, ETA Tue 03 Sep, by 7 PM. Want to reschedule or request doorstep drop?”

**General Info**

* User: “Is accidental damage covered?”
* Bot: “It isn’t covered under standard warranty. You can add the protection plan within 30 days of purchase.”

---

## 12) Checklists

**Authoring Checklist**

* [ ] Does the template ask only for necessary slots?
* [ ] Is output format specified (steps/table/JSON)?
* [ ] Are guardrails stated (no guessing; use tools)?

**Review Checklist**

* [ ] Factual vs. KB?
* [ ] Safe refusal when needed?
* [ ] Tone consistent with brand?

---

## 13) Appendix — Ready‑Copy Prompt Snippets

**A. Escalation**

```
It looks like we need a specialist. I can create a ticket with your best contact number and a preferred timeslot. Would you like me to do that now?
```

**B. Delay Apology**

```
Thanks for your patience—your parcel is delayed due to a hub backlog. I can request priority handling or set a pickup at a nearby point. Which works for you?
```

**C. Return vs. Exchange**

```
Quick check: are you aiming for a return or an exchange? I’ll guide
```




# Result: Thus the Prompts were exected succcessfully .
