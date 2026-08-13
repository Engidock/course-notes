# 🤖 Generative AI Project Mastery

> **Hey Fresher — Read This First!**
>
> Generative AI, in the practical sense you'll use on the job, means calling a large language model's API from your own code and building real product features around it — not chatting with a model in a browser tab. That shift matters: an API call needs error handling, cost control, structured output your code can parse, and guardrails against the model saying something wrong or unsafe to a paying customer.
>
> You've just joined **Threadloom**, a fashion e-commerce marketplace connecting independent Indian designers with buyers across the country. Threadloom's support team is buried — 40% of tickets are some version of "does this kurta run small?" or "will this fabric work for a summer wedding?" and agents spend most of their day re-answering near-identical questions by scrolling through product descriptions. Your manager's brief on day one: "We want an AI assistant that can answer product questions instantly, sound like us, and never make up a return policy that doesn't exist. Build it."

#### What You Will Learn and Build in This Project

You will build Threadloom's product-question assistant from a single hardcoded API call to a production-shaped service — adding structured output, retrieval-augmented generation over the real product catalog, function calling for order lookups, cost and latency controls, and basic evaluation before shipping. By the end you'll understand the concrete engineering decisions that separate a demo from something you'd trust in front of customers.

LLM API calls, system prompts, structured output with JSON schema, retrieval-augmented generation (RAG), embeddings and vector search, function calling / tool use, streaming responses, rate limiting and retries, caching, cost control, evaluation and guardrails

> **📦 Phase 1 — First API Call**
>
> Wire up the LLM API and get a working call-and-response for a single hardcoded product question.

> **📦 Phase 2 — Structured Output**
>
> Force the model to return machine-parseable JSON instead of free-form prose, so your app can actually use the answer.

> **📦 Phase 3 — Retrieval-Augmented Generation**
>
> Ground answers in Threadloom's real product catalog instead of letting the model guess or hallucinate.

> **📦 Phase 4 — Function Calling for Live Data**
>
> Let the assistant look up a real order status or return policy by calling Threadloom's internal APIs, not just recalling static text.

> **📦 Phase 5 — Reliability, Cost, and Caching**
>
> Add retries, rate-limit handling, response caching, and cost tracking before this touches real traffic.

> **📦 Phase 6 — Evaluation and Guardrails**
>
> Build a lightweight eval set and safety checks so you know the assistant is actually good before launch, not just "seems fine."

**Scene 1 — Threadloom, Jaipur | "We're drowning in the same five questions"**

> **Kavya** _Senior Backend Engineer, Customer Experience_
>
> Every day it's the same tickets: sizing, fabric care, whether a print will look different in person. Our agents know the answers cold, but they're typing the same paragraph fifty times a day instead of handling the tickets that actually need a human.

> **Arjun** _AI Platform Architect_
>
> The trap juniors fall into here is building a chatbot that sounds confident and is wrong half the time — worse than no chatbot at all, because customers trust it. We need this grounded in our actual catalog data and our actual policies, every single time.

> **You**
>
> So accuracy matters more than fluency here.

> **Arjun**
>
> Both matter, but if I have to pick one to get right first, yes — accuracy. Let's start simple and build up carefully.

### 1. Phase 1 — First API Call

**Business Problem:** Before building anything clever, you need the basic plumbing working: send a product question to an LLM, get a response back, print it. This proves the API key, network path, and SDK are all wired correctly before you add any real logic on top.

#### 1.1 The First Call

```python
# step1_first_call.py
import os
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=300,
    messages=[
        {
            "role": "user",
            "content": "A customer asks: 'Does the Rajasthani block-print cotton kurta "
                       "(product SKU TL-4471) run true to size or should I size up?'"
        }
    ]
)

print(response.content[0].text)
```

