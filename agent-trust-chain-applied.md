# The Agent Trust Chain: Applied

> Two scenarios. Same five layers. Very different starting points.

## What This Is

A companion to [The Agent Trust Chain](agent-trust-chain.md). That document defines the framework: five layers, principles, maturity indicators. This document shows what those layers look like in practice for two common profiles:

1. **Profile A: Mid-size enterprise** in the Microsoft ecosystem, hybrid cloud, existing IT governance, compliance obligations
2. **Profile B: Small business** with 15 to 40 staff, no IT department, ad hoc tooling, "it works so don't touch it" culture

These are composites, not specific organisations. But if you have worked in this space, you will recognise them immediately.

---

## Profile A: The Microsoft Enterprise

### Who They Are

- 200 to 2,000 staff across multiple offices
- Microsoft 365 E3/E5 licensing, Azure AD (Entra ID), SharePoint, Teams, Dynamics or similar LOB apps
- Hybrid infrastructure: on-premises Active Directory synced to Entra ID, some Azure VMs, maybe a legacy server room
- IT team of 5 to 20 people, one or two with security focus
- Compliance obligations: industry-specific (finance, health, government) or contractual (SOC 2, ISO 27001, client audit clauses)
- They have heard about "AI agents" from Microsoft Copilot marketing, from vendors, from their board. Some staff are already using ChatGPT with corporate data. Nobody is sure if that is a problem yet.

### What They Want to Deploy

They are considering or have already started:
- **Microsoft 365 Copilot** across the organisation
- **Custom Copilot agents** in SharePoint or Teams for internal knowledge queries
- **Power Automate flows** with AI Builder actions that read/write business data
- **Azure OpenAI Service** integrated into a LOB application (e.g., customer service triage)
- An ambitious IT lead has proposed an **autonomous agent** that monitors their Azure environment and remediates common issues

### Current State (Before the Framework)

If you walked in and ran the diagnostic today, here is what you would likely find:

```
Layer 0 - Containment:    ████░░░░░░ Basic
Layer 1 - Identity:       ████████░░ Intermediate
Layer 2 - Authorization:  ████░░░░░░ Basic
Layer 3 - Observability:  ██████░░░░ Basic/Intermediate
Layer 4 - Governance:     ██░░░░░░░░ None/Basic
```

**Why this shape:**
- **Identity is their strongest layer** because Microsoft does identity well. Entra ID, managed identities, service principals: the plumbing exists. They are already using it for humans and services.
- **Containment is basic** because their agents inherit whatever access the deploying user or service principal has. Nobody has asked "what is the blast radius?"
- **Authorization is basic** because Copilot permissions follow SharePoint permissions, which are a mess in most orgs (oversharing is endemic). Custom agents use service principals with broad scope because "it was easier to get it working."
- **Observability looks okay** because Microsoft has extensive logging (Unified Audit Log, Azure Monitor, Purview). But nobody is looking at the logs for agent-specific behaviour. They would spot a human doing something unusual; they would not spot an agent doing something unusual.
- **Governance is nearly absent.** Copilot was turned on. Nobody defined what happens when it does something wrong, how to expand or restrict its access over time, or who is responsible.

### What Each Layer Looks Like: Current vs Target

#### Layer 0: Containment

**Current state:**
- Copilot runs with the user's full M365 permissions. If a user can access the CEO's calendar, so can their Copilot.
- Custom agents use a service principal with `Sites.ReadWrite.All` because the developer needed access to multiple sites and this was the fastest way.
- The Azure remediation agent prototype runs with Contributor role on the subscription. It could delete any resource.
- No cost caps. The Azure OpenAI deployment has no spending limit configured.

