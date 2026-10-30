# Speaker Script — CloudxAI Conference
## "Who Fixes Your Kubernetes at 3 AM? Now It's an AI Agent."
### Bedrock AgentCore EKS Monitor

---

> **Talk format:** 30–40 minutes + 5–10 minutes Q&A
> **Audience:** Mixed — engineers, architects, product leaders, business decision-makers
> **Tone:** Conversational, story-driven, live-demo-friendly
> **Slides cue:** `[SLIDE X]` marks are placed throughout

---

## PRE-TALK CHECKLIST (10 min before)

- [ ] Port-forward webhook server running: `kubectl port-forward svc/alertmanager-webhook-server -n alertmanager-agent 8091:80 &`
- [ ] `curl http://localhost:8091/health` returns `{"status":"ok"}`
- [ ] Two terminals open: one for `kubectl apply`, one for `kubectl logs -f`
- [ ] Architecture diagram on screen as default slide
- [ ] JIRA project accessible in browser tab
- [ ] Slack channel open in browser tab

---

---

## SECTION 1 — THE HOOK  *(~3 minutes)*

---

`[SLIDE: Title slide — "Who Fixes Your Kubernetes at 3 AM? Now It's an AI Agent."]`

*[Walk to center stage. Pause. Make eye contact with the room.]*

Good morning, everyone. Before I start, I want to ask you a question.

**How many of you have ever been paged at 2 or 3 in the morning because a service went down?**

*[Pause. Look around. Smile.]*

Yeah. I can see a lot of tired hands going up. And for those of you who didn't raise your hand — you either work in product management...

*[Audience laughter]*

...or your on-call rotation just hasn't started yet.

*[Smile, then get serious.]*

Here's the thing. I've been in that situation. You wake up, you're groggy, your phone is screaming at you, and somewhere in a data center a pod has crashed — again. You SSH in. You run `kubectl describe pod`. You stare at the logs. You patch something. You go back to sleep. And two hours later, it crashes again.

This is the reality for millions of engineers running applications on Kubernetes today. And it doesn't have to be this way.

Today, I want to show you how we built an AI agent that **wakes up so you don't have to.** An agent that reads the alert, investigates the root cause, fixes what it can, and escalates what it can't — all within 60 to 90 seconds. Automatically. No human required.

---

---

## SECTION 2 — SETTING THE STAGE  *(~5 minutes)*

---

`[SLIDE: "The Modern Cloud Reality"]`

Let me give you some context first — and I promise to keep this jargon-free.

### What is Kubernetes? (For the non-technical folks in the room)

Think of Kubernetes — or K8s as engineers call it — like an **air traffic control system** for your software.

When a company builds an app today, they don't run it as one big program. They break it into dozens — sometimes hundreds — of small, independent pieces called **containers**. Think of containers like little self-contained boxes, each running one part of your application. One box handles user logins. Another handles payments. Another handles recommendations.

Kubernetes is the system that manages all those boxes. It decides which physical computer they run on. It makes sure they stay healthy. If one crashes, it tries to restart it.

Now — Kubernetes is incredibly powerful. It's what powers applications at Netflix, Airbnb, Spotify, NASA, and practically every major tech company on the planet. But it's also incredibly complex. When things go wrong inside a Kubernetes cluster, diagnosing the problem is like being a doctor who has to figure out which organ is sick — except you have 200 organs, they're all interconnected, and the patient can't tell you what hurts.

### The Scale of the Problem

`[SLIDE: "The On-Call Problem"]`

Let me give you some numbers that put this in perspective:

- The average engineering team running Kubernetes gets **15 to 30 alerts per week** from their monitoring systems.
- A skilled Site Reliability Engineer — an SRE — takes **5 to 20 minutes** to investigate and resolve a common alert.
- **40% of those alerts** are recurring — the same problems, over and over again. OOMKilled. CrashLoopBackOff. The same issues, every week.
- And on-call burnout is real. Studies show that engineers who are paged more than **2 to 3 times per night** show significantly reduced effectiveness the next day.

The dirty secret of the cloud industry is that we've built incredibly sophisticated infrastructure — and we're still relying on humans to babysit it at 3 in the morning.

**That changes today.**

---

---

## SECTION 3 — INTRODUCING THE PROJECT  *(~5 minutes)*