> **📖 What each part is doing**
>
> `Anthropic(api_key=...)` creates a client authenticated against your API key, read from an environment variable rather than hardcoded — never commit a key into source control. `model="claude-opus-4-5"` picks which model handles the request; larger models cost more per token but generally reason more reliably, which matters once you're grounding answers in real policy text. `max_tokens=300` caps the length of the response, which controls both cost and runaway generations. `response.content[0].text` pulls the actual text out of the response object — the API returns a list of content blocks (to support things like tool calls alongside text), so you index into it rather than treating the response as a plain string.

**Quiz: Right now this script has SKU `TL-4471` hardcoded in the prompt string. What's the immediate problem with shipping this as-is?**
- Nothing, this is exactly how it should work in production
- The model has no actual data about SKU TL-4471 — it will generate a plausible-sounding but potentially fabricated answer about sizing, since nothing in the prompt actually tells it what the product is
- The API call will fail because SKUs aren't allowed in prompts

> **Answer/explanation:** The second option is correct. The model has no access to Threadloom's product catalog — it only knows the model context you send in the prompt. Asking about "SKU TL-4471" without providing the product's actual size chart, fabric composition, or customer size-feedback data means the model can only generate a plausible-sounding guess based on general knowledge of kurtas, which is exactly the hallucination risk Arjun flagged in the opening scene. This gets fixed in Phase 3 with retrieval. Option 3 is false — there's nothing wrong with putting a SKU string in a prompt; it's just meaningless without grounding data behind it.

### 2. Phase 2 — Structured Output

**Business Problem:** Threadloom's support widget needs to render the assistant's answer as a formatted card — a short answer, a confidence flag, and a "needs human agent" boolean — not a wall of prose. Free-text responses can't reliably be parsed by the frontend.

#### 2.1 Forcing JSON with a Schema

```python
# step2_structured_output.py
import json
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

SYSTEM_PROMPT = """You are Threadloom's product-question assistant.
Always respond by calling the `answer_question` tool. Never respond in plain text."""

tools = [{
    "name": "answer_question",
    "description": "Return a structured answer to a customer's product question.",
    "input_schema": {
        "type": "object",
        "properties": {
            "answer": {"type": "string", "description": "A concise, direct answer, 2-3 sentences max."},
            "confidence": {"type": "string", "enum": ["high", "medium", "low"]},
            "needs_human_agent": {"type": "boolean"}
        },
        "required": ["answer", "confidence", "needs_human_agent"]
    }
}]

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=400,
    system=SYSTEM_PROMPT,
    tools=tools,
    tool_choice={"type": "tool", "name": "answer_question"},
    messages=[{"role": "user", "content": "Does the block-print kurta run true to size?"}]
)

tool_call = next(b for b in response.content if b.type == "tool_use")
structured = tool_call.input
print(json.dumps(structured, indent=2))
```

> **📖 Why tool calling instead of "please respond in JSON"**
>
> Asking a model in plain English to "respond in JSON" often works, but it's fragile — the model can wrap the JSON in markdown fences, add a conversational preamble, or produce syntactically invalid JSON under pressure. Defining a formal `tools` schema and forcing `tool_choice={"type": "tool", "name": "answer_question"}` makes structured output a first-class part of the API contract rather than a prompting convention — the model is constrained to fill in exactly the fields defined in `input_schema`, with the `enum` on `confidence` restricting it to one of three valid values instead of free text like "pretty sure" or "fairly confident," which your frontend code would then have to guess how to render. This is the difference between output your code can trust and output you have to defensively re-parse with regex.

**Prompting for a format vs. using structured tool output**

- **"Please respond in JSON" prompting** — quick to write, works most of the time in casual use, but has no enforcement — a determined edge case in the input can still produce malformed or off-schema output that breaks your parser.
- **Tool/function schema with forced `tool_choice`** — slightly more setup, but the API itself constrains the output shape, giving you a hard guarantee about which fields exist and what type/enum each one is. The right choice for anything feeding a UI or database.

### 3. Phase 3 — Retrieval-Augmented Generation

