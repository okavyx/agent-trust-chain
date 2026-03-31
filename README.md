# The Agent Trust Chain

A practical framework for evaluating, designing, and governing AI agent deployments in enterprise environments.

Five layers of trust. Built from operational experience running multi-agent systems in production. Not from whitepapers.

> **Disclaimer:** This framework is a permanent work in progress. It reflects our current understanding based on real-world operational experience, and it will evolve as the industry matures, as we encounter new failure modes, and as contributors bring new perspectives. Nothing here should be taken as legal, compliance, or security advice for your specific environment. Use it as a thinking tool and a starting point, not a finished standard. If a section feels incomplete, it probably is. That is by design: we would rather publish what we know today and improve it openly than wait for a perfection that never arrives.

## The Five Questions

Every serious conversation about AI agents in enterprise ends at these five questions:

1. **Containment** (Layer 0): If everything fails, what is the worst that can happen?
2. **Identity** (Layer 1): Can you verify who this agent is and who authorised it?
3. **Authorization** (Layer 2): Are permissions enforced at infrastructure level, or just in the prompt?
4. **Observability** (Layer 3): Can you see what it is doing in real-time and prove why it did it?
5. **Governance** (Layer 4): How do you adjust trust over time, and can you kill access instantly?

If you cannot answer all five, you know exactly where to invest.

## Documents

| Document | Description |
|----------|-------------|
| [The Agent Trust Chain](agent-trust-chain.md) | The framework. Five layers, principles, implementation patterns, assessment questions, maturity indicators. |
| [The Agent Trust Chain: Applied](agent-trust-chain-applied.md) | The field guide. Two detailed scenarios: a mid-size Microsoft enterprise and a small business with no IT department. Current state, target state, practical actions. |

## Quick Start

**For enterprise teams:** Read the framework, run the 25 assessment questions against your current agent deployment, plot your maturity profile. The gaps will be obvious.

**For small businesses:** Skip to [The Minimum Viable Starting Point](agent-trust-chain-applied.md#the-minimum-viable-starting-point) in the Applied doc. Five actions, no consultants needed, takes a day.

**For consultants:** Use the framework as a diagnostic tool. The five questions work in a 30-minute conversation. The maturity indicators give you a visual that executives understand immediately.

## What Makes This Different

- **Layer 0 (Containment)** is the layer nobody else talks about. It is the only layer that works when every other control fails.
- **The Composition Problem** in Layer 2 addresses what happens when individually safe permissions become dangerous in sequence. Most frameworks only evaluate permissions one at a time.
- **Observability, not audit.** Layer 3 splits into three sub-layers: telemetry (what happened), reasoning trace (why), and outcome verification (did it work). "We keep logs" is not observability.
- **Governance is a lifecycle, not a kill switch.** Layer 4 covers seven operations from initial grant through promotion, narrowing, and revocation. Trust evolves; static permissions do not.
- **Built from production incidents.** Every principle has an operational scar behind it.

## Contributing

This is a living framework. Contributions are welcome:

- Additional implementation patterns for any layer
- Industry-specific assessment criteria (healthcare, finance, government)
- Case studies (anonymised) of trust chain failures or successes
- Tooling recommendations for specific layers

Please open an issue for discussion before submitting large changes.

## License

[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)

You are free to share and adapt this material for any purpose, including commercial use, with attribution.

---

*Developed by [Okavyx](https://okavyx.com.au). Built from scars, not slides.*