---

`[SLIDE: Architecture Diagram]`

I want to introduce you to **Bedrock AgentCore EKS Monitor**.

*[Point to the architecture diagram.]*

At its core, this is an AI-powered Kubernetes operations agent. Let me walk you through what you're looking at.

### The Story of an Alert — In Plain English

Imagine you're running an application on AWS. You have an EKS cluster — that's Amazon's managed Kubernetes service. Inside that cluster, things are humming along... until one of your pods runs out of memory and crashes. This is called an **OOMKilled** event — Out Of Memory Killed.

Here's what happens next, automatically, without any human intervention:

**Step 1 — Detection.**
Prometheus, an open-source monitoring tool, is constantly watching every pod in your cluster. The moment that pod crashes, Prometheus fires an alert. Within 30 seconds.

**Step 2 — Routing.**
Alertmanager receives that alert and, after a 10-second group window, fires a webhook — an HTTP call — to our Webhook Server running inside the cluster. Total time so far: about 40 seconds.

**Step 3 — Invocation.**
The Webhook Server reads the alert, builds a natural language prompt describing the situation — "This pod in this namespace crashed with this reason, please investigate" — and sends it to **Amazon Bedrock AgentCore**.

**Step 4 — The Agent Thinks.**
This is where the magic happens. Amazon Bedrock AgentCore Runtime — running on AWS Graviton ARM64 processors — spins up our AI agent. The agent is powered by **Claude 3.5 Haiku**, Anthropic's fast, intelligent model.

The agent is not just a chatbot. It has **tools**. Real tools that can interact with your Kubernetes cluster:
- It can list all pods and check their status
- It can fetch logs from a crashing container
- It can describe a pod and look at its events
- It can check what deployment owns that pod
- It can **patch** that deployment's memory limit
- It can trigger a rollout restart
- It can check node health and cordon an unhealthy node
- It can create a JIRA ticket
- It can send a Slack message

**Step 5 — Remediation.**
The agent investigates. It sees: OOMKilled, Deployment found, current limit is 32 megabytes, that's clearly too low. Decision: patch the memory limit to 64 megabytes — double it — and restart the pod. Done.

Total time from pod crash to pod running again: **60 to 90 seconds.**

**No engineer was paged. Nobody woke up.**

---

---

## SECTION 4 — LIVE DEMO  *(~8 minutes)*

---

`[SLIDE: "Let's Break Some Things"]`

Okay. Let's make this real. I'm going to break some things — intentionally — and watch the agent fix them.

*[Open terminal. Make sure the webhook server health check is confirmed.]*

### Demo 1 — OOMKilled

I have a test deployment that runs a stress tool with an intentionally tiny memory limit of 32 megabytes. It's going to exhaust that memory immediately and crash.

```bash
kubectl apply -f app/test-crash.yaml
```

*[Switch to watching pods.]*

```bash
kubectl get pods -n default -w
```

*[Talk while waiting — about 60–90 seconds.]*

You can see the pod starting... and now it's OOMKilled. That's Kubernetes telling us: this container used more memory than we allowed, so we killed it.

Now let's watch the agent's logs.

```bash
kubectl logs -n alertmanager-agent deployment/alertmanager-webhook-server -f
```

*[Point to logs showing the alert being received, the prompt being built, the AgentCore invocation.]*

And there it is. The agent received the alert, investigated, saw the OOMKilled reason, looked up the deployment, and patched the memory limit. Let's verify.

```bash
kubectl get deployment demo-app-crashing -n default \
  -o jsonpath='Memory limit: {.spec.template.spec.containers[0].resources.limits.memory}{"\n"}'
```

It went from 32Mi to 256Mi — doubled, because that's what the agent decided based on its instructions. Pod is now Running.

*[Pause for effect.]*

**I did nothing.** The AI did it.

---

### Demo 2 — CrashLoopBackOff

This one is even more common in real production. A pod that starts, crashes immediately, starts, crashes again — Kubernetes calls this CrashLoopBackOff. It's one of the most irritating things to debug because the pod never stays up long enough to connect to it.

```bash
kubectl apply -f app/test-crashloop.yaml
kubectl get pods -n default -w
```

*[Wait for CrashLoopBackOff status.]*