**Target state:**
- Copilot access gated by a SharePoint permissions audit. Oversharing cleaned up *before* Copilot is enabled, not after. Sensitivity labels applied to confidential content so Copilot cannot surface it in responses to users who should not see it.
- Custom agents use application permissions scoped to specific sites, not tenant-wide. Each agent has a dedicated service principal with minimum viable permissions.
- Azure remediation agent runs in a sandbox subscription first. Production access limited to specific resource groups with explicit action allowlists (restart VM: yes, delete VM: no). Spending alerts at $50, hard cap at $200.
- Human-in-the-loop gate on any destructive action in production.

**How to get there:**
1. Run a SharePoint permissions audit using Microsoft Purview or a third-party tool (ShareGate, AvePoint). Fix oversharing before Copilot rollout.
2. Apply sensitivity labels via Microsoft Purview Information Protection. Label documents that contain confidential data. Copilot respects these labels.
3. Replace the `Sites.ReadWrite.All` service principal with site-scoped permissions using `Sites.Selected` in Microsoft Graph.
4. Create an isolated Azure resource group for the remediation agent. Use Azure RBAC custom roles that allow only specific actions (e.g., `Microsoft.Compute/virtualMachines/restart/action` but not `Microsoft.Compute/virtualMachines/delete`).
5. Set Azure cost alerts and budget caps in Azure Cost Management.

#### Layer 1: Identity

**Current state:**
- Copilot actions attributed to the user (reasonable, as it acts on behalf of the user).
- Custom agents use a shared service principal. Two different agents use the same credential because "we only had one registered."
- Azure remediation agent runs under a generic `automation@company.com` service account with no agent-specific identity.
- No delegation chain. When an agent acts, the audit log shows the service principal, not which agent logic triggered the action or which human authorised it.

**Target state:**
- Each agent (or agent class) has its own Entra ID application registration with a descriptive name and documented owner.
- Managed identities for Azure-hosted agents (no shared secrets).
- Delegation chain expressed via custom claims in the token or via structured log enrichment. Audit trail shows: this service principal, running this agent logic, triggered by this human request or automation rule.
- Agent registry document listing all active agents, their identities, their human owners, and their permission scope.

**How to get there:**
1. Register separate Entra ID applications for each agent. Use naming convention: `agent-<function>-<environment>` (e.g., `agent-kb-query-prod`, `agent-azure-remediation-dev`).
2. Use managed identities for anything running in Azure. No client secrets for compute workloads.
3. Add custom claims to application tokens (using claims mapping policies or application-level optional claims) to carry agent metadata.
4. Create an agent registry: a simple SharePoint list or internal wiki page listing every agent identity, its purpose, its owner, and its last review date.

#### Layer 2: Authorization

**Current state:**
- Copilot: follows user's existing M365 permissions (good in principle, but those permissions are too broad; see Layer 0).
- Custom SharePoint agent: `Sites.ReadWrite.All`, which means it can read and write every SharePoint site in the tenant.
- Power Automate flows: run as the connection owner's identity with their full permissions. A flow that reads one list has implicit access to everything the user can access.
- Azure remediation agent: Contributor role, which means it can create, modify, and delete any resource in the subscription.
- No composition analysis. Nobody has asked: "The KB agent can read HR documents and respond to any user in Teams. Can it be social-engineered into revealing salary data to someone who asks the right question?"

**Target state:**
- Copilot: permissions cleaned up at the SharePoint level (Layer 0 work). Sensitivity labels prevent access to classified content even when permissions are broad.
- Custom agents: minimum viable Graph permissions, scoped to specific resources. `Sites.Selected` instead of `Sites.ReadWrite.All`. `Mail.Read` on a specific mailbox instead of `Mail.Read` tenant-wide.
- Power Automate: service principal connections where possible (breaking the tie to individual user permissions). Flows reviewed for implicit access scope.
- Azure remediation: custom RBAC roles with explicit action lists. Deny assignments for destructive actions.
- Composition analysis documented: for each agent, "given its permissions, what is the worst combination of actions it could take?" reviewed at deployment and quarterly.

