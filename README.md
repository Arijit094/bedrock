# bedrock


⚠️ The one cost trap that changes everything
The diagram's "Bedrock Knowledge Bases → vector store" step, if you use the default OpenSearch Serverless vector store, has a minimum of 2 OCUs at $0.24/OCU‑hr ≈ $345/month even with zero queries. ⁠ For a jobless candidate, that's a dealbreaker.

Fix: use the Amazon Bedrock Managed Knowledge Base instead. It's pure pay‑per‑use with no minimum:

Index storage: $5 / GB / month (your 500 docs ≈ a few GB → a few $/month)
Retrieval: $1 / 1,000 retrieve calls
Document parsing, embedding model, and re‑ranking are included free ⁠
That single swap takes your vector‑store line from ~$345/mo → under ~$10/mo. This is the #1 thing to mention in your README/interview — it shows you understand cost architecture, a real DevOps skill.

The full stack (what you're actually building)
Layer	AWS service	Role
Ingestion	Amazon S3	Store the 500 uploaded PDFs/DOCX
Indexing	Bedrock Managed Knowledge Base	Chunk + embed + store vectors (no minimum)
Query	Amazon API Gateway + Lambda	Expose an HTTP endpoint; calls RetrieveAndGenerate
Generation	Bedrock (Claude Haiku)	Context‑aware answers
IaC	Terraform	Everything above, versioned
CI/CD	GitHub Actions (OIDC, no long‑lived keys)	terraform plan on PR → apply on merge
Monitoring	CloudWatch (logs, metrics, alarms, dashboard)	Observability
Docs	GitHub README + architecture diagram	The "LinkedIn demo"

Model choice: use Claude Haiku (the cheapest capable model) — ~$1/M input, $5/M output tokens. ⁠ For a 500‑doc internal assistant at personal scale, LLM cost is cents per month.

Phase‑by‑phase build (follow in order)
Phase 0 — Account & safety rails (Day 1)

New AWS account. Set up a billing alarm (Budgets, e.g. alert at $5) so you never get surprised.
Use region ap‑south‑1 (Mumbai) — you're in India, lower latency, and Bedrock/Haiku are available there.
Install awscli, terraform, git, python3.
Phase 1 — Data (Day 1–2)
4. Create an S3 bucket, upload your 500 documents. (For the demo you can use public sample docs — HR policy, deployment runbooks, incident SOPs — if you don't have real ones.)

Phase 2 — Knowledge Base (Day 2)
5. Create the Managed Knowledge Base pointing at that S3 bucket (via Console once, to learn it; you'll automate it in Phase 4). Note the Knowledge Base ID and the model ARN (Haiku). ⁠

Phase 3 — Query app (Day 2–3)
6. Write one Lambda (Python) that calls bedrock_agent_runtime.retrieve_and_generate() with your KB ID + Haiku ARN, returns the answer + source citations.
7. Put API Gateway (HTTP API) in front of it → you now have a working POST /ask.
8. Test with curl. Ask: "What is our leave policy?" — if you get a grounded answer, the core works. 🎉

Phase 4 — DevOps automation (this is your star, Day 3–5)
9. Terraform: declare S3 bucket, IAM roles, Lambda, API Gateway, CloudWatch, and the Knowledge Base. terraform apply should reproduce everything from Phase 2–3.
10. GitHub Actions CI/CD:
- Use OIDC (GitHub → AWS) so no AWS keys live in GitHub — a strong, interview‑worthy detail.
- On PR → terraform validate + terraform plan (post the plan as a comment).
- On merge to main → terraform apply. ⁠
11. CloudWatch: enable Lambda + API GW logs, a dashboard (invocations, 4xx/5xx, latency, error rate), and alarms (e.g. 5xx > 0, or latency threshold) → notify via SNS (email).

Phase 5 — Documentation & demo (Day 5–6)
12. README: problem, architecture diagram (your image, polished), setup steps, cost breakdown, and a "how it works" section.
13. Record a 2–3 min screen capture: push a commit → watch CI run plan/apply → open the endpoint → ask a question → show the answer + CloudWatch dashboard. This video is your LinkedIn demo.
14. Post on LinkedIn: short caption + the video + link to the public GitHub repo.

Realistic monthly cost (personal scale)
Item	Est. cost/mo
S3 (500 docs, ~2–5 GB)	~$0.10
Managed KB storage ($5/GB)	~$10–25
Retrieve calls ($1/1k)	~$0.01–1
Lambda (1M req/mo free)	$0 (free tier)
API Gateway (1M req/mo free)	$0 (free tier)
Claude Haiku tokens	~$0.50–5
Total	~$10–35/month

You can also run it for ~$0 during the build using the $200 free‑tier credits new AWS accounts get (valid 6 months).

How to frame it for the 3‑year gap (LinkedIn + interviews)
Don't say "I built a RAG chatbot." Say:

"I built an enterprise document assistant on AWS and — more importantly — the DevOps platform around it: Terraform IaC, OIDC‑based GitHub Actions CI/CD with plan‑on‑PR/apply‑on‑merge, CloudWatch monitoring with alarms, and a cost‑optimization decision that cut vector‑store spend from ~$345/mo to ~$10/mo by choosing the Managed Knowledge Base over a provisioned store."

That sentence proves current DevOps skills + cloud‑cost thinking, which is exactly what closes a gap. The AI is just the payload.



1. Can you run this on a brand-new free-tier account? Yes — with one catch.
A new account (created after 15 July 2025) gets $100 in credits at sign-up + up to $100 more by completing 5 activities (one of which is literally "try a prompt in the Bedrock playground"). ⁠ That's up to $200, valid 6 months on the Free plan. ⁠

What's always free (resets monthly, never expires) vs what eats your credits:

Service	Free?
Lambda (1M req/mo)	✅ Always free — your query app costs $0
API Gateway (1M calls/mo)	✅ Always free
S3 (5 GB)	✅ Always free — holds your 500 docs
CloudWatch (basic logs/metrics)	✅ Mostly free
Bedrock tokens (the LLM)	❌ Pay-per-use → uses your credits
Knowledge Base storage	❌ Pay-per-use → uses your credits

So the only things that cost money are the LLM tokens and the KB index — and at your scale that's a few dollars, comfortably inside $200. ⁠

The catch (read this): Bedrock has no permanent free tier — you're spending credits. ⁠ And there's a documented grey area that Claude (a third-party model) may be billed via AWS Marketplace, which credits may not cover. ⁠ To remove all doubt and keep cost near zero, switch from Claude Haiku to an Amazon-native model (e.g. Amazon Nova Lite or Titan) — it's cheap, definitely covered by credits, and good enough for a 500-doc assistant. That's the one change I'd make to my earlier plan.

2. What you actually need to know (the GenAI minimum)
You do not need to learn "GenAI." You need ~10 words so you can explain the flow. You're the engineer who deployed a RAG app, not the person who built the model. Learn just these, one line each:

RAG = Retrieval‑Augmented Generation — "look up the relevant docs, then let the LLM answer from them."
Embedding — turning a piece of text into a list of numbers (a vector) that captures its meaning.
Vector store — a database of those numbers; you search it by meaning, not keywords.
Chunking — splitting each document into small pieces before embedding (you can't embed a 40‑page PDF as one blob).
Similarity search — "find the 3–5 chunks closest in meaning to the user's question."
The two steps — Retrieval (find the right chunks) → Generation (LLM writes the answer using those chunks).
Grounding — the answer is based on real docs, so it's citable and less made‑up.
Hallucination — the LLM inventing facts; RAG reduces this by forcing it to use retrieved context.
Knowledge Base — the AWS service that does chunk + embed + index for you automatically.
retrieve_and_generate — one Bedrock API call that does retrieval and generation in a single step (this is why your app is one small Lambda, not a big ML pipeline).
That's the entire GenAI surface you need. Everything else is DevOps you already know.

3. What interviewers WILL ask (and how you're positioned)
From the GenAI/architecture side (the ones that catch people off guard — you'll now be ready):

"What is RAG and why did you use it instead of fine‑tuning a model?" → Answer: no retraining, updates instantly when docs change, answers are grounded/citable, far cheaper.
"Walk me through what happens when a user asks a question." → The 5‑step flow in your diagram.
"How do you reduce hallucinations?" → Retrieval + grounding + citations.
"Why a Managed Knowledge Base instead of OpenSearch?" → This is your killer answer — I chose it to cut vector‑store cost from ~$345/mo (provisioned minimum) to ~$10/mo pay‑per‑use. That's a cost‑architecture decision, pure DevOps thinking.
From the DevOps side (your home turf — you'll ace these):

Why Terraform / IaC, how your CI/CD works, why OIDC (no AWS keys in GitHub), how you monitor (CloudWatch dashboard + alarms + SNS), how you handle secrets, how it scales, and your cost breakdown.
The pattern: the AI questions test whether you understand the flow; the DevOps questions test whether you can build and run it. You're strong on the second, and the 10 words above cover the first.

4. How much can you blindly follow me?
Honest breakdown:

~90% of the building is blind‑followable. Terraform, GitHub Actions, IAM, S3, CloudWatch — that's your world; I'll give you paste‑and‑run code. The AWS console clicks (enable model access, create the KB) — I'll give you exact step‑by‑step clicks. The Lambda handler — you paste it, and I'll explain the 5–6 lines that matter.
The ~10% that needs your brain is the interview, not the build. You can't "blindly follow" an interview — but I'll hand you the exact talking points (the 10 words + the cost‑decision story above) so you're not guessing.
Net: you follow my instructions to build it, and you study the 10‑word cheat sheet to explain it. Your DevOps knowledge is untouched and stays the star.

















⚠️ The Mumbai catch (this is the part that matters for you)
In ap‑south‑1 (Mumbai), Nova Lite is not available in‑region. You can't use the plain model ID amazon.nova-lite-v1:0 there — it will fail. ⁠

You must use the Geo cross‑region profile:

apac.amazon.nova-lite-v1:0