The agent will investigate, see it's a CrashLoop, and execute a rollout restart — a fresh deployment of the pod. Sometimes a transient failure in initialization is all it is, and a restart resolves it.

*[Show the agent log and the restart happening.]*

---

### Demo 3 — ImagePullBackOff (Escalation)

This one is different. I have a pod trying to pull a Docker image that doesn't exist.

```bash
kubectl apply -f app/test-imagepull.yaml
```

This is a problem the agent **cannot fix on its own.** No amount of restarting will help if the image doesn't exist — someone needs to fix the code or the registry credentials. So instead of blindly restarting things, the agent does something smarter.

*[Wait and watch logs.]*

It immediately recognizes: `ImagePullBackOff` — this is not auto-fixable. It calls `create_jira_ticket` and `send_slack_notification`. Let's switch to our browser.

*[Switch to JIRA — show the ticket was created with the alert details, namespace, pod name, and description.]*

*[Switch to Slack — show the notification message.]*

The on-call engineer wakes up tomorrow morning, sees a JIRA ticket, sees a Slack message with context already written — namespace, pod, root cause — and can go directly to fixing it. No investigation time lost.

---

---

## SECTION 5 — HOW THIS HELPS  *(~5 minutes)*

---

`[SLIDE: "The Business Impact"]`

Let me step back from the technical details and talk about what this actually means for your organization.

### For Engineering Teams

The most immediate impact is **reduced toil**. Toil is what Google's SRE book calls "manual, repetitive, automatable work." OOMKilled and CrashLoopBackOff alerts are the definition of toil. The same fix, over and over. An engineer shouldn't need to do that at 3 in the morning, or at 3 in the afternoon for that matter.

With this system, your engineers spend their time building features — not babysitting pods.

**Quantified:** If you have a team getting 20 alerts per week, with an average 10-minute resolution time, that's over 3 hours of engineer time per week consumed by repetitive Kubernetes firefighting. Multiply by 52 weeks. Multiply by the cost of an engineer. You're looking at tens of thousands of dollars per year, per team, in recovered productivity.

### For On-Call Culture

On-call burnout is a real problem in the industry. Engineers leave companies because of unsustainable on-call rotations. Every unnecessary page erodes trust and morale. A system that handles the automatable cases means fewer pages, better sleep, and engineers who actually want to be on call.

### For Reliability

Here's the counterintuitive part: **the AI is often faster and more consistent than a human.** When I'm woken up at 3 AM, I might misread the logs. I might accidentally restart the wrong pod. I might forget to patch the memory limit and just restart, so it crashes again in 20 minutes.

The agent doesn't get tired. It doesn't guess. It follows a precise decision tree every single time. And if it's not sure — if the case is ambiguous — it escalates rather than risks making things worse.

### For Business Continuity

The time between a pod crashing and an engineer fixing it is downtime. Real downtime that real users experience. Reducing that from 15–20 minutes to 60–90 seconds is a 10–15x improvement in mean time to recovery, or MTTR. For a business processing transactions, that's not just a nice-to-have — that's a competitive advantage.

---

---

## SECTION 6 — UNDER THE HOOD  *(~5 minutes, for technical audience)*

---

`[SLIDE: "How It Works — The Technical Story"]`

For the engineers in the room, let me go a level deeper.

### Amazon Bedrock AgentCore

The heart of this system is **Amazon Bedrock AgentCore** — a managed runtime for AI agents launched by AWS. Think of it as Lambda, but for agents. You give it:
- A container image with your agent code
- An IAM role with the permissions your agent needs
- Environment variables for configuration

And AgentCore handles:
- Spinning up the container when invoked
- Managing the session state
- Handling the invocation lifecycle
- Running on Graviton ARM64 for cost efficiency

You invoke it with a simple boto3 call: `invoke_agent_runtime()`. It sends your payload to the container, runs your agent logic, and returns the response. Fully serverless. No EC2 to manage. No Kubernetes to debug. *(The irony of using Kubernetes to debug Kubernetes is not lost on me.)*

### Strands Agents Framework

The agent itself is built on the **Strands Agents** framework — an open-source SDK for building tool-using agents. It's declarative and elegant. You define your tools — regular Python functions decorated with `@tool` — and pass them to the `Agent` constructor along with your model and system prompt. Strands handles the entire agentic loop: calling the model, parsing tool calls, executing tools, feeding results back, and deciding when to stop.