**Business Problem:** The assistant is still answering from general knowledge, not Threadloom's actual catalog. You need to ground every answer in real product data — size charts, fabric descriptions, and past verified customer Q&A — pulled from a vector database at query time.

#### 3.1 Embedding the Product Catalog

```python
# step3_index_catalog.py
import voyageai

vo = voyageai.Client(api_key=os.environ["VOYAGE_API_KEY"])

product_docs = [
    {
        "id": "TL-4471",
        "text": "Rajasthani block-print cotton kurta. Fabric: 100% cotton, pre-washed. "
                "Size chart: model is 5'6\" wearing size M. Customer feedback: "
                "'runs slightly large, size down if between sizes' (verified, 340 reviews)."
    },
    # ... one entry per SKU, generated from product metadata + verified review summaries
]

embeddings = vo.embed(
    [d["text"] for d in product_docs],
    model="voyage-3",
    input_type="document"
).embeddings

# Store embeddings + original text + metadata in a vector index (e.g. pgvector, Pinecone, Qdrant)
```

```python
# step3_retrieve_and_answer.py
def retrieve_context(question: str, k: int = 3):
    query_embedding = vo.embed([question], model="voyage-3", input_type="query").embeddings[0]
    results = vector_index.query(vector=query_embedding, top_k=k, include_metadata=True)
    return "\n\n".join(r.metadata["text"] for r in results.matches)

def answer_with_rag(question: str):
    context = retrieve_context(question)
    system_prompt = f"""You are Threadloom's product-question assistant.
Answer ONLY using the product information below. If the information isn't
present in the context, set needs_human_agent to true instead of guessing.

CONTEXT:
{context}"""
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=400,
        system=system_prompt,
        tools=tools,
        tool_choice={"type": "tool", "name": "answer_question"},
        messages=[{"role": "user", "content": question}]
    )
    return next(b for b in response.content if b.type == "tool_use").input
```

> **📖 Why retrieval fixes the hallucination problem from Phase 1**
>
> `vo.embed(...)` converts text into a numeric vector where semantically similar text ends up close together in vector space — this lets `vector_index.query()` find the product entries most relevant to a customer's question even if they don't share exact keywords ("runs small" vs. "true to size" vs. "size up"). `input_type="document"` vs. `input_type="query"` matters because Voyage's embedding models are trained asymmetrically — documents and queries get embedded slightly differently to optimize retrieval accuracy, and using the wrong type for each side measurably hurts search quality. The critical instruction is in the system prompt: *"Answer ONLY using the product information below... set needs_human_agent to true instead of guessing"* — this explicitly gives the model permission to say "I don't know" rather than confabulate an answer, which is the single highest-leverage sentence in the whole system for preventing hallucinated size or fabric claims.

> **Key takeaways**
> - RAG doesn't make a model smarter — it gives the model the specific facts it needs at query time, so it doesn't have to guess.
> - Explicitly instructing the model to defer when context is insufficient is what actually reduces hallucination, not just "adding more context."
> - Query and document embeddings can require different embedding modes/models — check your provider's docs rather than assuming symmetry.

### 4. Phase 4 — Function Calling for Live Data

**Business Problem:** Some questions aren't about the catalog at all — "where's my order" or "can I return this after 20 days." These need live data from Threadloom's order-management system and return-policy service, not static embedded text, since order status changes by the minute.

#### 4.1 Defining Tools for Live Lookups

