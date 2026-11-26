
.. _chapter1:

=======================
Power of Attorney (PoA)
=======================

A Power of Attorney (PoA) is a **human-approved, scoped, time-limited, revocable authorization** that lets an AI agent perform an action it otherwise could not perform on its own. Its purpose is to answer one governance question: *on whose authority is this agent acting?*

Purpose
-------

Some actions are too sensitive for an agent to run unsupervised. A PoA lets a person pre-approve a specific class of those actions within fixed limits. Without a valid PoA, the guarded action is blocked. Other actions the agent can take are unaffected.

A PoA ties every agent action back to an explicit human intent. This supports:

* **Accountability:** each action can be traced to the person who granted the authority.
* **Least privilege:** the agent receives only the authority it needs, for the time it needs it.
* **Control:** the person can withdraw the authority at any time.

Core concepts
-------------

.. list-table::
    :header-rows: 1
    :widths: 25 75

    * - Concept
      - Meaning
    * - **Principal**
      - The party whose authority is being used. Usually a human, and sometimes an agent acting for a human.
    * - **Delegator / grantor**
      - The party that grants authority, for example the human who approves a PoA.
    * - **Delegate / agent**
      - The party that receives authority and acts on it.
    * - **PoA type**
      - A reusable template that defines one kind of guarded action and its allowed value ranges.
    * - **PoA instance**
      - A specific grant, issued from a type, with concrete values, an expiry, and a status.
    * - **Constraint**
      - A limit on what the grant allows: maximum or minimum values, allowed formats or patterns, required identifiers, time windows, or frequency.
    * - **Intent / purpose**
      - The declared reason for an action. It is checked against what the grant allows.
    * - **Presentation**
      - The short-lived proof an agent sends with a single action, derived from the long-lived grant.

The PoA flow
------------

End to end, a guarded action flows like this::

    +----------+      +----------------------------+      +----------------+      +---------------------+      +------------------+
    |  Person  |      |            PoA             |      |     Agent      |      |  Enforcement point  |      |  API/MCP/Tool/   |
    | approves | ---> | (scopes, limits, expiry)   | ---> | guarded action | ---> |   Checks the PoA    | ---> |  Other Agent     |
    +----------+      +----------------------------+      +----------------+      +----------+----------+      +------------------+
                                                                                             |
                                                                                       (if invalid)
                                                                                             |
                                                                                             v
                                                                                  +---------------------+
                                                                                  |       Blocked       |
                                                                                  |     If invalid      |
                                                                                  +---------------------+

How it works
------------

#. **Discovery.** The agent looks for an active PoA that covers the task. If none exists, it looks for a matching PoA type it can request.
#. **Request.** The agent drafts a PoA with the proposed values. The draft is stored in a pending state and does not yet grant anything.
#. **Approval.** The agent gives the user a one-time signing link. The user authenticates through their identity provider, reviews the limits in plain language, and authorizes or rejects the request. Only the addressed user can approve.
#. **Issuance.** When the user approves, the system creates a cryptographically signed PoA. Issuance-time policy rules are evaluated at this point, and the request can still be refused.
#. **Presentation.** For each guarded action, the agent presents a short-lived token derived from the PoA. The long-lived PoA itself is never sent to the tool or service.
#. **Verification.** The gateway in front of the tool checks the token in three steps:

   #. **Cryptographic check:** the signature is valid, the token has not expired, the chain back to the human root is intact, and the token has not been revoked.
   #. **Policy decision:** the action is evaluated against the PoA's constraints, the declared intent, the runtime context, and the organization's business policies.
   #. **Forwarding:** only if both checks pass is the request forwarded to the tool.

#. **Reuse.** Later actions that fall within the PoA's limits reuse it without a new approval. Actions outside those limits, or made by a different agent or user, require a new request.
#. **Expiry or revocation.** When the PoA expires or is revoked, covered actions are blocked and a new approval is required.

**Triggers.** A new PoA can be started in two ways:

* **Proactive:** an administrator or the user creates the request in advance, and the user signs it when ready.
* **Reactive:** the agent meets a tool or service that requires authority it does not hold, and the response directs it to the approval process.

