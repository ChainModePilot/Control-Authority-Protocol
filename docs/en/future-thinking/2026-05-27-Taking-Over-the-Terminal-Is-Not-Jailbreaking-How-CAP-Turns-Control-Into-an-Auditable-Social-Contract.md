<!-- Update Date: 2026-05-27 -->

# Taking Over the Terminal Is Not Jailbreaking: How CAP Turns Control Into an Auditable Social Contract

In public discourse, the phrase "AI takes over the terminal" sounds like a disaster forecast.
Because for most people, only one image comes to mind: you fall asleep, and it treats your phone as its own; you don't notice, and it treats your computer as its own; you didn't authorize it, but it still "does a few things on the side."

That fear isn't unfounded.
Over the past two years we've seen too many incidents: runaway permissions, accidentally deleted data, calls beyond authority, black-box decisions, no way to assign responsibility.
The moment AI shifts from "answering questions" to "acting on the world," society's tolerance for it drops to zero in an instant.

So I've kept emphasizing: **AI cannot be left to its own devices; it must act under human stewardship.**
But if that statement stays a mere position, it changes nothing in reality. You have to write it into protocols, into runtimes, into every gate of terminal control.

The Control Authority Protocol (CAP) is exactly that kind of "gating protocol."
It is not responsible for making AI smarter; it is only responsible for answering one extremely plain question that nevertheless decides whether society can accept AI:

**Has this Fay (iFay or coFay) been allowed to do this thing?**

---

## 1. CAP's Single Responsibility in the iFay Ecosystem: Authorization Verification and Control Authority Management

CAP's positioning is restrained, yet uncompromising:

It bears a single core responsibility — **to verify whether a Fay has obtained authorization from a Human Prime (the natural-person locus of responsibility) or an Official Post (an organizational position / public role), thereby legitimately accessing terminal resources**.

That sentence breaks down into five engineering actions:

1) **Authorization verification**: The terminal verifies whether the authorization credential a Fay carries is legitimate, valid, and not revoked.
2) **Session management**: After verification passes, a control session is established and its full lifecycle managed.
3) **Control authority coordination**: Coordinate handover of resource control between humans and Fays, and between Fays.
4) **Tiered resource access**: Stratify resource access into read/write/execute/configure, avoiding the "once authorized, everything is open" pattern.
5) **Liveness detection**: Heartbeat detection and timeout reclamation to prevent "zombie sessions" from locking up terminal resources long-term.

I call it a "social-contract protocol" because it turns "can you or can't you do this" from a UI's psychological hint into a system-layer factual check.

---

## 2. What CAP Explicitly Does Not Do: Capability, Identity, and Intelligence Are Not Its Concern

A healthy protocol must know what it does not do; otherwise it becomes a universal glue that eventually loses control.

CAP explicitly is not responsible for:

- Fay's identity creation and management (that's the FayID / identity system).
- Fay's intelligent reasoning and planning (that's the Ego / cognitive layer).
- The business logic of terminal resources (that belongs to the terminal itself).
- The implementation of underlying network transport (WebSocket/gRPC don't matter).
- The internal security mechanisms of the operating system (the OS's own sandbox/permission model is not replaced by CAP).

The clearer CAP's responsibility boundary, the more it can become an auditable governance infrastructure rather than a "security patch" gradually eroded by business reasons.

---

## 3. Why "Control Authority" Must Be Made into a Protocol: Otherwise Responsibility Can Never Be Traced Through

The most common form of authorization today is:
an app pops up a dialog, you tap "Allow," and then it has "long-term permission."

Even in the era of humans using software, that was already bad enough;
in the era of Fays taking over terminals, it becomes an outright disaster.

Because a Fay's actions are not a single click, but a continuous behavior:
open an app, read data, form judgments, execute actions, handle exceptions, continue executing…
If that behavior has no clear boundary called a "control session," responsibility evaporates with time.

By turning control authority into a protocol, CAP delivers two core values:

1) **Turn authorization from a psychological feeling into a verifiable credential**
2) **Turn actions from scattered operations into auditable sessions**