```python
tools = [
    {
        "name": "answer_question",
        "description": "Return a structured answer to a customer's question.",
        "input_schema": { "...": "as defined in Phase 2" }
    },
    {
        "name": "get_order_status",
        "description": "Look up the current shipping status of a Threadloom order by order ID.",
        "input_schema": {
            "type": "object",
            "properties": {"order_id": {"type": "string"}},
            "required": ["order_id"]
        }
    },
    {
        "name": "check_return_eligibility",
        "description": "Check whether an order is still within Threadloom's return window.",
        "input_schema": {
            "type": "object",
            "properties": {"order_id": {"type": "string"}},
            "required": ["order_id"]
        }
    }
]

def handle_conversation(user_message: str, order_id: str | None):
    messages = [{"role": "user", "content": user_message}]
    response = client.messages.create(
        model="claude-opus-4-5", max_tokens=500, system=SYSTEM_PROMPT,
        tools=tools, messages=messages
    )

    while response.stop_reason == "tool_use":
        tool_use = next(b for b in response.content if b.type == "tool_use")
        if tool_use.name == "get_order_status":
            result = order_service.get_status(tool_use.input["order_id"])
        elif tool_use.name == "check_return_eligibility":
            result = return_service.check_eligibility(tool_use.input["order_id"])
        else:
            break

        messages.append({"role": "assistant", "content": response.content})
        messages.append({
            "role": "user",
            "content": [{"type": "tool_result", "tool_use_id": tool_use.id, "content": json.dumps(result)}]
        })
        response = client.messages.create(
            model="claude-opus-4-5", max_tokens=500, system=SYSTEM_PROMPT,
            tools=tools, messages=messages
        )

    return response
```

> **📖 How the tool-use loop actually works**
>
> The model doesn't call `order_service.get_status()` itself — it can't reach your infrastructure. Instead, when it decides a tool is needed, the API response comes back with `stop_reason == "tool_use"` and a `tool_use` block describing which tool and what arguments it wants. Your code is responsible for actually executing that call against Threadloom's real order service, then sending the result back to the model as a `tool_result` message so it can incorporate that live data into its final answer. The `while` loop handles the case where the model needs to chain calls (checking order status before deciding on return eligibility, for example) — it keeps calling back into the API until `stop_reason` is no longer `tool_use`, meaning the model has everything it needs to respond.

**Quiz: Why not just embed all order data into the vector index alongside the product catalog, the same way Phase 3 handled product info?**
- Order data can't be turned into embeddings
- Order status changes constantly (shipped, delivered, returned) and is customer-specific — re-embedding and re-indexing on every status change would be wasteful and still introduce staleness; a live function call fetches the current truth at query time
- Function calling is required by the API and static embedding is not allowed for this use case