**How to get there:**
1. Audit current Graph API permissions for all application registrations: `Get-MgServicePrincipal | Get-MgServicePrincipalAppRoleAssignment`. Remediate overbroad grants.
2. Migrate from delegated permissions (acting as a user) to application permissions with granular scopes where appropriate.
3. For the KB agent: implement a response filter that checks whether the requesting user has access to the source documents before including them in a response. Do not trust the agent to make this decision; enforce it in the application layer.
4. Document the composition risk for each agent in the agent registry.
5. Implement Conditional Access policies for service principals (available in Entra ID P1/P2) to restrict where and when agent identities can authenticate.

#### Layer 3: Observability

**Current state:**
- Microsoft Unified Audit Log captures M365 actions. It is on by default. Nobody reviews it for agent-specific patterns.
- Azure Activity Log captures resource operations. Same story: it exists, nobody is watching it for agent behaviour.
- Copilot interactions logged in M365 audit log (search for `CopilotInteraction` events). Most orgs do not know this exists.
- Custom agent actions logged at the application level, inconsistently. Some log everything, some log nothing. No standard format.
- No reasoning trace. When the KB agent gives a wrong answer, nobody can reconstruct *why* it chose that answer.
- No outcome verification. The Azure remediation agent reports "fixed" but nobody checks.

**Target state:**
- Agent-specific audit log queries or alerts in Microsoft Sentinel / Defender for Cloud.
- `CopilotInteraction` events monitored for anomalies (unusual volume, access to sensitive content, off-hours usage).
- Custom agents log to a centralised system (Azure Monitor / Log Analytics / Application Insights) in a structured format that includes: action, target resource, triggering event, agent identity, result, and token/cost metrics.
- Reasoning traces stored for custom agents: the prompt, the retrieved context, the model response, and the final action taken. Not for every interaction (cost-prohibitive), but for actions that modify data or have compliance implications.
- Outcome verification for the remediation agent: after "restarting VM," check that the VM is actually running. Log the verification result alongside the action.

**How to get there:**
1. Enable Microsoft Purview Audit (Premium) if not already active. Set up retention policies for agent-related events.
2. Create Sentinel analytics rules for agent-specific patterns: unusual volume of `CopilotInteraction` events, service principal authentications from unexpected locations, agent actions outside business hours.
3. For custom agents: adopt a structured logging standard. At minimum: `{ timestamp, agent_id, action, target, trigger, result, token_count, cost }`. Ship to Log Analytics.
4. For the remediation agent: add post-action verification steps (e.g., after restart, poll VM status for 60 seconds and log the result).
5. For compliance-sensitive agents: log the full prompt/response chain (encrypted at rest) with a retention policy that matches your compliance obligations.

#### Layer 4: Governance

**Current state:**
- Copilot was enabled tenant-wide in a single change. No phased rollout, no review schedule, no defined owner.
- Custom agent permissions set at deployment. No review cadence. No process to expand or restrict.
- No escalation protocol. When the KB agent gives a wrong answer, users email IT. There is no defined triage path.
- Revocation capability exists (disable the application in Entra ID) but has never been tested. Nobody knows how long it takes to propagate.
- No trust evolution. The remediation agent has the same permissions today as on day one, regardless of its track record.

**Target state:**
- **Defined ownership:** Every agent has a named human owner in the agent registry. Owner reviews permissions and behaviour quarterly.
- **Phased rollout:** New agents deployed to a pilot group first. Promotion to broader access based on defined success criteria (e.g., 30 days, no incidents, user satisfaction above threshold).
- **Escalation protocol:** When an agent does something wrong, there is a defined triage path, incident documentation, root cause analysis. Not different from a service incident; just needs to include agent-specific context (what was the prompt, what did it retrieve, why did it act that way).
- **Revocation tested:** Quarterly drill to disable an agent's application registration, verify that all dependent systems stop accepting its tokens within the expected timeframe (Entra ID tokens can be valid for up to 1 hour by default; configure Continuous Access Evaluation to reduce this).
- **Automated narrowing:** If the remediation agent fails an outcome verification, its permissions are automatically restricted until a human reviews. If Sentinel detects anomalous agent behaviour, the service principal is automatically blocked via Conditional Access.
- **Trust promotion:** The remediation agent starts with read-only + restart in dev. After 30 days clean, promoted to restart in production. After 90 days, considered for additional actions (scale operations, configuration changes). Each promotion documented and approved.