The system prompt is the "brain" that governs behavior. It's a precise set of decision rules:
- If `ImagePullBackOff`: immediately escalate. Never restart.
- If `OOMKilled`: patch memory to 2× current (capped at 2048 MB), then restart.
- If `CrashLoopBackOff`: rollout restart once.
- If `NodeNotReady`: check conditions, cordon the node, always escalate — nodes need human eyes.
- If `kube-system` or `monitoring` namespace: ignore entirely. Never touch platform internals.

These guardrails are critical. You don't want an AI agent accidentally restarting your Prometheus stack.

### Authentication — IRSA

The webhook server uses **IRSA** — IAM Roles for Service Accounts — to authenticate to AWS. This is Kubernetes-native IAM: each pod gets an AWS role via a signed JWT from the EKS OIDC provider, exchanged against STS. The AgentCore runtime has its own IAM execution role, added to the EKS `aws-auth` ConfigMap so it can call the Kubernetes API directly.

No static credentials. No secret rotation headaches. Pure AWS IAM.

### Infrastructure as Code

Everything is Terraform. The EKS cluster references, the ECR repository, the S3 bucket for code artifacts, the IAM roles, the Kubernetes resources — all of it. You can tear it all down and bring it back with one command.

---

---

## SECTION 7 — THE FUTURE  *(~5 minutes)*

---

`[SLIDE: "The Future — From Reactive to Predictive"]`

What we've built today is a **reactive** agent. Something breaks — agent fixes it. That's already a huge step forward. But this is just the beginning.

### Near-term Roadmap (6–12 months)

**Multi-cluster and multi-cloud support.**
Today this works on one EKS cluster. The natural next step is supporting multiple clusters — even across regions and cloud providers. Imagine a single agent that monitors your production cluster in us-east-1, your staging cluster in eu-west-1, and your GKE cluster on Google Cloud — all from one deployment.

**Richer escalation playbooks.**
Right now, when the agent escalates, it creates a JIRA ticket and sends a Slack message. What if it also attached the last 50 lines of logs, the deployment spec, a link to the relevant Grafana dashboard, and a suggested fix based on similar past incidents? The ticket would arrive with 80% of the investigation already done.

**Memory and learning.**
The current agent has no memory between alerts. Each invocation is fresh. A next-generation version would maintain a memory store — perhaps backed by Amazon Bedrock Knowledge Bases — of past incidents and resolutions. The agent would know: "This specific deployment has OOMKilled three times in the past week. Doubling the memory limit keeps failing. Let me escalate and recommend a code review instead."

**Remediation confidence scoring.**
Before taking action, the agent could output a confidence score. Above 90%? Auto-fix. Between 60–90%? Fix but notify. Below 60%? Escalate only. This gives teams the ability to tune autonomy levels.

### Medium-term Vision (1–2 years)

**Predictive operations — before the alert fires.**
What if the agent didn't wait for a crash? What if it watched the trend lines? "This pod's memory usage has grown 20% in the last hour. At this rate, it will OOMKill in 45 minutes. I'm patching the limit now." Prometheus can export these trend metrics. The agent can reason about them.

**Capacity planning as a tool.**
The agent could analyze historical resource usage patterns and make recommendations: "Your demo-app deployment consistently uses 80% of its CPU limit between 2 and 4 PM on weekdays. You should increase the limit or trigger an autoscaler policy." From reactive to advisory.

**Integration with CI/CD.**
When the agent determines that a crash is caused by a bad deployment — a new image that's broken — it could automatically trigger a rollback via your CI/CD pipeline. ArgoCD, FluxCD, GitHub Actions — these all have APIs. A fully autonomous "deploy, observe, rollback" loop.

### Long-term Vision (3–5 years)

`[SLIDE: "The Autonomous Cloud"]`

Here is where I want to get a little philosophical — because I think what we're building here is more significant than it might appear.

For the last 20 years, cloud operations has been fundamentally a **human cognitive task**. You needed experienced engineers who understood how systems fail, what the error messages mean, and how to fix them. That knowledge was expensive, rare, and couldn't scale to the size of modern infrastructure.