> **Answer/explanation:** The second option is correct. RAG over embeddings is well suited to relatively stable, shared knowledge — a product's fabric composition doesn't change hour to hour. Order status is the opposite: it's highly dynamic, tied to one customer, and wrong the moment it goes stale. Function calling lets the model fetch the current, authoritative value directly from the source system at the moment it's needed, rather than trusting a snapshot that might be minutes or hours old. Option 1 is false (any text can technically be embedded); option 3 is false (there's no such API restriction — the two approaches solve different data-freshness problems and are often combined, as this phase does).

### 5. Phase 5 — Reliability, Cost, and Caching

**Business Problem:** Threadloom wants to launch this assistant to all customers, which means real production traffic: rate limits will get hit, the API will occasionally time out, and re-answering the exact same common question ("does this run small") thousands of times a day is pure wasted spend.

#### 5.1 Retries with Backoff

```python
import time
from anthropic import APIStatusError, APITimeoutError

def call_with_retry(request_fn, max_retries=4):
    for attempt in range(max_retries):
        try:
            return request_fn()
        except APIStatusError as e:
            if e.status_code == 429 and attempt < max_retries - 1:
                wait = (2 ** attempt) + 0.5
                time.sleep(wait)
                continue
            raise
        except APITimeoutError:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise
```

> **📖 Why exponential backoff, not a fixed retry delay**
>
> A `429` status means Threadloom is being rate limited — retrying immediately just gets rate limited again and adds load exactly when the API is asking you to back off. `wait = (2 ** attempt) + 0.5` doubles the wait time on each successive retry (0.5s, 2.5s, 4.5s, 8.5s), giving the upstream system room to recover and spreading retries out instead of hammering it in a tight loop. Distinguishing `APIStatusError` (rate limits, server errors) from `APITimeoutError` (network-level timeout) matters because a timeout might mean the request never even reached the model, while a 429 means it was rejected outright — both are retryable, but for different underlying reasons worth logging separately for debugging.

#### 5.2 Caching Repeat Questions

```python
import hashlib
import redis

r = redis.Redis(host="cache.threadloom.internal", decode_responses=True)

def answer_with_cache(question: str, sku: str):
    cache_key = "qa:" + hashlib.sha256(f"{sku}:{question.lower().strip()}".encode()).hexdigest()
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    result = answer_with_rag(question)
    r.setex(cache_key, 60 * 60 * 24, json.dumps(result))  # cache for 24 hours
    return result
```

> **📖 What makes a question "cacheable" here**
>
> Normalizing the question (`.lower().strip()`) before hashing means "Does it run small?" and "does it run small" hit the same cache entry instead of being treated as different questions purely due to casing or whitespace. Including `sku` in the cache key scopes the cache correctly — the same phrase asked about two different products must produce two different answers. `setex` sets a 24-hour expiry rather than caching forever, because product data, size-chart feedback, and even the model version behind the assistant can change, and a stale cached answer is worse than a cheap re-computed one. This single change is often the biggest cost lever available — sizing questions are highly repetitive across customers, so a meaningful fraction of traffic can be served from cache instead of hitting the API at all.

> **Key takeaways**
> - Retry only on genuinely retryable errors (429, timeouts, 5xx) with exponential backoff — retrying a 400 (bad request) just wastes calls on a request that will never succeed.
> - Cache keys need enough specificity (question + product ID here) to avoid serving a wrong answer, but enough normalization to actually get cache hits.
> - A time-bounded cache (`setex`, not permanent) balances cost savings against staleness risk.

### 6. Phase 6 — Evaluation and Guardrails

**Business Problem:** Before this ships to real customers, Arjun wants proof it's actually good — not a demo that looked fine on three test questions. You need a repeatable evaluation set and a basic safety check for the one failure mode that would actually damage trust: inventing a return policy that doesn't exist.

#### 6.1 A Minimal Eval Harness

```python
# eval/eval_set.py
eval_cases = [
    {
        "question": "Does the block-print kurta run true to size?",
        "sku": "TL-4471",
        "must_mention": ["size down", "runs large"],
        "must_not_mention": ["always true to size"]
    },
    {
        "question": "Can I return this if I don't like the color in person?",
        "sku": "TL-4471",
        "must_mention": ["7 day", "return"],
        "must_not_mention": ["no returns", "final sale"]
    },
    # ... 20-30 more real, curated cases covering sizing, fabric, returns, shipping
]

def run_eval():
    results = []
    for case in eval_cases:
        answer = answer_with_rag(case["question"])["answer"].lower()
        hits = all(phrase in answer for phrase in case["must_mention"])
        violations = any(phrase in answer for phrase in case["must_not_mention"])
        results.append({"case": case["question"], "pass": hits and not violations})
    pass_rate = sum(r["pass"] for r in results) / len(results)
    return pass_rate, results
```

> **📖 Why this counts as an evaluation, not just a demo**
>
> A demo is "I asked it three questions and it seemed right." An eval is a fixed, repeatable set of curated cases with explicit pass/fail criteria that you can re-run after every prompt change, model upgrade, or catalog update — so you can tell objectively whether a change made things better or worse instead of relying on vibes. `must_mention` and `must_not_mention` are a simple but effective check here: they don't require exact string matching of the whole answer, just confirmation that specific facts are present (the real return window) and specific dangerous claims are absent (a fabricated policy). A real production eval set for Threadloom would grow to hundreds of cases and include human-graded quality scores alongside these mechanical checks, but even 20-30 cases catch regressions a single manual test would miss.

#### 6.2 A Guardrail Against Policy Hallucination

```python
FORBIDDEN_POLICY_CLAIMS_SYSTEM_SUFFIX = """
Return and refund policy facts you may state:
- Standard return window: 7 days from delivery, unworn with tags.
- Sale items: exchange only, no cash refund.
- Custom/made-to-order items: non-returnable.

Do not state any return, refund, or warranty policy not listed above.
If asked about something not listed, set needs_human_agent to true.
"""
```

> **📖 Why this is a guardrail, not just more context**
>
> This isn't retrieved dynamically like the product catalog in Phase 3 — it's a small, hand-verified, hardcoded policy block appended directly into the system prompt for every request, specifically because Threadloom's legal and support leads reviewed and signed off on this exact wording. Policy claims carry outsized risk (a customer who's told "yes, we accept returns after 30 days" and then denied one is a much worse outcome than a missed sizing tip), so this block trades RAG's flexibility for the certainty of a fixed, reviewed source of truth, with an explicit instruction to defer to a human for anything outside it.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Add a `list_similar_products` tool that the assistant can call when a customer asks "what else looks like this but cheaper," backed by a vector similarity search over the product catalog embeddings from Phase 3.
2. Add streaming to the Phase 1 call using the SDK's streaming interface so the support widget can render tokens as they arrive instead of waiting for the full response.
3. Extend the eval harness from Phase 6 to also measure average response latency and average token cost per case, and set a budget threshold that fails the eval if median cost per answer exceeds a target you define.
4. Add a lightweight moderation pre-check that runs before the main assistant call and flags/blocks obviously abusive or off-topic input before it reaches the model.
5. Build an A/B comparison script that runs the same 20 eval questions through two different system prompts (or two different models) and reports which one has a higher pass rate and lower average cost.