**How to get there:**
1. Add governance fields to the agent registry: owner, deployment date, last review date, current trust level, next review date.
2. Write an agent deployment checklist that mirrors existing change management but adds agent-specific items (blast radius documented, composition analysis done, observability configured).
3. Configure Continuous Access Evaluation (CAE) in Entra ID for agent service principals to reduce token validity windows.
4. Create a Sentinel playbook (Logic App) that automatically disables a service principal when specific alert conditions are met.
5. Define trust levels (e.g., Dev, Pilot, Production-Limited, Production-Full) and the criteria for moving between them.

---

## Profile B: The Small Business

### Who They Are

- 15 to 40 staff. Maybe a single office, maybe hybrid/remote.
- No IT department. One person who is "good with computers" handles everything, or they have an MSP on a monthly retainer who mostly deals with printer issues and password resets.
- Infrastructure is a mix of: Microsoft 365 Business Basic/Standard (email and OneDrive), a NAS somewhere, maybe a legacy on-prem server running line-of-business software nobody fully understands, consumer-grade networking (ISP-supplied router, no firewall), a shared Google Drive from before they migrated to M365 that still has critical files in it.
- No documented IT policies. No change management. No incident process. "IT governance" would get a blank look.
- Security awareness: the owner knows phishing is bad. MFA might be enabled on some accounts. Passwords might be on a sticky note. The Wi-Fi password is the business name followed by the year it was set up.
- They are interested in AI because they have seen demos, a competitor mentioned it, or a staff member started using ChatGPT for customer emails and it is actually pretty good.

### What They Want to Deploy

They are not thinking in terms of "agent deployment." They are thinking:
- "Can AI answer our customer enquiries so Sarah does not have to?"
- "Can it read our documents and help new staff find things?"
- "Can it do our bookkeeping / invoicing / scheduling?"
- "My competitor has a chatbot on their website. I want one too."

The reality is they are about to deploy agents; they just do not call them that. The chatbot on the website that can look up order status? That is an agent with read access to their order system. The AI that answers customer emails? That is an agent with write access to their email and customer data.

### Current State (Before the Framework)

```
Layer 0 - Containment:    ░░░░░░░░░░ None
Layer 1 - Identity:       ░░░░░░░░░░ None
Layer 2 - Authorization:  ░░░░░░░░░░ None
Layer 3 - Observability:  ░░░░░░░░░░ None
Layer 4 - Governance:     ░░░░░░░░░░ None
```

This is not a criticism. It is the starting point. Every small business is here until someone shows them the risk and the path forward.

### The Real Risk

The danger is not sophisticated attacks. It is mundane things at business-ending scale:

- The chatbot is connected to the CRM via an API key with full read/write access. A prompt injection in a customer message causes it to dump the customer database into a chat response. **Privacy breach. Mandatory notification. Possible fine.**
- The email AI is connected via OAuth with full mailbox access. It auto-replies to a phishing email with internal information because it was trying to be helpful. **Data leak.**
- The bookkeeping AI has access to the accounting system. It miscategorises transactions for three months before anyone notices. **Financial reporting errors. Tax implications.**
- A staff member connects ChatGPT to the company SharePoint using a third-party integration they found on Google. Corporate data is now flowing through an unvetted third party's servers. **Shadow AI. Data sovereignty violation.**

None of these require a sophisticated attacker. They require an AI doing exactly what it was set up to do, in a way nobody anticipated.

### What Each Layer Looks Like: Practical and Proportionate

The goal for a small business is not "Advanced" on every layer. It is "enough to prevent business-ending mistakes." Proportionate controls, minimal overhead, maximum protection per dollar.