Large Language Models changed that equation. For the first time, we have systems that can read a stack trace and understand it. That can look at a set of events and reason about causality. That can decide between remediation strategies based on context.

We are moving toward a world where the cloud becomes **self-operating** — not in a scary science fiction sense, but in the same way that your car's antilock brakes operate automatically without you thinking about them. The complexity is hidden. The system handles it.

The role of the engineer in this world isn't eliminated — it's **elevated**. Instead of debugging pod crashes at 3 AM, you design the decision frameworks the agent uses. You review the cases it escalated. You improve the playbooks. You focus on the 5% of problems that genuinely require creative human problem-solving.

This is the promise of **agentic operations**: reliable systems that run at cloud scale, healed by AI agents, with humans in the loop for what matters most.

---

---

## SECTION 8 — WHY THIS PROJECT MATTERS NOW  *(~2 minutes)*

---

`[SLIDE: "Why Now?"]`

You might be wondering — why is this possible now, and not five years ago?

Three things converged:

**First, the models got good enough.** Claude 3.5 Haiku is fast, accurate, and can reason about structured data like Kubernetes events and pod descriptions. Two years ago, you couldn't reliably prompt a model to make a correct decision between "patch memory" and "escalate to JIRA" based on an alert payload.

**Second, the infrastructure for agents matured.** Amazon Bedrock AgentCore is brand new. The Strands Agents framework is brand new. A year ago, building a reliable, serverless, production-grade agent runtime required enormous custom engineering. Today it's a pip install and a container push.

**Third, the operational pain hit a critical mass.** Cloud infrastructure has grown faster than teams can hire. The average team managing Kubernetes in 2025 is stretched thinner than ever. The economics now strongly favor automation.

The timing is right. The technology is ready. The need is acute.

---

---

## SECTION 9 — CALL TO ACTION + CLOSE  *(~2 minutes)*

---

`[SLIDE: Project GitHub + Architecture Diagram]`

The code for everything I showed you today is open source. The architecture diagram, the Terraform configs, the agent code, the monitoring configs — everything is in the repository linked on this slide.

If you're running Kubernetes today, this is something you can stand up in an afternoon. One EKS cluster, a `bash deploy.sh`, a Helm chart, and you're monitoring with AI.

If you're not running Kubernetes but you're thinking about cloud operations at scale, I hope today gave you a mental model for how agentic AI fits into your operational stack — not as a replacement for your engineers, but as a force multiplier that handles the repetitive, the automatable, and the 3 AM pages so your team can focus on what they do best.

To summarize what we covered today:

- **The problem:** Kubernetes operations generates enormous amounts of repetitive, automatable toil that steals engineering time and drives on-call burnout.
- **The solution:** An AI agent powered by Claude on Amazon Bedrock AgentCore that detects, investigates, and remediates common Kubernetes failures in under 90 seconds — automatically.
- **The value:** 10–15x faster recovery, reduced engineering toil, better on-call health, and a foundation for predictive autonomous operations.
- **The future:** Predictive scaling, multi-cluster support, self-healing CI/CD pipelines, and ultimately a cloud that operates as autonomously as the infrastructure it runs.

We are building the nervous system for the autonomous cloud. And it starts with a pod that crashes at 3 AM — and fixes itself before you even feel your phone buzz.

Thank you.

*[Pause. Smile.]*

I'm happy to take questions, show more of the code, or go deeper on any part of this. Who wants to go first?

---

---

## Q&A PREPARATION — LIKELY QUESTIONS & SUGGESTED ANSWERS

---

**Q: "What if the AI makes a wrong decision and makes things worse?"**

Great question, and it's one I take seriously. The agent operates within hard guardrails. It will only patch memory up to a cap of 2 gigabytes. It will only restart a workload once. It will never touch system namespaces like `kube-system` or `monitoring`. For any ambiguous case — anything that doesn't clearly match OOMKilled, CrashLoop, or ImagePullBackOff — it escalates to a human rather than guessing. The blast radius of any single decision is deliberately limited. And because every action is logged, you have a full audit trail to review.

---

**Q: "How does it authenticate to Kubernetes? Is it secure?"**