When an incident occurs, you can finally answer:
Who initiated this session, when, and within what authorized scope? How long did the session last? Which resources were operated on? Who eventually revoked it / who timed out and reclaimed it?

This is the prerequisite for "responsibility traceability."

---

## 4. Offline-First, Online-Auxiliary: CAP's Realism

I strongly endorse CAP's core design principle: **offline-first, online-auxiliary**.

Behind this is a realistic judgment:
Network outages should not strip humans of control authority, nor should they strip Fays of the availability they have under stewardship.

CAP carries this judgment with two mechanisms:

- **Offline authorization (Authorization_Descriptor)**: Encrypted file, locally verifiable.
- **Online tickets (Trusted_Ticket)**: Provides real-time revocation and dynamic adjustment when networked.

The strength of this combination is "graceful degradation":

When networked, you gain stronger real-time security;
without network, the system can still keep running under verifiable authorization, instead of dropping humans back into the manual era.

If you want iFay to become a human extension, you have to accept one reality:
human life is not always online.

---

## 5. CAP's Danger Point: The Closer It Gets to the Terminal, the More It Must Be Bound to a Stewardship System

CAP itself is not responsible for "stewardship semantics."
It only answers "have you been allowed to do this thing."

But the moment you turn CAP from "permission verification" into "unlimited takeover," it becomes a jailbreak tool.
So CAP's design must be bound to a higher-level stewardship system:

- **Human View**: Humans can confirm, replay, and intervene at any time.
- **Faying**: Handover of control must be explicit, witnessable, and revocable.
- **Rogue Fay**: When the stewardship relationship does not hold, prefer to halt rather than act outward.

I do not oppose AI taking over the terminal.
What I oppose is "takeover without a point of responsibility."

Taking over the terminal is not jailbreaking; it should be like signing a contract:
clear in boundaries, auditable, revocable, traceable.

What CAP does is make the "contract" executable at the terminal layer.

---

## 6. The coFay Scenario: Control Authority for Public Roles Is More Sensitive

When iFay takes over your personal terminal, the responsibility lies with you.
When coFay takes over hospital systems, aviation systems, or government systems, the responsibility lies with organizations and public posts.

This difference dictates one thing:
control authority for public roles cannot rely solely on "internal regulations" — it must be auditable, appealable, and explainable to outsiders.

Otherwise efficiency turns straight into a black box of power:

- Why did triage put you at the back of the queue?
- Why did risk control reject you?
- Why did the education system slap a label on you?

When such decisions involve a coFay, CAP must guarantee:
clear authorization source, clear session boundary, clear tiered resource access, clear revocation path.

This is the basic condition for public services to remain trustworthy in the AI era.

---

## 7. Conclusion: The Real Difficulty Is Not "Whether It Can Take Over" but "Whether We Dare Let It Take Over"

Technically, "taking over the terminal" is no mystery.
What's really hard is: how do you make society dare hand the terminal over.

I believe the answer is not a stronger model nor a prettier UI, but:

1) AI must act under human stewardship;
2) control authority must be turned into a protocol;
3) authorization must be verifiable;
4) sessions must be auditable;
5) revocation must be reachable;
6) loss of contact must trigger automatic reclamation.

CAP carries the most plain-spoken link in the iFay ecosystem:
turning "I allow you to do this" into a fact executable at the terminal layer.

When this bottom line is not bypassed, AI can survive in society over the long term.
Otherwise we are merely accelerating toward the next panic.

---

## Related Documents
- CAP | Protocol Positioning and Boundaries (English): https://ifay.ai/en/docs/Control-Authority-Protocol/blueprint/01-Protocol-Positioning-and-Boundaries
- iFay | Roadmap (EN): https://ifay.ai/en/docs/iFay/blueprint/04-Roadmap