#### Layer 0: Containment

**What it looks like for a small business:**

You are not building network segmentation or Kubernetes resource limits. You are applying common sense at scale:

1. **Separate accounts.** The AI does not use Sarah's login. It has its own account with its own permissions. If Sarah leaves, the AI keeps working. If the AI is compromised, Sarah's account is fine.
2. **Read-only where possible.** The website chatbot needs to *look up* order status, not *change* orders. Give it read-only access to the order system. This single decision eliminates an entire class of disasters.
3. **No email send capability unless essential.** If the AI is answering enquiries, it drafts responses for a human to review and send. It does not send autonomously. This is a human-in-the-loop gate that costs nothing.
4. **Financial limits.** If the AI is using a paid API (OpenAI, etc.), set a monthly spending cap. $50, $100, whatever is appropriate. An uncapped API key is an open chequebook.
5. **Test with fake data first.** Before connecting the AI to real customer data, run it against test data for a week. See what it does. This is the small-business version of a sandbox environment.

**Practical actions:**
- Create a dedicated M365 account for each AI tool (e.g., `ai-chatbot@company.com`)
- Set API spending caps on every AI service
- Default to read-only access; justify every write permission
- AI drafts, human sends (for any external communication)

#### Layer 1: Identity

**What it looks like for a small business:**

You do not need SPIFFE or mTLS. You need to be able to answer: "Which AI tool did this?"

1. **One account per AI tool.** Not a shared `ai@company.com` used by three different tools. Each tool has its own identity so you can tell them apart in logs.
2. **Document what is connected.** A simple spreadsheet: Tool name, what it is connected to, which account it uses, who set it up, when. This is your agent registry. It does not need to be fancier than that.
3. **API keys are not shared.** Each tool gets its own API key. If one is compromised, you revoke that key without breaking everything else.

**Practical actions:**
- Create the spreadsheet. Right now. List every AI tool anyone in the business is using, including personal ChatGPT accounts used for work.
- Separate API keys per tool
- Name accounts descriptively: `chatbot-website@`, `ai-bookkeeping@`, not `bot1@`

#### Layer 2: Authorization

**What it looks like for a small business:**

1. **Ask the question explicitly:** "What is the worst this tool could do with the access it has?" For every AI tool. Write the answer down. If the answer scares you, reduce the access.
2. **Remove access you do not understand.** If the AI integration asked for 12 permissions and you only understand 4 of them, that is not fine. Either understand them all or remove the ones you do not need. (In practice: most small businesses click "Allow" on OAuth screens without reading them. This is the single highest-risk behaviour.)
3. **No admin accounts for AI tools.** Ever. The chatbot does not need Global Admin. The bookkeeping AI does not need full accounting admin. Find the minimum permission level that works.
4. **Review third-party integrations.** If a tool is connecting your data to an external service, you need to know: where is the data going, is it stored there, who else can access it, what happens if that company goes away? If you cannot answer these questions, do not connect it.

**Practical actions:**
- For each AI tool in your registry: list every permission it has, document whether each is needed, remove unnecessary ones
- Replace admin-level connections with read-only or limited-scope alternatives
- Review OAuth consent grants in M365 Admin Centre (Enterprise Applications, User Consent)
- Block users from consenting to new applications without admin approval (Entra ID, Enterprise Applications, Consent and Permissions: set "Users can consent to apps" to No)

#### Layer 3: Observability

**What it looks like for a small business:**

You are not building a SIEM. You need to be able to answer: "What did the AI do last Tuesday?"

1. **Turn on audit logging.** Microsoft 365 has audit logging. It is probably already on. Make sure it is. Make sure the retention period covers at least 90 days.
2. **Know where to look.** When something goes wrong (and it will) you need to know where the logs are for each AI tool. Some tools log to M365 audit, some have their own dashboards, some log nowhere. Document this in your registry.
3. **Check periodically.** Set a monthly calendar reminder: spend 15 minutes reviewing what your AI tools have been doing. Look for: unusual volume, actions you do not recognise, access to data you did not expect. You do not need a dashboard for this. You need a habit.
4. **Keep the receipts.** For paid AI APIs: review the usage dashboard monthly. Spikes in usage could mean the tool is working overtime on something you did not intend, or your API key has been compromised.