It uses IRSA — IAM Roles for Service Accounts. The webhook server inside the cluster has an AWS IAM role scoped only to what it needs: invoke AgentCore. The AgentCore runtime has its own IAM role, added to the EKS `aws-auth` ConfigMap with controlled RBAC. No static credentials anywhere in the system. Everything is signed, scoped, and rotated automatically by AWS.

---

**Q: "What about other cloud providers — Azure AKS, Google GKE?"**

The agent code itself is cloud-agnostic. The Kubernetes Python SDK works against any conformant cluster. The AgentCore-specific parts are AWS, but conceptually you could swap AgentCore for a different agent runtime. The IRSA authentication layer would need replacement — Azure has Workload Identity, GKE has Workload Identity Federation, both do the same thing. It's an afternoon of work to port.

---

**Q: "Could this be used for databases, not just Kubernetes?"**

Absolutely. The agent pattern — alert fires, agent investigates with tools, agent either fixes or escalates — is completely general. You could give the agent tools to query a database, check replication lag, promote a read replica, or restart a connection pool. The Strands framework and AgentCore runtime work for any domain. Kubernetes is just the first domain we built tools for.

---

**Q: "What does this cost to run?"**

Very little. AgentCore runtime charges are invocation-based — you pay only when an alert fires. Claude 3.5 Haiku is one of the most cost-efficient models available. The Prometheus/Alertmanager stack is open source. An average team with 20 alerts per week would spend a few dollars per month on AI inference. Compare that to even one hour of engineer time per week.

---

**Q: "What happens if AgentCore itself is unavailable?"**

The webhook server returns a 200 status to Alertmanager even when AgentCore fails — this prevents Alertmanager from retrying infinitely and flooding the system. The failure is logged. Alertmanager's own alert routing still fires Slack/email alerts through its standard channels. The AI layer is an enhancement, not a single point of failure for the existing alerting pipeline.

---

**Q: "How do you prevent alert storms? If 50 pods crash at once..."**

Two mechanisms. First, Alertmanager's grouping: alerts from the same alert rule in the same namespace are grouped together, so 50 OOMKilled events become one webhook call. Second, the agent's session ID is scoped per-alert fingerprint — each unique alert gets its own conversation context, preventing cross-contamination between concurrent incidents.

---

**Q: "Can it handle custom application errors, not just Kubernetes-level failures?"**

Currently it handles infrastructure-level errors — the things Kubernetes itself tells you about. For application-level errors — "this microservice returned HTTP 500 because of a null pointer in business logic" — you'd need to wire in application-level metrics and expose them via Prometheus. That's absolutely possible and would be a natural extension. The agent can already read pod logs; reading those logs and reasoning about application errors is a capability the model already has.

---

---

## TIMING GUIDE

| Section | Content | Duration |
|---------|---------|----------|
| 1 | Hook — the 3 AM story | 3 min |
| 2 | Context — Kubernetes, the scale of the problem | 5 min |
| 3 | Introducing the project — architecture walkthrough | 5 min |
| 4 | Live demo — 3 scenarios | 8 min |
| 5 | Business impact — why this helps | 5 min |
| 6 | Technical deep-dive (optional, for tech audience) | 5 min |
| 7 | The future roadmap | 5 min |
| 8 | Why now | 2 min |
| 9 | Call to action + close | 2 min |
| **Total** | | **~40 min** |
| Q&A | | 5–10 min |

> **For a 25-minute slot:** Cut Section 6 (technical deep-dive) and compress Section 7 to 2 minutes. The hook, demo, impact, and close are non-negotiable.

> **For a 20-minute slot:** Cut Section 6 entirely. Run only Demo 1 (OOMKilled) live; describe Demo 2 and 3 verbally. Compress future to 60 seconds.

---

---

## KEY PHRASES TO MEMORIZE

These are the one-liners that land with mixed audiences. Use them as anchors.

- *"The AI wakes up so you don't have to."*
- *"60 to 90 seconds — from pod crash to pod running. No human required."*
- *"It knows what it can fix. It knows what it can't. And it knows the difference."*
- *"We're not replacing engineers. We're deleting the parts of the job that shouldn't require an engineer."*
- *"Every unnecessary page is a withdrawal from the trust and energy of your team."*
- *"This is the nervous system for the autonomous cloud."*

---

*End of Speaker Script*
*Version: CloudxAI Conference 2026*