**Quiz: Threadloom's assistant correctly answers 95% of eval questions on the first try, but Arjun still wants a human-in-the-loop review before full launch. Why might that be reasonable even with a high eval pass rate?**
- A 95% pass rate is actually a failing grade for any software system
- A curated eval set, however good, can't cover every real customer phrasing and edge case; a staged rollout with human review catches failure modes the eval didn't anticipate before they reach all customers
- Eval harnesses like this one are not a real engineering practice

> **Answer/explanation:** The second option is correct. An eval set is only as good as the cases it contains — it's a sample of anticipated questions and phrasings, not an exhaustive guarantee of correctness against every way a real customer might ask something. A staged rollout (a percentage of traffic, or human review of flagged low-confidence answers) is a standard risk-management practice that catches the failure modes nobody thought to write a test case for, especially in a customer-facing context where a wrong answer has real cost. A 95% pass rate isn't "failing" in any general sense (option 1 is false — it depends entirely on what's being measured and the acceptable risk for the use case), and eval harnesses of exactly this shape are standard practice in real LLM application engineering (option 3 is false).

### Generative AI Project Complete 🎉

You took Threadloom's product-question assistant from a single hardcoded API call to something shaped like a real production system: structured output the frontend can render reliably, retrieval-augmented generation grounded in the real product catalog, function calling for live order and return-eligibility lookups, retries and caching for reliability and cost, and an evaluation harness plus policy guardrails before anything reached real customers.

> **Kavya** _Senior Backend Engineer, Customer Experience_
>
> Sizing questions dropped off our support queue by almost a third in the first two weeks. That's not the assistant being clever — that's it being grounded and honest about what it doesn't know.

> **Arjun** _AI Platform Architect_
>
> The guardrail block in Phase 6 is the one piece I'd never let anyone skip. Everything else can be "pretty good." Policy claims have to be exactly right, every time.

> **You**
>
> I went in thinking this was mostly about prompt-writing. Turned out to be mostly about deciding what the model should never be allowed to guess.

> **Next: Prompt Engineering Project Mastery**
>
> - Go deeper into how prompt structure, examples, and reasoning instructions change output quality
> - Learn a systematic process for testing prompt variants instead of guessing what "sounds better"
> - Build the eval discipline from Phase 6 here into a full versioned prompt-testing workflow