Both paths lead to the same approval and issuance steps. Discovery is simplest when the agent is configured in advance with the PoA types it may request. Reactive discovery needs a standard challenge the tool can issue and the agent can interpret.

Delegation and attenuation
--------------------------

* **Attenuation only.** A delegation can narrow authority but never widen it. A child PoA must be within the scope, amount limits, and expiry of its parent. In practice this means:

  * numeric limits in the child must be equal to or lower than the parent's;
  * list-valued scopes in the child must be subsets of the parent's;
  * the child cannot introduce fields or permissions the parent did not grant;
  * the child's expiry cannot be later than the parent's.

* **Chains.** An agent can sub-delegate part of an active PoA to another agent. Each link in the chain is verified, so a presented chain is valid only if every link is valid.
* **Machine delegation.** An authorized parent agent can issue a sub-delegation without a new human approval, provided the parent permits it and the child stays within bounds.
* **Re-delegation control.** The parent can forbid further re-delegation, and the depth of chains can be limited.

Revocation
----------

* **Status-based.** Revocation sets a status on the PoA in the operational store.
* **Checked on every use.** Each verification re-checks revocation for every link in the chain. Revoking a parent therefore invalidates its entire sub-tree without a separate cascade step.
* **Fail closed.** If revocation status cannot be determined, the action is refused.
* **Propagation target.** Revocation should reach all enforcement points quickly, typically within seconds. An event-driven cache can meet this target; a direct status lookup is simpler but depends on the freshness of the store.

Trust and key custody
---------------------

* **Signing keys.** Agents sign with keys they hold. Human users usually do not hold a signing key of their own, so the system signs the human's root authority on their behalf, using a key that is custodied for that user and used only after the user approves through an authenticated flow.
* **Key separation.** Key custody is separate from the identity provider's user database.
* **Algorithm agility.** The signature scheme should allow migration to a different algorithm, including post-quantum algorithms, without reissuing stored PoAs.

Architecture (generic)
----------------------

.. list-table::
    :header-rows: 1
    :widths: 25 75

    * - Component
      - Responsibility
    * - **PoA service**
      - Drafts, issues, stores, and manages PoAs. Performs sub-delegation and revocation.
    * - **Issuance policy engine (PDP)**
      - Decides whether a given user may request or approve a given PoA type. Evaluates conditions such as geography or step-up approval.
    * - **Verifier**
      - Performs the technical checks only: signature, expiry, chain, scope match, and revocation. Returns valid or invalid. It does not make allow or deny decisions. It can be stateless.
    * - **Authorization server (optional)**
      - Can issue separate short-lived access tokens to agents. Many designs skip it and let the agent derive presentations directly from the PoA. Whether to add it is a design choice.
    * - **Runtime policy engine (PDP)**
      - Makes allow or deny decisions for the action, using intent, constraints, and business rules.
    * - **Gateway (PEP)**
      - Intercepts each tool call, invokes verification and the policy decision, and forwards only permitted calls.
    * - **Control-plane facade**
      - Exposes PoA management operations to agents, for example listing templates, requesting, delegating, and revoking. It holds no authority itself.
    * - **Operational store**
      - Holds active PoAs, templates, links in the delegation chain, and revocation records.
    * - **Audit log**
      - Append-only, tamper-evident record of issuance, presentation, verification, policy decisions, and revocation.

Security properties
-------------------

* **Fail closed.** Any invalid, expired, misaligned, revoked, or out-of-scope link results in denial.
* **No unbounded authority.** Agents never hold a standing, unlimited grant.
* **Human intent is explicit.** Every grant traces to a declared purpose and a human approval.
* **Traceability.** Each action carries an identifier that links it to the grant, its approval, and the request that caused it. The identifier must be propagated through every hop in a chain.
* **Injection resistance.** Prompt-level instructions cannot skip approval. The approval step runs outside the agent's control.
* **Confused-deputy resistance.** A PoA is bound to a specific agent and user, so a different agent or user cannot reuse it.

Summary
-------

A PoA is a revocable, bounded, human-granted permission for an agent to perform a specific kind of guarded action. Authority is carried in a signed token that can only narrow as it is delegated. Each action is checked at call time for validity, scope, intent, and policy. Any failed check denies it.
