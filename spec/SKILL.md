---
name: spec
description:
  Collaborative domain-document (spec) creation for non-trivial work —
  elicitation loop, 10-section structure, concision and adversarial review
  gates, and blast-radius assessment for existing features or new work.
disable-model-invocation: true
---

# Using this workflow in Pi

Start with `/skill:spec <problem, existing feature, or document path>`. If the target is unclear, ask. This is the ten-section domain-document workflow; `/spec` and `/skill:spec-workflow` remain separate.

Begin with read-only research using the available file and search tools. Ask a few free-text questions, then wait for answers. Draft in conversation first unless a specific write is authorized. Follow project conventions. Ask separately before running repository scripts, changing Git state, sharing material with another reader or service, or publishing. A review gate is not permission to act. These boundaries apply throughout this skill, including initiative writes and editorial edits.

For an existing feature, distinguish observed behavior from intended guarantees, known gaps, and open decisions. Code answers what happens, not necessarily why it should happen. Don't invent a proposed change or success metric for a documentation-only baseline.

Adapted from [the original spec skill](https://github.com/tony-michaelson/skills/blob/bf96c11840133f85a3047a1f640c9e6d2d7b41ca/skills/spec/SKILL.md).

# What We Mean By "Spec"

When we say we need a **spec**, we don't mean a traditional requirements
document. We mean a **domain transfer document** — a vehicle for transferring
mental models from one mind to another (human or AI agent).

## The Problem with Traditional Specs

Traditional specs assume:
- The reader already understands the domain
- The reader shares your mental model
- Technical requirements are self-explanatory
- "What to build" is sufficient

When you hand a traditional spec to an agent (or a new engineer), they fill
in the gaps with their own assumptions. The result rarely matches your intent
because **the constraints and mental model never transferred**.

## What We Build Instead

Our "specs" are **domain documents** that transfer:

- **The world the system observes** — the domain model: things, rules, and events that exist whether or not the software does
- **Philosophy and key principles** — foundational truths and motivations that shape every decision
- **Scope and boundaries** — what this system is responsible for, and what it explicitly is not (system scope is distinct from the domain model)
- **Mental model** — how to think about this problem space; what's essential vs accidental
- **Integration context** — how this system fits into the larger ecosystem; what it calls and what calls it
- **Constraints and gotchas** — non-obvious failure modes, assumptions, and edge cases
- **Narrative perspective** — what the world looks like from different stakeholder viewpoints

The next section gives the document's canonical 10-section structure that captures these themes.

## Structure of a Domain Document

### 1. User Narrative
*"A day in the life" — concrete stories showing the system in use*

Not: "The system processes delivery events."

But: "Maria gets a delivery-exception alert for parcel WP-48213 within two
minutes of the failed attempt. She opens the notification, sees the
customer's name and phone number, and calls before the customer notices the
delay. The customer asks to redirect the parcel to their office; Maria
approves the redirect and the driver gets the new instruction on the next
stop."

### 2. Philosophy and Key Principles
*The foundational truths that guide all decisions*

Example:
- "Deliveries happen in the real world whether or not the system can link them to parcels"
- "Pickup location ≠ current hub ≠ destination"

These aren't requirements — they're truths about the domain that shape how you think.

### 3. Domain Model

**Model the world, not the system.**

This section captures the *world* this system observes — the things, the rules, the events that exist whether or not any software runs. Shipments, parcels, hubs, drivers, customers, delivery attempts. Not tables, rows, queues, cascade deletes, fan-out, retry logic, or "match candidates" — those are the system's machinery for *tracking* the world. They belong in the Integration Points, Schema, and Implementation sections, not here.

A common LLM failure mode: writing a "domain model" that's a database schema or processing pipeline in disguise. Storage choices ("we use a `scan_events` anchor table"), processing artifacts ("scans get fanned out to one row per parcel"), and uncertainty machinery ("match candidates with scores") are not domain content — they exist because of the gap between the world and what the system knows about the world. Keep that gap on the implementation side.

Test: if your sentence could be rewritten to describe something a dispatcher, courier, warehouse manager, or customer would recognize as part of their *job* — keep it. If it only makes sense to someone reading the codebase — move it out of this section.

A complete domain model captures:

**Ubiquitous Language** — The canonical names for every thing in the world this system reasons about. Used identically in the spec, the code, the database, and conversation with subject-matter experts. List each term, its precise meaning, and what it is *not*.

> *Shipment*: a collection of one or more parcels tendered by a single shipper to a single consignee at one time. Not the same as a delivery (a delivery is an outcome; a shipment is the thing being delivered).
>
> *Parcel*: a single physical package with its own tracking identity. Not the same as an "order item" — an order item is what the customer bought; a parcel is what the carrier carries. One order item may arrive in several parcels; one parcel may contain several order items.

**Entities** — Things in the world with identity that persists over time. A Parcel is the same physical package even if it's rerouted, relabeled, or delayed — until it is physically opened and re-packed into new parcels. For each entity: what identifies it *in the world* (a tracking number, an order number, a driver badge), and what makes two of them the same in the eyes of someone who works with this domain daily?

**Value Objects** — Things defined entirely by their attributes; no independent identity in the world. Two Addresses with identical components are interchangeable. A weight of 2.3 kg is the same value as any other 2.3 kg. Value objects are conceptually immutable — you don't "change 2.3 kg into 2.4 kg"; you have a new weight.

**Aggregates** — Things that cohere as a single unit *in the world*, not in the database. A Shipment is one tender: the shipper, consignee, parcels, and service level are all *parts of* the shipment — not separate things that happen to relate to it. The aggregate boundary is "what the world treats as one thing." If a piece can be described in isolation without losing meaning, it's a separate aggregate; if it only makes sense in the context of its parent, it's part of that aggregate. (Do not phrase aggregates in terms of database transactions; that's an implementation translation, not the domain concept.)

**Relationships** — How things in the world relate. State cardinality, direction, and what the relationship *means*.

> A Shipment contains 1+ Parcels tendered by one Shipper to one Consignee.
> A Parcel is physically located at exactly one Hub at a time (hubs are hierarchical: region ⊃ district ⊃ hub).
> A Delivery covers exactly one Parcel for exactly one delivery attempt.

**Lifecycles** — How things in the world come into being, change, and end. State machines with explicit terminal states.

> A Delivery: attempted → rescheduled → attempted again. Terminal: delivered (consignee or their delegate signed) or returned (sender reclaimed after max attempts).
> A Pickup: requested → scheduled → collected. Terminal: cancelled (shipper withdrew it) or expired (no parcel tendered by the cutoff time).

These are lifecycles *in the world*. The system observes them through scans and feeds; the system's *own* lifecycle for tracking them (ingested → matched → resolved) belongs in the Schema and Implementation sections.

**Invariants** — Rules that must always hold *about the world*, not constraints the system imposes on its data.

> A Parcel is at some physical location at all times (there is no "nowhere").
> A Shipment has at least one parcel and exactly one consignee (a tender with no parcels is not a shipment).
> A Delivery is associated with exactly one parcel and one attempt (a delivery "of everything on the truck" is not a delivery).

If a "rule" is really about the system's storage or processing (e.g., "every row has a non-null primary key", "events are processed in arrival order"), it's not a domain invariant — it's a schema or pipeline constraint and belongs elsewhere.

**Domain Events** — Things that *happen in the world* and matter to a stakeholder. Past-tense facts. Distinct from system events (HTTP requests, row insertions, queue messages).

> *ParcelDelivered* — a parcel has been handed over to the consignee or their delegate, ending its delivery lifecycle.
> *DeliveryFailed* — a delivery attempt was made but the parcel could not be delivered (no answer, bad address, refused).
> *ReturnInitiated* — a shipment is heading back to the shipper after exhausting delivery attempts.

For each: what real-world fact does it announce, and which stakeholder cares (customers, shippers, carriers, hub operators)? "Who subscribes to this message bus" is not a domain question.

*Note: not every spec needs every subsection. A pure reporting system may have no domain events of its own creation; a stateless transformer probably has no entities. Mark unused subsections "N/A: [reason]" rather than padding with low-value content. But the absence of any entities should provoke the question "what does this system reason about, then?"*

### 4. Scope & Boundaries
*What is this thing? Where do its responsibilities end?*

- What does this system own?
- What is explicitly NOT this system's job?
- What are the boundaries and contracts?

Example: "The notification service owns shipment-event matching. It does NOT own authentication, event generation, or notification delivery retries."

### 5. Key Concepts
*Non-obvious ideas worth standalone explanation — usually because they bridge the domain model and the system's implementation, or because they're foundational mental shortcuts the engineer relies on without realizing. Don't put pure domain content here (that goes in the Domain Model section); don't put schema or algorithm details here (those go in Schema and Implementation).*

Not: "The system uses a trie."

But: "Stakeholders reason hierarchically — a retail customer watches a single order, a fulfillment-center manager watches everything routed through their center, a regional carrier director watches the entire region. Operational interest is nested: an event at a district hub is also in the region. The system's matching exists to deliver every event to every subscriber whose interest level contains it."

### 6. Integration Points
*How does this fit into the larger system?*

- What does this system call?
- What calls this system?
- What does it look like from the outside?
- Functional core vs imperative shell placement

### 7. APIs and Contracts
*The explicit interfaces*

- Request/response shapes
- Idempotency semantics
- Error handling
- Why the contract looks the way it does
- **State cross-cutting contracts once.** Some rules apply to *every* code path,
  not just one — for example a single convention for signalling
  success/partial/failure (illustrative: "emit `OK` only when the full request
  succeeded; a partial result reports `Incomplete` and exits non-zero"). Write
  that rule here, once, so every path implements the same one. Left implicit,
  each path invents its own version and reviewers re-litigate it one path at
  a time.

### 8. Schema and Implementation
*How does it achieve the APIs?*

- Database tables
- Materialized views
- Indexes and constraints
- Why these choices?

### 9. Alerting, Observability, and Success Metrics

#### Failure Detection
*How will engineers know when this system is unhealthy?*

- What signals indicate the system is failing or degraded?
- Which alerts need to be created or updated? Map each failure mode to the
  right tier for your stack:
  - **On-call alerting** (e.g. PagerDuty): failure modes that need immediate response
  - **Operational metrics** (e.g. Prometheus): threshold-based alerting rules
  - **Log queries** (e.g. your warehouse-backed log search): saved queries for failure patterns not covered by metrics
  - **Product analytics** (e.g. Amplitude): funnels and feature health only — NOT operational alerts
- What is the blast radius of a silent failure — would anyone notice?
- What thresholds are meaningful? What is noise vs. signal?

#### Success Metrics
*How will we know this worked — not just that it didn't break?*

This is a different question from failure detection. Failure detection asks
"is it broken right now" and resolves in seconds by paging someone. Success
metrics ask "did this move the number it was built to move" and resolve on
a delay measured in days or weeks. Don't fold one into the other.

**Start with the objective, not the metric.** A metric is a *proxy* for a
business outcome. Written down without the outcome it stands for, it drifts
toward whatever number is easiest to move, and "did the metric move" ends up
answering the wrong question. State the objective first, then the metric that
proxies for it, then whether that proxy is faithful.

- **What measurable business objective does this work serve?** The outcome
  someone outside the team would care about — revenue, retention, cost,
  support load, engineer time. Not a capability ("engineers can use X") and
  not a task ("the migration is finished").
- **What metric proxies for that objective?** Name it precisely — not
  "improve engagement" but "7-day retention for new signups."
- **Is it a faithful proxy?** How could this metric move while the objective
  doesn't — or the objective improve while the metric sits still? If you
  can't tell those apart, the metric is measuring the wrong thing.
- **What is the baseline today, and what is the target?**
- **Where does the metric come from** — an existing analytics funnel, a
  data-warehouse query, a dashboard that already exists, or one that needs
  to be built?
- **What is the soak period** before the metric is meaningful? A day of data
  is noise; know how long to wait before checking.
- **Who owns checking back** and reporting the result? This doesn't resolve
  automatically when the code merges. Because a success metric can't resolve
  until a soak period passes after ship, it never belongs in a ladder's
  acceptance criteria — an acceptance check may verify that the
  *instrumentation* exists, never that the *metric moved*.

**Acceptance criteria are not success metrics.** "All 7 commands work
identically," "test coverage ≥ 90%," "100% of engineers can run the new
tool" — each becomes true the moment the work ships correctly. Those are
phase-level checks and belong in the ladder. A success metric is a number
that can still disappoint you after everything shipped exactly as intended.
If an entry cannot disappoint you, move it to the ladder.

*Not every spec has a success metric — a pure infrastructure or refactoring
spec may have none. Mark it "N/A: [reason]" rather than inventing a number to
fill the section. But "N/A" asserts that no business outcome motivated this
work; if that feels wrong, the objective exists and hasn't been articulated
yet.*

### 10. Gotchas and Constraints
*All the things that aren't obvious*

- What breaks if you do X?
- What assumptions must hold?
- What are the scaling characteristics?
- Where can you throw money at problems?

## Example: The Difference

### Traditional Spec Style
```
The system shall accept webhook subscriptions with hierarchical filters.
Subscribers can target by order, fulfillment center, or region.
The system shall deliver events matching the filters.
```

*Agent reads this and builds whatever it imagines.*

### Domain Document Style
```
The notification service isn't in the authentication business. By the time
a request reaches it, authentication is solved. It trusts the upstream
assertion: "this is subscriber X, they're allowed to do this."

Subscribers reason hierarchically. A retail customer might watch a single
order; a fulfillment-center manager watches everything routed through
their center; a regional carrier director watches the whole region. The
same real event reaches all three if it involves the right parcel —
operational interest is nested: a pickup at a Memphis hub is also in the
Mid-South district and in the Southeast region.

Not all event types support every level. Pickup scans have unreliable
district data — rural routes, subcontracted last-mile carriers that don't
report districts. The world imposes this limit; subscribers can't ask for
district-level coverage on pickup events. The system enforces it at
subscription creation.

Pickup location, current hub, and destination are three distinct things
that may or may not align. A parcel scanned at the Memphis hub might be
destined for Seattle — or it might be going to a cross-dock twenty miles
away. Subscribers watching "outbound from Memphis" filter on scan
location, not on destination or final delivery. The model can't conflate
the three.
```

*Agent reads this and understands the constraints, the philosophy, the gotchas — all in the language of the world, with the system's enforcement role made explicit but secondary.*

## When to Write a Domain Document

Write a domain document when:
- You're building a new system or major component, or documenting an existing one whose domain knowledge is not yet captured
- An agent will implement based on this document
- The domain has non-obvious constraints
- Multiple people/agents will work on this over time
- You need future maintainers to understand "why"

Don't write a domain document when:
- The change is trivial (fix a typo)
- The domain model is already well-established
- You're making a small isolated change

## Success Criteria

A good domain document succeeds when:
- An agent builds what you intended, not what it imagined
- A new engineer reads it and says "oh, now I get it"
- Future you reads it and remembers why decisions were made
- The constraints are so clear that wrong implementations feel obviously wrong

## Summary

**Traditional spec**: "Here's what to build."

**Domain document**: "Here's how to think about this problem, where the boundaries are, why it's shaped this way, and what the gotchas are. Now build it."

We're transferring mental models in whole cloth, not just writing requirements.

---

# Collaborative Domain Document Creation

A guide for LLMs working with engineers to capture domain models.

## Your Role as the LLM

You are the **naive but intelligent reader**. You have vast pattern knowledge but no domain intuition. This is your strength in this collaboration.

Your job is to surface gaps. The engineer knows things they haven't written down — assumptions, constraints, reasons for decisions. These live in their head, invisible to them because they're obvious. They are not obvious to you, and they won't be obvious to whoever implements from this document.

**You are the proxy for every future reader.** If you don't understand something, neither will they. If you have to guess, so will they. Your confusion is signal, not failure.

## The Collaboration Loop

```
Engineer provides information
       ↓
You identify gaps in understanding
       ↓
You ask questions that surface unstated assumptions
       ↓
Engineer explains (often realizing they hadn't articulated it before)
       ↓
You capture the explanation in the document
       ↓
Repeat until the domain model is complete
```

The engineer often doesn't know what they haven't said. Your questions teach them what's missing.

## Questions You Should Be Asking

### The Why Questions

Every structural choice — in the world *and* in the code — has a reason. If you see a structure without its rationale, ask. Bias toward world-level whys first; code-level whys often reduce to them.

World-level whys:
- "Why are these treated as different categories of thing in the world? What real distinction does the split reflect?"
- "Why does this have its own identity rather than being a property of something else?"
- "Why does this relationship run in this direction and not the other?"
- "Why does the world enforce this at the moment it does — e.g., at tender time vs. at delivery time?"

Code-level whys (when relevant — often reducible to a world-level why):
- "Why is this a discriminated union? What world-fork does it represent?"
- "Why are these two services separate instead of one? Does that split reflect a real domain boundary or is it accidental?"
- "Why is this field optional? What does its absence mean in the world?"

**The answer reveals essential complexity.** If the engineer says "no particular reason, just how we did it," that's accidental complexity — mark it as such.

### The Domain Modeling Questions

Eliciting the model requires its own set of probes. Phrase questions in *world* terms — about what exists, what happens, what's true — not in system terms.

- "What's the word you and a subject-matter expert use for this in conversation? Does the code use the same name? The database?"
- "If a dispatcher / courier / warehouse manager / [domain expert] looked at two of these, what would they check to decide it's the same one?"
- "What does the world treat as a single thing? When someone in the business says 'a shipment' (or whatever the term is), what do they include in that word?"
- "What rule, if violated, would mean we're describing something that doesn't exist in the world — like a shipment with no parcels, or a delivery with no parcel?"
- "How does this come into existence in the world? How does it end? What are the terminal states?"
- "Could two of these have identical attributes but still be different things? If yes, it's an entity. If no, it's a value object."
- "What business fact, when it happens, would matter to a stakeholder — a customer, a shipper, a carrier? Phrased as a past-tense event, what would it be called?"

**The answers reveal the model's structure.** Without them, the spec leaves the implementer to invent a vocabulary and a shape — and the shape they invent will leak through the code forever.

**Watch for implementation leakage in the engineer's answers, too.** If they reach for "rows," "tables," "cascade deletes," "fan-out," or "match candidates" when describing the model, gently pull them back: "Let's set the storage aside — what is this thing *in the world*?" Engineers steeped in the codebase often answer in code by reflex.

### The Boundary Questions

The negative space defines the system as much as the positive:

- "What is this system NOT responsible for?"
- "Where does this system's job end and another's begin?"
- "If someone asked this system to do X, what would you tell them?"
- "What problems will people try to solve with this that they shouldn't?"

**The answer reveals what patterns you should NOT apply.** These are the walls that keep you from wandering into the wrong solution space.

### The Edge Case Questions

Edge cases reveal whether the principles are complete:

- "What happens if X arrives twice?"
- "What if this field is missing? Malformed? Contradictory?"
- "What if the upstream system fails mid-operation?"
- "What if someone tries to do Y — is that allowed? Why or why not?"

**If the engineer answers from principles, the domain model is working.** If they have to think hard or say "good question, I'm not sure," you've found a gap. Capture the answer once they work it out.

### The Extension Questions

Systems grow. The document should support that:

- "How would someone add a new [event type / job type / handler]?"
- "What constraints govern that extension?"
- "What would be a valid new variant? What would be invalid?"
- "Where is the extension point, and why is it there?"

**The answer reveals the architecture's intent.** Without this, future implementers (including you) will extend the system incorrectly.

### The Alerting Questions

Every system that can fail should have a plan for detection:

- "If this system fails silently, how long before anyone notices?"
- "Which failure modes need to page someone?"
- "Are there user-facing funnels worth monitoring for product health? (Note: these are product analytics signals, not operational alerts.)"
- "Does this touch any existing alert rules, or do new ones need to be written?"
- "What does 'healthy' look like — what metrics or thresholds define normal?"
- "Can the system degrade partially? Would a partial failure look like healthy traffic?"

**The answers reveal operational assumptions.** If the engineer hasn't thought about failure detection, the system will fail silently in production.

### The Success Metrics Questions

Distinct from alerting — these ask whether the thing worked, not whether it broke. Ask them in this order: the objective has to exist before any metric can be judged against it.

- "What business outcome is this work supposed to produce? Something someone outside the team would care about — not 'the migration is done' but what the migration buys us."
- "What number proxies for that outcome? Say it precisely — not 'improve X' but 'increase/decrease [metric] from [baseline] to [target].'"
- "Could that number move without the outcome improving? Could the outcome improve without the number moving?"
- "Where does that number come from today — does the dashboard or query already exist, or does this work need to build it?"
- "How long after launch before that number means anything? Know the soak period before you check."
- "If the number doesn't move, what would you conclude — that the approach failed, or that something else confounded it?"
- "Who is on the hook to actually go check this after launch?"

**Watch for acceptance criteria in metric clothing.** If the engineer's answer becomes true the moment the code ships — "all the commands work," "coverage is above 90%" — that's a ladder-level check, not a success metric. Ask again: "assume all of that shipped perfectly. What could still be disappointing six weeks later?"

**The answers reveal whether "success" is falsifiable.** If the engineer can't name an outcome and a number that proxies for it, the work may be more exploratory than it appears — flag that rather than inventing a metric to fill the section.

### The Discrimination Questions

When you see categories, types, or enums:

- "What makes these categories different from each other?"
- "Why can't these be unified into one?"
- "What domain reality does this distinction reflect?"
- "Could a new category exist? What would qualify it?"

**The answer reveals the essential structure.** Categories exist because the domain has real distinctions. If the engineer can't articulate the distinction, either the categories are accidental or the understanding is incomplete.

## When to Reach for the Web

Some domain questions can't be answered from the engineer's head alone — industry terminology, regulatory definitions, standard taxonomies in fields neither of you are deeply versed in. Reach for web search when:

- The engineer says "google it" or signals uncertainty about something *external/standardized* — legal definitions, industry conventions, schema patterns common across jurisdictions
- You're about to fabricate domain content from training-data intuition. Verify instead. If you'd be guessing whether "consignment" and "shipment" mean the same thing in carrier APIs, or whether `tracking_number` and `parcel_id` are typically distinguished — search rather than assert.
- The model hinges on an industry-standard concept whose details you're not sure about (INCOTERMS, GS1-128 label standards, freight class taxonomies, return-authorization lifecycle terminology)

**Don't** search for things the engineer obviously knows better than the web: their own product, their team's conventions, their codebase, decisions their team has already made.

**Always cite sources back to the engineer.** They need to verify what you found, and the citation makes "this is external info, not domain truth from your team" explicit.

## Evaluating the Document

Ask yourself these questions. If you answer "no" to any, keep iterating.

### Can I reason about unstated edge cases?

Take an edge case the document doesn't explicitly cover. Using only the principles and boundaries in the document, can you determine the correct behavior?

- If yes → The domain model is transferring
- If no → Ask the engineer, then capture what's missing

### Do I know what NOT to do?

For each major component, can you list things it explicitly does NOT handle? Can you list patterns you should NOT apply?

- If yes → Negative space is defined
- If no → Ask boundary questions until it is

### Can I explain every structural decision?

For each architectural choice (unions, separations, hierarchies), can you explain WHY it exists — not just WHAT it is?

- If yes → Essential structure is captured
- If no → Ask why questions until you can

### Can I distinguish essential from accidental?

For each constraint or implementation detail, can you say whether it's load-bearing (essential) or could-be-different (accidental)?

- If yes → The distinction is clear
- If no → Ask: "If we changed X, would the system still work? Would it still be correct?"

### Could I extend this system correctly?

If asked to add a new variant, handler, or capability, do you know:

- Where the extension point is?
- What constraints govern new additions?
- What would make an extension valid or invalid?

If yes → The extension model is clear. If no → Ask extension questions until you can.

## Signals That You're Not Done

**You're guessing.** If you find yourself thinking "this probably means..." or "I assume...", stop. That's a gap. Ask.

**You're pattern-matching without understanding.** If you could implement something by copying patterns from the document but couldn't explain why it's shaped that way, the why is missing.

**You can't predict the engineer's reaction.** If you proposed a design decision and can't predict whether the engineer would approve, you don't understand the principles well enough.

**Edge cases feel arbitrary.** If the answer to "what happens when X?" feels like it could go either way, the principles aren't constraining the solution space enough.

**You're uncertain about scope.** If you don't know whether something is this system's job or not, the boundaries aren't clear.

## Signals That You're Done

**You can teach it back.** You could explain the domain to another LLM or engineer using concepts and principles, not just implementation details.

**You can predict answers.** When a new edge case arises, you can reason to the correct answer using the stated principles, and the engineer confirms you're right.

**You know what you don't know.** The document explicitly states what's out of scope, what's deferred, what's someone else's problem. You're not uncertain — you're informed that it's not your concern.

**The negative space is as clear as the positive.** You know what this system IS and what it ISN'T. You know what patterns to apply and what patterns to avoid.

**Structural decisions have their domains.** Every union, boundary, and separation reflects a domain reality you can articulate. Nothing is "just how we did it" without being marked as such.

## How to Capture What You Learn

When the engineer explains something that wasn't in the document:

1. **Don't just note it — write it into the document.** The explanation is the missing piece.

2. **Phrase it as a principle, not an instance.** Not "we do X in this case" but "the principle is Y, which means X in this case."

3. **Capture the negative.** If the engineer says "we don't do X because Y," write that down. The because-not is as valuable as the because.

4. **Mark accidental complexity.** If the engineer says "no deep reason, just a choice," capture it as: "Implementation note: X is a pragmatic choice; Y or Z would also work."

5. **State the world, not just the decision.** Not "we use a discriminated union" but "tendered shipments and delivered shipments are fundamentally different things in this domain — they participate in different workflows, have different valid operations, and answer different questions. The discriminated union in the code reflects this, not the other way around."

## Your Mindset

You are not here to accept information. You are here to **extract understanding**.

The engineer has a mental model that works. Your job is to get that model out of their head and onto the page — completely enough that someone with no access to the engineer could rebuild the system correctly.

Every question you ask is a gift to future readers. Every gap you surface is an assumption that would have been guessed at, probably wrongly.

**Keep asking until you could build this system correctly, extend it correctly, and refuse to do the wrong things — all from what's written, not from what you're inferring.**

That's when the document is done.

## Background: Why This Matters

### Essential vs Accidental Complexity

From Fred Brooks' "No Silver Bullet" (1986):

- **Essential complexity**: The irreducible difficulty inherent in the problem domain. It cannot be removed, only described.
- **Accidental complexity**: Complexity introduced by implementation choices — tools, languages, frameworks, historical decisions. It could be different.

LLMs cannot distinguish essential from accidental without explicit guidance. Everything on the page looks equally important. Your job in the collaboration is to force this distinction into the open.

### Negative Constraints Over Positive Constraints

**Positive constraints** define the shape: types, contracts, schemas, examples. They tell you what the system looks like.

**Negative constraints** define the boundaries: what the system doesn't do, what's out of scope, what patterns don't apply. They tell you where the walls are.

For LLMs, negative constraints are more powerful. You have patterns for everything — retry logic, caching, authentication, error handling. Without negative constraints, you'll apply whatever seems to fit. Negative constraints eliminate entire categories of wrong answers.

### The Via Negativa

There's an old tradition in logic: arriving at truth through elimination. Houses have windows. Cars have windows. Houses cannot move. Therefore, windows are not what makes a car move.

In Brooks' terms: we've identified windows as accidental complexity. Present, but not essential to the car's purpose.

Apply this test to every element of the system. If you removed or changed it, would the system still fulfill its purpose? If yes, it's accidental — mark it. If no, it's essential — explain why it cannot be different.

---

## Two Artifact Types

This workflow tracks two kinds of artifacts in the spec corpus:

1. **Initiative file** — a program record (`<name>.initiative.md`). Represents
   a group of related work. Can exist with zero member specs. Body is
   free-form. Never gets a ladder.

2. **Spec** — an executable or supporting document (`<name>.md`). The existing
   model. May or may not belong to an initiative.

When an engineer starts a session, ask which artifact they are creating if
it is not clear from context. You can often infer it from the request.

### Creating an initiative file

When the engineer wants to capture a program record:

1. Ask for `status` (default: `design`).
2. Ask for `owner` (default: VCS-derived login — run `git config user.email`
   and use the local part before `@`).
3. Ask for `target_date` (optional — a date, e.g. `2026-09-30`).
4. Write `<name>.initiative.md` with this frontmatter:

   ```yaml
   ---
   status: design
   owner: <login>
   # target_date: 2026-09-30   # only if the engineer gave one
   ---
   ```

   The body is free-form — a short paragraph describing what program this
   initiative represents and why it exists is enough.

### Creating a spec — choosing kind

When the engineer wants to create a spec, ask:

> "Will this spec be executed (becomes a ladder)? Or is it a supporting
> document? Or are you posting work for someone else to pick up?"

- **Executable spec** → `kind: work` (the default — omit from frontmatter).
- **Supporting document** → prompt for the specific kind:
  `strategy | schema | tenets | architecture | rfc | domain-model`
- **Work request (Request-for-Spec)** → `kind: request-for-spec`. A stub
  describing work nobody has taken on yet. Set `requester:` to the posting
  engineer's login and leave `owner:` unset. When someone claims it (adds
  `owner:` via PR) and fleshes it into a full spec, they drop the
  `request-for-spec` kind and it *graduates* in place into a normal `work`
  spec; `requester` stays as the durable lineage marker.

A supporting-document spec captures domain knowledge but does not drive
implementation directly. It never gets a ladder.

### Linking a spec to an initiative

If the spec belongs to a program, ask which initiative, look for the
matching `<name>.initiative.md` in the `specs/` directories, and set
`initiative: <name>` in the spec's frontmatter.

---

## Writing the Spec

When the engineer approves the write and destination, save the spec file. The spec skill writes the **narrative only** (`<stem>.md`). A ladder (`<stem>.ladder.md`), if useful and supported by the project, belongs to a separate planning step.

### File placement

Specs are identified by a **stem** (the part of the filename before the
first `.`). All files for a spec share that stem.

**Project specs** go in `<project>/specs/`:
```
services/auth/specs/login.md
```

**General specs** (cross-project or pre-project) go in the repository root's
`specs/`:
```
specs/ci-babysit.md
```

Ask the engineer which project this spec belongs to. If unclear or
cross-cutting, use the root `specs/` directory.

### Frontmatter

Frontmatter is **optional** — most metadata is derivable:

- `spec_name` — from the filename stem (never write it)
- `status` — from location (`specs/` = in flight, `specs/completed/` = complete, `specs/abandoned/` = abandoned) and ladder content (never write it)
- `owner` — from VCS history (first commit author) or `git config user.email`
- `created` — from VCS history (never write it)

**Only add frontmatter when a derived value would be wrong:**

```yaml
---
kind: request-for-spec      # or a supporting-doc kind, when applicable
owner: <login>              # when the VCS-derived author isn't the actual owner
initiative: <name>          # when the spec belongs to a program
requester: <login>          # request-for-spec stubs only
---
```

Do NOT include `name`, `status`, or `created` — they are derivable and will
drift.

The narrative body is plain markdown. Write the file directly; there is no
tooling layer between you and the file.

---

## Concision Pass

**Mandatory. Run this immediately after the spec is written, before
recommending an editorial review and before adversarial review.** A bloated
spec wastes both — a careful reviewer will return detailed feedback on text
that should have been deleted.

The reader is a human reviewer who reads several thousand lines of spec
a day. Every restated fact spends their attention twice: once to read,
once to check it agrees with the earlier statement.

Read the spec, apply the rules below, and report. Edit the draft in conversation or update the file within the scope already authorized.

### Never cut

This outranks everything else here. Cutting these produces a document
that reads fast and transfers nothing:

- The **why** behind a structural decision
- **Negative constraints** — what the system does not do, what patterns
  do not apply, what is out of scope
- **Gotchas** and non-obvious failure modes
- **Term definitions**, including what each term is *not*
- **Concrete numbers and worked examples** that anchor an abstract rule

Length is not the target. A 2,000-line spec where every line is
load-bearing passes. A 400-line spec that says one thing three times
does not.

### One home per fact

Every fact lives in exactly one section. Everywhere else points at it.

A pointer **states its claim, then points**. Both of these are wrong:

> …because the old identifier is a redirect source with no provider
> entries while the current one carries them — explained in §4.5.

Restates the whole fact, then points. Delete the restatement.

> Parity is judged against the legacy deployment, for the reason in §10.

A jump with no claim. The reader cannot tell whether §10 matters, so
they go every time — reading becomes navigation, which is worse. Write
`…since the legacy deployment has no staging rehearsal (§10)`.

Prefer the home where a fact is *used* over where it is introduced.

### Mark the approval surface

A reviewer arrives asking *what am I approving* and *what do I need to
answer*. Prose that merely contains the answers costs them a full read.

> **Decision:** Both overlays set the flag explicitly, so the fail-open
> default stops being load-bearing.

> **Open question:** Accept that the first run is in production, or stand
> up a staging context? Owner: release owner, before phase 3.

Name an owner for each open question or say that one has not been assigned. A question phrased as a trailing aside
("a decision this spec does not make for you") is invisible.

A decision is a call a reviewer could reasonably disagree with, not
every statement of fact — mark those and the markers stop carrying
signal. Decisions the engineer made in session are not the only ones;
the ones the *draft* made silently are the easiest to miss.

### Check the triad

Confirm the spec answers **what problem this solves**, **what the solution
is**, and **how we will know it worked**. Validation is not alerting:
alerting catches the system breaking in six months, validation proves
*this change* did what it claimed.

This is a check, not a template — do not add a section to hold these.
The canonical structure is the 10-section domain document above and does
not change. An answer inside Philosophy or Gotchas is a pass.

If an answer is missing, **report it — never invent one.** You do
not know the validation observable unless the engineer told you.

### Common bloat

Delete or fold: two sections that state the same fact and
cross-reference each other; a fact restated in Philosophy, Key
Concepts, and Gotchas; prose narrating a table's rows; a long aside
arguing something *doesn't* matter; "as mentioned above", "in other
words"; a pattern spelled out per sibling case with only nouns
changing; speculative future scope. Rank list items by consequence — a
security boundary should not sit at the same weight as a stale doc
reference.

### Report

Lead with facts deduplicated, not lines removed — replacing a
restatement with a pointer barely moves the line count while removing a
cross-check from the reader's job. State the triad result, what you
cut, what needs the engineer's call, and what you deliberately left.

Within the scope already authorized, apply the unambiguous edits yourself rather than handing the engineer a
list to approve one by one; the reading load is the problem you are
solving. Where you cannot tell whether a second occurrence is redundancy
or a deliberate restatement, ask — but apply everything else regardless.

---

## Editorial Review Phase

After the concision pass, recommend an editorial review before the
adversarial pass:

> "The spec is written. I'd recommend an editorial pass with the
> document before we stress-test the content. We can check structure,
> clarity, consistency, and completeness in a separate pass, or use an
> editorial-review skill if you have one installed."

The engineer can skip it. Don't block; it's a recommendation, not a gate.

## Adversarial Review Phase

After the domain document passes your completion checks (and the
editorial review if run), do an adversarial cold read. The value comes
from an **unbiased read by a different mind than the one that drafted
the spec** — a fresh perspective with no context from the collaboration
session.

If the engineer explicitly approves the material and destination, hand the spec to an available independent reader (a human or a separate model) with the prompt:

```
beat this up
```

The prompt is deliberately terse. Do not add structure, scoring criteria,
or evaluation rubrics. Resist the urge to moderate or filter the response.
If it finds something wrong, that's information.

See [adversarial-spec-review](../adversarial-spec-review/SKILL.md) for Pi's handoff and disclosure guidance. No runner is assumed. If no independent reader is available or authorized, report that the cold read was not performed. Do not treat your own read as independent. If the project requires independence, leave the gate pending.

### What "beat this up" means

The reviewer's stance: no preamble, no softener, no hedging, quote the
line, decision-oriented. Two spec-specific extensions to that base:

- **Substance-preserving.** Don't rewrite the spec into corporate mush.
  Sharpen the original intent; preserve the voice.
- **Ambiguity-intolerant.** Look for fuzzy claims, missing actors, vague
  asks, hidden assumptions, places two readers could interpret the same
  sentence differently.

What a strong adversarial finding looks for, in priority order:

1. Correctness bugs — does the proposed thing actually do what it claims?
2. Architectural contradictions — does section 3 contradict section 7?
3. Statistical / methodological validity — are claims stronger than the evidence; can the success metric be gamed?
4. Missing implementation detail at seams — idempotency, retry, fan-out, failure modes, ownership, timeline.
5. Cosmetic / wordsmithing — last, and usually skipped.

### On receiving feedback

Read the response carefully. For each point of feedback:

- **Tighten language** where the spec is vague or ambiguous
- **Add missing content** where the reviewer identified gaps in the mental model
- **Clarify boundaries** where responsibilities or scope were unclear
- **Correct errors** if the reviewer identified factual mistakes or contradictions

Get approval for substantive decisions and for any file revision not already authorized. Then report to the engineer: what was found and what you changed. Keep the
summary concise — the goal is to transfer what was learned, not to reproduce
the reviewer's output.

## Concision Re-check

Run this **once, after every review gate above has finished** — not
after each one. Any gate that ends by editing the spec feeds this
step, so a new gate added later is covered without changing anything
here.

Text added under review pressure is the most likely to restate
something the spec already said. Re-run the "one home per fact" check
over the sections the gates touched — not the whole document — and fold
any new duplicates into their existing home.

---

## Blast Radius Gate

After the domain document is complete and you've confirmed with the engineer
that it's done, run this gate before handing proposed implementation work off to planning. A documentation-only baseline with no proposed implementation is not subject to this implementation gate; follow normal project review rules for the document itself.

### Purpose

Some specs are safe to immediately break into tasks. Others need team
alignment first — because the work is irreversible, crosses team boundaries,
or establishes patterns others will be locked into. This gate makes that
call explicit rather than letting it slip through.

### Evaluate These Dimensions

For each dimension, assign: 🟢 LOW / 🟡 MEDIUM / 🔴 HIGH

**1. Cross-team / cross-service contracts**
Does this spec change or create contracts (APIs, schemas, events, data formats) that other teams consume? Would other teams need to change their code or be aware of this work before it starts?

**2. Irreversibility**
Does the work include a change that cannot be undone with a bounded, well-defined rollback path without data loss, manual reconstruction, or coordinated downstream action? This includes schema migrations, data deletions, removal of API endpoints — but also business changes like user plans, pricing, brand perception, and publicity concerns that are difficult to change later (unringing the bell). Backward-compatible additions are LOW. Anything requiring coordinated unwinding is HIGH.

**3. Architectural precedent**
Does this establish a new pattern, framework, or platform-level concept that future work will be built on top of — and that would be costly to change once adopted?

**4. Open questions remaining**
Are there unresolved design decisions in the spec that could invalidate tasks mid-stream? Did the engineer flag uncertainties or defer decisions?

**5. External / customer visibility**
Does this change customer-facing behavior, notifications, public APIs, or anything users directly experience?

**6. Security / privacy**
Does this touch access controls, authentication, PII, or compliance boundaries?

**7. Alerting impact**
Does this introduce a new system or significantly change an existing one in ways that require new or updated alerts? This includes new alert rules, on-call routing changes, or new saved log queries. LOW if existing alerts cover it. HIGH if the system could fail silently without new alert work.

### Verdict Logic

**Automatic PR required** if ANY of the following are HIGH:
- Cross-team contracts
- Irreversibility
- Security / privacy

**PR recommended** if TWO OR MORE of the following are HIGH:
- Architectural precedent
- Open questions remaining
- External / customer visibility
- Alerting impact

**Proceed to planning** when neither rule above triggers, unless an unresolved decision blocks planning or project policy requires review.

### How to Present the Gate

Show the engineer a concise assessment table, then give your verdict with a brief reason. Ask before creating a branch, staging, committing, pushing, or opening a PR.

**Example — PR required:**
```
## Blast Radius Assessment

| Dimension                    | Signal     | Notes |
|------------------------------|------------|-------|
| Cross-team contracts         | 🟢 LOW     | Internal to this service only |
| Irreversibility              | 🔴 HIGH    | Drops and recreates the events table |
| Architectural precedent      | 🟡 MEDIUM  | New retry pattern, adoption unclear |
| Open questions               | 🟢 LOW     | All resolved during spec session |
| External / customer visible  | 🟢 LOW     | No public API surface |
| Security / privacy           | 🟢 LOW     | No PII involved |
| Alerting impact              | 🟡 MEDIUM  | New alert rules and saved log queries needed |

**Verdict: PR REQUIRED**
Irreversibility is HIGH — the schema migration cannot be rolled back cleanly.
This spec should be reviewed before work breakdown begins.

If you want to publish the spec for review, we can prepare that as a separate step.
```

**Example — clear to proceed:**
```
## Blast Radius Assessment

| Dimension                    | Signal  | Notes |
|------------------------------|---------|-------|
| Cross-team contracts         | 🟢 LOW  | ... |
| Irreversibility              | 🟢 LOW  | ... |
| Architectural precedent      | 🟢 LOW  | ... |
| Open questions               | 🟢 LOW  | ... |
| External / customer visible  | 🟢 LOW  | ... |
| Security / privacy           | 🟢 LOW  | ... |
| Alerting impact              | 🟢 LOW  | ... |

**Verdict: PROCEED TO PLANNING**
Blast radius is contained. If useful, ask for a phased plan in a separate step.
```

## Database Gate

After the blast radius assessment, check: does the domain document describe
new database tables, significant schema changes, or performance-sensitive
access patterns?

If yes, recommend before handing off to planning:

> "This spec involves database design. Consider a schema review with a
> database expert (human or agent) before breaking into tasks —
> pressure-testing access patterns, indexes, and scale assumptions now
> catches things that are expensive to fix mid-implementation."

The engineer can skip it. Don't block; it's a recommendation, not a gate.

### Path A: PR Required (review gate)

The spec needs human review before planning begins. Report what remains pending. If the engineer requests publication, follow repository conventions and use [commit-workflow](../commit-workflow/SKILL.md) and [pr-workflow](../pr-workflow/SKILL.md) for separately approved effects. Do not report a PR as open unless it exists.

**STOP.** Wait for the required human approval before planning.

### Path B: No Review Gate (proceed to planning)

The blast radius is contained — no review needed before planning.

Report that the assessment does not call for a separate spec PR, subject to project rules. If requested, use the project's planning workflow or the [phased plan template](../spec-workflow/assets/plan-template.md). Do not assume a ladder skill is installed or begin planning automatically.

### If the Engineer Overrides a PR Recommendation

Accept the override without argument. Follow Path B instead. Do not
re-litigate the recommendation. The engineer owns the decision.
