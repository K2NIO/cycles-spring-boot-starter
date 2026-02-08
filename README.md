# Cycles: Economic Governance for Autonomous Agents (Spring AI)

**Stop your autonomous agents from bankrupting you.**

Cycles is a JVM-level **economic governance layer** for Spring Boot applications.
It enforces **deterministic spend limits** on guarded AI execution—preventing infinite loops, runaway recursion, and API bill shock *before* they happen.

Rate limiters control **velocity**.
**Cycles controls economic exposure.**

---

## 🛑 The Problem: “The $5,000 Loop”

You deploy an agent to summarize daily news.

A prompt bug causes it to retry endlessly on a `500` error from the provider.
You have a standard rate limiter: `100 req/min`.

**What happens:**

* **00:00** — Agent enters a retry loop
* **00:10** — Rate limiter allows ~1,000 calls
* **01:00** — ~6,000 calls
* **08:00** — You wake up to a **$4,500 OpenAI bill**

**Rate limits do not stop bad logic. They only slow it down.**

---

## ⚡ The Solution: Economic Governance

Cycles introduces a new primitive: **the Cycle**.

> **1 Cycle = 1 unit of execution risk**

A Cycle is a normalized measure of economic risk.
You decide what actions cost more based on blast radius:

* LLM calls
* Tool invocations
* Writes
* Deployments
* External side effects

Instead of limiting *requests per second*, Cycles limits **total risk budget per execution**.

As the budget tightens, Cycles **degrades behavior safely**.
When the budget is exhausted, execution **halts deterministically**.

---

## ✨ The Annotation

Add one dependency.
Add one annotation.
Define a hard economic envelope.

```java
@Service
public class ResearchAgent {

    private final OpenAiClient ai;

    // Economic envelope for this execution.
    // As the budget tightens, Cycles degrades capability and throttles.
    // If the execution budget is exhausted, Cycles halts deterministically
    // (with optional fallback or partial output per policy).
    @Cycles(
        profile = "agent-default",
        bucket = "execution",
        limit = 50
    )
    public Report generateMarketReport(String ticker) {

        // Even if this call loops indefinitely,
        // Cycles will cut the execution once the risk budget is exhausted.
        return ai.recursiveDeepDive(ticker);
    }
}
```

This applies to the **entire call graph**—not just this method.

---

## 🚀 Key Features

* **Atomic Accounting**
  Budget authorization and decrement happen atomically via Redis Lua scripts
  (single round trip, no race conditions).

* **Low Overhead**
  Typically single-digit millisecond overhead per charged action
  (Redis RTT dependent).

* **Context Propagation**
  Budgets travel with execution context.
  If Service A calls Service B, they share the same risk budget.

* **Real-Time Control (“Panic Button”)**
  Adjust budgets or policies live—no redeploy required.

* **Audit-Ready (Enterprise)**
  Tamper-evident execution ledger for compliance and forensic analysis.

---

## ⚠️ Enforcement Model (Important)

Cycles enforces spend limits **only on guarded execution paths**
(via `@Cycles` and Spring AOP interception).

* **Guarded execution** → deterministic enforcement
* **Unguarded execution** → allowed, but surfaced in the dashboard

This lets teams:

* Start with visibility
* Gradually harden enforcement
* Avoid breaking existing systems

---

## 📦 Installation

Add the starter:

```xml
<dependency>
  <groupId>io.cycles</groupId>
  <artifactId>cycles-spring-boot-starter</artifactId>
  <version>0.1.0-beta</version>
</dependency>
```

---

## ⚙️ Configuration

Cycles uses Redis as a distributed execution ledger.

```yaml
cycles:
  enabled: true
  storage: redis

  redis:
    host: localhost
    port: 6379

  profiles:

    agent-default:
      description: "Default agent execution profile with progressive degradation"

      buckets:
        - name: execution
          scope: EXECUTION
          key: "{executionId}"
          initialLimit: 200
          ttlSeconds: 3600

          thresholds:
            spent:
              yellow: 0.70
              orange: 0.90
              red: 1.00

        - name: agent
          scope: AGENT
          key: "{agentId}"
          initialLimit: 5000

        - name: global
          scope: GLOBAL
          key: "global"
          initialLimit: 100000

      policies:
        green:
          allow:
            actions: ["*"]

        yellow:
          degrade:
            modelTier: downgrade
            contextWindow: reduce
          throttle:
            minDelayMs: 250

        orange:
          block:
            actions: [WRITE, EXECUTE, DEPLOY]
          degrade:
            modelTier: minimal
          throttle:
            minDelayMs: 1000
          retries:
            max: 1

        red:
          onExhaust: HALT
          emit:
            mode: PARTIAL_RESULT
          fallback:
            strategy: SUMMARY_ONLY

      autoExtend:
        enabled: true
        maxExtra: 50
        windowSeconds: 300
        conditions:
          - errorRate < 0.01
          - riskLevel <= LOW
          - noBlockedActions == true

      notifications:
        onTransition:
          - to: ORANGE
            channel: LOG
          - to: RED
            channel: USER
            includeOptions: true


    group-strict:
      description: "Strict team-level budget (no degradation)"

      buckets:
        - name: group
          scope: GROUP
          key: "{teamId}"
          initialLimit: 20000

        - name: global
          scope: GLOBAL
          key: "global"
          initialLimit: 100000

      policies:
        red:
          onExhaust: THROW_EXCEPTION
```

---

## 🆚 Cycles vs. Rate Limiters

| Feature  | Rate Limiters     | Cycles                        |
| -------- | ----------------- | ----------------------------- |
| Metric   | Requests / second | **Risk / execution**          |
| Protects | Servers           | **Wallets**                   |
| Response | Throttle          | **Degrade → Restrict → Halt** |
| Scope    | Single service    | **Distributed call graph**    |
| Use Case | Traffic spikes    | **Autonomous agents & LLMs**  |

---

## 🧠 Architecture

Cycles uses an **atomic authorize-and-burn interceptor**.

1. **Intercept** guarded execution
2. **Authorize + decrement** budget atomically in Redis
3. **Verdict**

   * ✅ Solvent → proceed
   * ❌ Insolvent → halt execution (exception + optional fallback)

No probabilistic limits. No best-effort throttling.
Execution is **deterministically governed**.

---

## 🔮 Roadmap

* **v0.1** — Spring AOP + Redis (Beta)
* **v0.5** — Monitoring dashboard
* **v1.0** — `X-Cycles-Budget` HTTP context propagation

---

## 🤝 Private Beta

We are onboarding **Fintech and Enterprise Java teams** running Spring AI in production.

If runaway AI spend keeps you up at night, let’s talk.

👉 **[Request Access](https://docs.google.com/forms/d/e/1FAIpQLSd4FB1W_NrmHqf873lUUSP2V6_uWEVG2J6OteQ9hM8yWynKNQ/viewform?usp=dialog)**

---

**License:** Apache 2.0

---
