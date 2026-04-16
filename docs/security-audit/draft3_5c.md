## 8. Conclusion and Outlook

The Agent Authority Problem reveals that the ambient authority model cannot secure systems whose intent emerges from — and can be corrupted by — the data they process. This property holds for any architecture where data and instructions share a processing channel, encompassing all current LLM-based agent systems.

Our dual-gate event-driven capability architecture demonstrates that this problem can be circumvented by replacing the authorization model. The EventInjectionGate constrains what enters the system; the CapabilityGate constrains what the LLM can do. Together they transform agent security from a single-barrier model (dependent on LLM injection resistance) to a multi-barrier model with quantifiable probability bounds (Theorem 1) and deterministic zero-probability guarantees for untrusted sources (Section 3.3) and Rule-of-Two-enforced batches (Theorem 3).

Three challenges remain. The cross-batch contamination gap demands session-level taint management. The semantic attack boundary marks the interface between structural security and behavioral safety. And empirical validation against standard benchmarks (Section 4.4) will quantify the bound's tightness and the false positive/negative rates of taint tracking.

The class of architectures providing event provenance, cascade enforcement, batch atomicity, trust-level tool restriction, and deterministic gating represents a necessary direction for autonomous agents in high-stakes domains. The critical requirement is not any specific implementation, but the abandonment of ambient authority as the organizing principle for agent security.

---

## References

1. Gabriel, I. et al. We need a new ethics for a world of AI agents. *Nature* **644**, 291–294 (2025).
2. OpenClaw Project. Security advisories GHSA-2026-25253 (RCE), GHSA-2026-25254, GHSA-2026-25255. GitHub (2026).
3. Sangfor Technologies. OpenClaw Security Risks. Report at https://www.sangfor.com/blog/cybersecurity/openclaw-ai-agent-security-risks-2026 (accessed 10 April 2026).
4. Censys. Exposed OpenClaw Instances Analysis. Report at https://censys.io (accessed 10 April 2026).
5. KuCoin Research. AI Trading Agent Vulnerability: $45M Crypto Security Breach. Report at https://www.kucoin.com/blog (accessed 10 April 2026).
6. OWASP. Top 10 for Large Language Model Applications v2.0. https://owasp.org (2025).
7. Nature Editorial. Let 2026 be the year the world comes together for AI safety. *Nature* **637**, 8–9 (2025).
8. Nature Machine Intelligence Editorial. Multi-agent AI systems need transparency. *Nat. Mach. Intell.* **8**, 1–2 (2026).
9. MITRE. CVE-2026-25253. https://cve.mitre.org (2026).
10. MITRE. CVE-2025-32711 (EchoLeak). https://cve.mitre.org (2025).
11. Dennis, J. B. & Van Horn, E. C. Programming semantics for multiprogrammed computations. *Commun. ACM* **9**, 143–155 (1966).
12. Debenedetti, E. et al. Defeating Prompt Injections by Design. Preprint at https://arxiv.org/abs/2503.18813 (2025).
13. Greshake, K. et al. Not what you've signed up for. *Proc. AISec '23*, 79–90 (2023).
14. Xiao, A. et al. Agentic AI Security. Preprint at https://arxiv.org/abs/2510.23883 (2025).
15. Zhan, Q. et al. InjecAgent. *Findings of ACL 2024* (2024).
16. Yi, J. et al. Benchmarking Indirect Prompt Injection. Preprint at https://arxiv.org/abs/2312.14197 (2023).
17. Partnership on AI. AI Incident Database. https://incidentdatabase.ai (accessed 10 April 2026).
18. Wallace, E. et al. The Instruction Hierarchy. Preprint at https://arxiv.org/abs/2404.13208 (2024).
19. EU Parliament. Regulation (EU) 2024/1689 (AI Act). *Official Journal of the EU* (2024).
20. Birgisson, A. et al. Macaroons. *Proc. NDSS* (2014).
21. FINOS. Agent Authority Least Privilege Framework. https://air-governance-framework.finos.org (accessed 10 April 2026).
22. Suh, G. E. et al. Secure Program Execution via Dynamic Information Flow Tracking. *Proc. ASPLOS* (2004).
23. Ouyang, L. et al. Training language models to follow instructions with human feedback. *Proc. NeurIPS* **35**, 27730–27744 (2022).
24. Bai, Y. et al. Constitutional AI. Preprint at https://arxiv.org/abs/2212.08073 (2022).
25. European Commission. EU AI Act Articles 9, 12, 14, 15. Regulation (EU) 2024/1689 (2024).
26. The White House. Executive Order 14110. *Federal Register* **88**, 75191 (2023).
27. China Cyberspace Administration. Interim Measures for Generative AI Services. (2023).
28. Berditchevskaia, A. et al. Risks of AI scientists. *Nat. Commun.* **16**, 5003 (2025).
29. Cisco Security. Personal AI Agents like OpenClaw Are a Security Nightmare. Report at https://blogs.cisco.com (accessed 10 April 2026).
30. Qualys. Anatomy of an Autonomous AI Agent Risk. Report at https://blog.qualys.com (accessed 10 April 2026).
31. MarketsandMarkets. AI Agents Market Size. Report at https://www.marketsandmarkets.com (accessed 10 April 2026).
32. Costa, M. & Köpf, B. Securing AI Agents with Information-Flow Control. Preprint at https://arxiv.org/abs/2505.23643 (2025).
33. Siu, V. et al. A Framework for Formalizing LLM Agent Security. Preprint at https://arxiv.org/abs/2603.19469 (2025).