**Practical actions:**
- Verify M365 audit logging is enabled and retention is set to at least 90 days
- Add a "Logging" column to your agent registry: where does this tool log, and how do you access those logs?
- Monthly 15-minute AI tool review (calendar it)
- Monthly API usage/cost review

#### Layer 4: Governance

**What it looks like for a small business:**

1. **Someone is responsible.** Pick one person. When something goes wrong with an AI tool, that person triages it. They do not need to fix it; they need to know it happened and make sure it gets addressed.
2. **You can turn it off.** For every AI tool: document how to disable it. Not "delete the account" as that might lose data. How to disable it quickly and reversibly. Test this. If you cannot disable your chatbot in under 5 minutes, that is a problem.
3. **Quarterly review.** Every 3 months, review the registry: Are these tools still needed? Have permissions crept? Any issues in the last quarter? This takes 30 minutes and prevents slow accumulation of risk.
4. **New tools go through a checklist.** Before connecting a new AI tool to company data, answer five questions: (1) What data does it access? (2) What is the worst it could do? (3) Who is responsible? (4) How do we turn it off? (5) Where are the logs? If you cannot answer all five, do not connect it yet.

**Practical actions:**
- Name the AI tools owner
- For each tool in the registry: add a "How to disable" column with step-by-step instructions
- Schedule quarterly AI tool review (30 minutes, calendar it)
- Create a one-page "New AI Tool Checklist" with the five questions above

### The Minimum Viable Starting Point

If a small business does nothing else:

1. **The spreadsheet.** List every AI tool, what it is connected to, what access it has, who set it up.
2. **Read-only default.** Every AI tool gets read-only access unless there is a documented reason for write access.
3. **Human-in-the-loop for external comms.** AI drafts, human sends. No autonomous customer-facing communication.
4. **Spending caps.** Every paid AI API has a monthly limit.
5. **One person is responsible.** Named. Not "everyone."

That is it. Five actions. No consultants needed. No enterprise software. This gets a small business from "None" across the board to "Basic" on every layer. And Basic across every layer is dramatically better than Advanced on one layer and None on the rest.

---

## Comparing the Two Profiles

| Concern | Enterprise (Profile A) | Small Business (Profile B) |
|---------|----------------------|---------------------------|
| **Biggest risk** | Compliance violation, regulatory fine, data breach at scale | Business-ending privacy breach, financial error, shadow AI |
| **Identity solution** | Entra ID application registrations, managed identities | Dedicated M365 accounts per tool, separated API keys |
| **Authorization approach** | Graph API scoping, custom RBAC roles, Conditional Access | OAuth consent review, read-only defaults, admin approval for new apps |
| **Observability** | Sentinel, Log Analytics, structured logging, reasoning traces | M365 audit log, monthly manual review, API usage dashboards |
| **Governance** | Agent registry, trust levels, automated narrowing, CAE | Named owner, quarterly review, disable procedures, new tool checklist |
| **Cost to implement** | Significant (tooling, staff time, possibly licensing) | Minimal (time only, mostly configuration of existing tools) |
| **Timeframe** | 3 to 6 months for full maturity | 1 to 2 days for minimum viable, ongoing quarterly reviews |
| **What success looks like** | Every agent has documented blast radius, scoped identity, monitored behaviour, and governed lifecycle | Every AI tool is listed, minimally permissioned, monitored enough to notice problems, and one person can turn it off |

---

## License

[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)

You are free to share and adapt this material for any purpose, including commercial use, with attribution.

---

*Companion to [The Agent Trust Chain](agent-trust-chain.md). That document is the framework. This document is the field guide.*

*Developed by [Okavyx](https://okavyx.ai).*
