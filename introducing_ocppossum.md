# Introducing OCPPossum

### How EVCS diagnoses sessions using AI
----
## Because we're a lean team, we need fast and trustworthy answers

We run a large charging network with almost two thousand charging ports, and we proudly make our lean organization a virtue rather than a constraint, pushing us to find creative and efficient ways to do more or do faster, staying nimble and simple.

Within the organization, we are not short of data, with every session recorded and every OCPP message logged and more. In these two years at EVCS, I created a suite of dashboards, fine-grained alerts, and detailed reports that allow us to monitor network performance down to the single charging port. I also built FlintBot, our internal chatbot and AI agent that can access all the data available across EVCS and provide detailed answers about (almost) everything.

However, one recurring question was still taking a significant amount of time, or a significant amount of tokens, to be answered: *why did this session fail?*

## One question, three systems

The reason session diagnosis is expensive, in time or tokens, is that no single system knows the answer and all the clues are spread across three sources:

The **session record** in the data warehouse knows what the driver experienced and what they were billed: when it started, when it ended, how much energy, how much the driver was charged, or if they used one of our subscription plans. But it knows nothing about how the charger actually performed.

The **charger registry** is the thing that maps a station to the OCPP charge point identifier the equipment reports under. It is a small, unglamorous mapping table, but key to connecting the session data with what actually happened on the ground.

The **OCPP logs** know exactly what the hardware did — every status change, every fault, every meter sample — organized by charging station and time. The OCPP logs provide us with a picture of what the driver experience was.

Unfortunately, due to the architecture of our systems, there is no shared key between a billing session and an OCPP transaction to identify the correct messages.

Reconciling a session with its messages from the EVSE means identifying which charger performed the session, which port, and when the session occurred, and verifying that overlapping data agree (for example, does the charger's meter report the same amount of kWh dispensed as our billing data?).

So, connecting a session to its logs isn't a simple join. It is a chain of five or six "judgment calls" that need to be assessed to have the complete picture.

## Do it once, in a custom MCP server

The obvious modern approach is to hand a language model access to all three systems and ask it the question. We think that is the wrong shape, and the reason is specific.

Until recently, I was delegating this reconciliation to the capabilities of the LLM powering FlintBot: through regular MCP servers, FlintBot can connect to our data warehouse running in Snowflake and the hundreds of terabytes of OCPP logs stored in an Elasticsearch index. Although the LLM is instructed to match a session with its logs, by the very nature of how LLMs work, this step is not deterministic: the model may decide to overlook some rules and return different answers to the same question at different times.

Furthermore, why should the bot burn tokens learning the same logic over and over again? Use your AI for what it is actually very good at: summarizing large amounts of structured and unstructured data in an intuitive, easily consumable format and quickly surfacing important or unexpected patterns that human eyes would take longer to find.

So I put the chain of "judgment calls" in code by writing my own MCP server, called **OCPPossum**. So that:

**It is repeatable.** The same session identifier produces the same reconciliation today, next week, and forever. No more erosion of trust due to stochastic choices.

**It is verifiable.** It's a deterministic checklist instead of a set of instructions that may or may not be followed by the model.

**It is cheap and fast.** One call instead of a dozen exploratory queries, and nothing spent rediscovering something we already documented.

That leaves a clean division of scope between components of our AI systems, connected by the Model Context Protocol. **The code embedded in the MCP server does the deterministic work: identity, correlation, validation, detection. The model does the linguistic work: explaining, contextualizing, and answering the follow-up question a human actually asks next.** Each side is doing something it is reliably good at.

## What it actually does

Given one session identifier, the tool runs six steps:

- **Resolve** the session — times, energy, connector type, payment.
- **Map** the station to its OCPP charge point identifier, and derive which physical connector was used, so concurrent sessions on a dual-port charger stay separate.
- **Fetch** every protocol message for that charger inside the session window, padded at both ends so the lead-up and the aftermath are visible — the failure often happens before the session officially starts.
- **Correlate** the billing session to the protocol transaction, recovering the transaction identifier from whichever message carries it, and attach a confidence to the result.
- **Reconcile** metered energy against billed energy, and flag disagreement.
- **Interpret** — build the event timeline, extract faults, reduce the meter samples to a charging curve, and run detection rules that separate *the vehicle stopped drawing* from *the charger stopped supplying*.

What comes back is a fixed summary, computed rather than described:

```
Date:               2026-07-29 (PST)
Site:               Northside Garage
Charger:            CHARGER-B (Vendor B, AC Level 2)
Connector:          1
Transaction:        #218
Start:              22:23:26 UTC  (meterStart: 7081876)
Stop:               15:34:28 UTC  (meterStop: 7142844 Wh)
Duration:           ~17.2 hr
Energy:             60.9682 kWh (matches billed KWH)
Stop reason:        "EVDisconnected" (The cable was unplugged)
Plug to charge:     43 sec
Idle after charge:  3 sec
Power:              peak 6.67 kW, avg 3.22 kW
Body temp:          ~42.0°C — normal
```

Alongside it, a single-line activity bar across the whole window — `#` charging, `~` plugged in but not drawing, `-` idle:

```
22:13 |###########################~~~~~~~~~~~~~~~~~~~~~| 15:44
```

That one line says something no table does. Nearly ten hours of genuine charging, and then seven and a half hours of a finished car still occupying the connector. Nothing failed. But that connector earned nothing for most of the night, and this is what that looks like.

## From NetOps to the Support Desk

This unlocks a lot of potential all around the monitoring and operations of EVCS, but in particular for two teams:

**Network Operations.** The first use is diagnosis at scale: instead of going through hundreds of rows of logs one ticket at a time, anyone can ask about any session and get a reconciled answer with the evidence attached. How did the session play out? Were there any errors reported and at what time? What does the charging curve look like? With one question in plain English, the engineer can narrow down the root cause of a faulty charger.

**Customer Support.** Perhaps the more interesting use: an agent on a live call can now have a richer picture of the driver's experience while charging, as well as the issue that may prevent a successful session. No warehouse access, no OCPP knowledge, no escalation to engineering, no callback needed. Is the connector not latching correctly? Is the handshaking happening? Can the session be troubleshot on the spot and recovered?

## The takeaway: improve your AI agent with custom tools

Stepping aside from the EV context: if you find yourself stuffing the system prompt of your AI agent with dozens of lines instructing the model on how to navigate convoluted, multi-hop business logic, consider moving that logic out of the prompt and investing your time in customizing the tools the model can use.

OCPPossum is an example of how you can improve your AI's answers by simply reducing the amount of guessing the model needs to do.
