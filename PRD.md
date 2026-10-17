# PRD: CybrDial — Unified Outbound Dialer & Voice AI Platform

**Version:** 1.2 (Updated Oct 17, 2026 — hosting section added)

**Author:** Fito / CybrFlux

**Status:** Draft — living document

**Repo:** https://github.com/CybrFlux/CybrDial

**Google Doc:** https://docs.google.com/document/d/13zPkSvIA9GzFXZN4EPSmnNsW_A9N5AO-681cO1pZYaU/edit

**Prior versions:** v1.1 https://docs.google.com/document/d/1LrkIE-mXP0JvFahE38pLjbhAuN5ElCKc3uw_F-54-9w/edit · v1.0 https://docs.google.com/document/d/108Cp2ulPoVP98fs0m-pf7kMj0wNySZhS2HUg5WVvSpc/edit

---

## 1. Executive Summary

Build a self-hosted, multi-tenant outbound sales and contact-center platform that consolidates Convoso (predictive dialer + compliance), RingCentral (UC + RingCX), and Dialpad (AI-native voice). Own the agentic orchestration and CRM-integration layer. Rent telephony pipes from Telnyx. Fork MIT/BSD/MPL components for SIP edge, media, dialing logic, and voice-AI pipelines. Never merge AGPL into a commercial product without a legal review.

**Crossover math:** Below ~2–3M outbound minutes/month, hybrid (own the brain, rent the throat) beats full self-host. Start hybrid.

---

## 2. Telephony (Telnyx — confirmed Oct 2026)

| Item | Rate | Notes |
|---|---|---|
| SIP outbound | $0.005/min | US local starting rate |
| SIP inbound | $0.0032/min | US local |
| STIR/SHAKEN | Free | A-level on numbers you own |
| Branded Calling | $50 OTC + $50 MRC/brand + $0.075/call | **Limited to US-to-US T-Mobile & Verizon mobiles only.** Landlines and other carriers ignore it. |
| Voice AI agents | $0.05/min all-in | SIP + STT + LLM + TTS |
| Premium AMD | Included with Voice API | Human/machine + iOS screening detection |

**Implication:** Branded Calling is a partial answer-rate lift, not a full CNAM replacement. Budget for it on mobile-heavy campaigns only.

---

## 3. Open-Source Components (fork-safe)

### MIT / BSD / Apache (safe to fork and white-label)

| Component | License | Use |
|---|---|---|
| **DialerJazz** (moazzam-07) | MIT | Power dialer UI + campaigns/leads. Telnyx WebRTC. Tinder-style swipe mode. Small (8★) but modern React/Node stack. Fork for agent console. |
| **Patter** (PatterAI) | MIT | Voice-AI SDK wrapping Telnyx/Twilio/Plivo. Python + TS parity. Vapi/Retell alternative. |
| **Pipecat** | BSD-2 | STT→LLM→TTS pipelines. 16.5k★. Transport-agnostic. |
| **LiveKit + SIP bridge** | Apache-2.0 | SFU + SIP↔WebRTC. Affinity-based load balancing for concurrent calls. |
| **Dograh** | BSD-2 | Self-hosted Vapi/Retell alt. Visual workflow builder, MCP-native, Telnyx. |
| **Bolna** | MIT | End-to-end voice-agent framework. |
| **FreeSWITCH** | MPL 1.1 | Media engine. File-level copyleft only — app code stays proprietary. |
| **mod_amd** (seanbright) | CC0 | Heuristic AMD baseline. Not production-grade alone. |
| **whisper-vm-finetune** | MIT | ML AMD via Whisper STT + classifier. ~98% on hold-out. |
| **vmhunter-acoustic-amd** | MIT | ONNX CNN AMD. ~97% F1. CPU-friendly. |
| **Jambonz 0.9.x** | MIT | Full CPaaS. Pin to 0.9.x — v10+ is commercial (Jan 2026). |
| **AgentLine** | MIT | Phone API for AI agents. MCP-native. Hermes-aligned. |
| **mod_fcc** | MIT | FreeSWITCH REST/WS call-control. |

### AGPL / GPL (study only — do not merge)

- VICIdial, GOautodial, ICTDialer, dev0miky/Outbound-calling-system-P1-Broadcast — all AGPL. Study architecture, reimplement in own code.
- Kamailio (GPLv2), RTPEngine (GPLv3) — run at arm's length as separate containers.

### Twenty CRM (AGPLv3 with carve-out)

Twenty is AGPLv3, but applications that interact via documented Application Interfaces (GraphQL/REST/webhooks/MCP) without modifying core can stay proprietary. Their SDK packages are MIT. **cybrflux-crm is a fork** — fine for internal use. If you host modified Twenty for paying clients, you must share source. Prefer app-layer extensions over core forks for client-facing features.

---

## 4. AMD Strategy (revised)

FreeSWITCH `mod_avmd` only detects the beep after a voicemail greeting. It does not classify human vs machine. Options:

1. **Telnyx Premium AMD** (preferred for v1) — included with Voice API, handles iOS call screening, no self-hosted ML needed.
2. **MIT ML models** — whisper-vm-finetune or vmhunter-acoustic-amd for self-hosted accuracy at scale.
3. **SignalWire mod_com_amd** — commercial, per-channel licensing. Evaluate if Telnyx AMD accuracy is insufficient.

Feed measured AMD latency back into the pacing algorithm. The 1–3s classification window counts toward TCPA's 3% abandonment cap.

---

## 5. Compliance Module (first-class)

No good open-source TCPA/DNC engine exists. Build internal suppression + pluggable API integrations.

| Requirement | Implementation |
|---|---|
| Federal DNC | DNC.com / Contact Center Compliance API (~$0.003–0.02/number). Internal DNC list with 5-year retention. |
| RND (Reassigned Numbers) | Register at reassigned.us. Pricing reduced April 2025 — ~$0.008/query at low volume, ~$0.0016 at 2M+. Searchbug offers API wrapper. Query before every dial. |
| Litigator scrub | DNC.com Litigator Scrub or Trestle add-on. Flag serial TCPA plaintiffs. |
| Mini-TCPA (FL, OK, WA, MD) | Written consent required. Calling hours 8am–8pm local. FL/OK/MD cap at 3 calls per number per 24h. Configurable per-state rules engine. |
| Express written consent | 5-field audit trail. Capture before autodialing cells. |
| Abandoned-call rate | ≤3% per 30-day window. Real-time monitoring. |
| STIR/SHAKEN | Telnyx provides A-level on owned numbers. |
| Recording consent | Two-party states (CA, FL, etc.) — play disclosure or get consent. |

**Safe harbor:** RND query returning "no" shields from TCPA liability if you have consent and the database was wrong. Log every query result.

---

## 6. Architecture (updated)

### Pre-connect (CybrDial)

- Telnyx SIP trunk + number pool with rotation
- DialerJazz-derived agent console (React) or custom
- Go dialer engine: pacing, lead claiming (Postgres RLS), call state
- Redis for hot state (pacing counters, agent presence, caller-ID pool)
- AMD via Telnyx Premium or self-hosted ML
- Compliance gate: DNC + RND + litigator + Mini-TCPA rules before dial

### Post-connect (next-level-life / cybrflux-crm)

- On a call workspace: Client → Policy → Review flow
- Server-owned submission journal with CAS idempotency (reuse from life-desk)
- Deal Desk: checklist, notes, assignment, admin vs staff scoping
- GraphQL/REST to Twenty for CRM sync

### Handoff

Agent pickup event from CybrDial triggers next-level-life overlay with call context (lead ID, campaign, recording URL).

### Event path for call records

Syncable-entity runner in cybrflux-crm validates + recomputes cache on every write — too heavy for hundreds of events/sec. Add lightweight path: Redis stream or Postgres LISTEN/NOTIFY writes call records directly. Trigger full syncable-entity flow only on structural changes (new disposition codes, schema).

---

## 7. Hosting & Infrastructure (NEW in v1.2)

Split across three layers by sensitivity and latency.

### Layer 1 — Telephony (Telnyx owns entirely)

- SIP connection points at sip.telnyx.com with US signaling IPs whitelisted
- AnchorSite set to US East (Ashburn) so media anchors on Telnyx's private backbone near your servers — cuts ~43ms off p95 RTT vs Twilio
- Data locality locked to US for call recordings, transcripts, CDRs at rest (matters for insurance clients with PHI-adjacent data)
- Failover is automatic geographic with 30-second switch, RPO 1 hour, RTO 4 hours

### Layer 2 — SIP & Media Stack (Kamailio + FreeSWITCH + RTPEngine)

- **Kamailio** at the edge as signaling proxy: 2 cores / 2GB enough for pilot; 4-core handles 5–10k SIP transactions/sec with 100k+ concurrent registrations. Never touches media.
- **FreeSWITCH** behind it for AMD, recording, IVR, dialplan. Single well-tuned node on pass-through G.711 carries 5–10k concurrent calls; realistically 800–1,500/box with recording, 500–1,000 with transcoding. Production: 3–4 nodes at 8 cores / 16GB each behind the Kamailio pair.
- **RTPEngine** on its own 8-core box handling 2–4k RTP streams.
- **Hard recommendation: bare metal, not VM** — RTP packet timing depends on a reliable kernel clock hypervisors don't always propagate. Debian is the reference OS.
- Kamailio (GPLv2) and RTPEngine (GPLv3) run at arm's length as separate containers — do not merge into proprietary code.

### Layer 3 — App Layer (Google Cloud, us-east4, Ashburn VA)

- **Provider: Google Cloud us-east4** — literally in Ashburn, same city as Telnyx's Equinix PoP. Best network proximity. GCP runs 7–15% cheaper than AWS on equivalent compute with automatic sustained-use discounts up to 30% and no upfront commitment. $300 in credits for 90 days covers the pilot.
- All three majors (AWS, GCP, Azure) carry SOC 2 Type 2 — compliance is control-based, not host-mandated. Use the carve-out method: auditor relies on provider's report for physical security; you own IAM, VPC rules, audit logging, change management.
- **AWS is the fallback** if you later need deeper managed services — but support alone runs ~9% of your first $10k/month, brutal at pilot scale.
- **Azure is the odd one out** — most certifications at 100+, flat $100 support, built for Microsoft shops; you'd pay for identity integration you don't use.
- **Fly.io and DigitalOcean** also have Type 2 if you want cheaper options for non-sensitive workloads.
- Existing Hostinger VPS keeps running Back and lighter bots — but the dialer media path must NOT share that box. RTP is bandwidth-hungry and jitter-sensitive.

### Network proximity (the tie that binds)

- Co-locate media servers as close as possible to Telnyx's PoP
- Internal RTT under 1ms between Kamailio, RTPEngine, FreeSWITCH on the same VLAN
- Whole stack stays under the 800ms threshold where conversations start feeling robotic
- Self-hosted STT/TTS GPU boxes go in the same region

### Pilot footprint (10 seats)

One GCP project in us-east4: one Kamailio box, one FreeSWITCH node, one RTPEngine, one small Compute Engine VM for Postgres + Redis, Cloud Storage for recordings. Grow into the production topology as tenant count climbs.

### Anti-pattern

Don't recreate single-point-of-failure risk in the app layer. Last October a major AWS outage took Twilio down across the board; Telnyx stayed up because their infrastructure spans multiple clouds. Telnyx is already your multi-cloud insurance on telephony — don't park everything in one hyperscaler account.

---

## 8. Phased Rollout

**Phase 0 (weeks 1–4):** Telnyx trunk, FreeSWITCH + Kamailio in Docker, Go dialer stub (manual + preview), React agent console from DialerJazz patterns, basic DNC + RND gate. **Hosting: single GCP us-east4 project.**

**Phase 1 (weeks 5–10):** Predictive mode, Telnyx AMD, caller-ID rotation, recording + Cloud Storage, Nango connector (Jobber or Salesforce), post-call transcript via Pipecat, compliance dashboard. 10-seat pilot.

**Phase 2 (weeks 11–18):** Live AI coaching, AI inbound receptionist, MCP server for Hermes, sentiment analytics. Second client.

**Phase 3 (weeks 19–30):** Multi-site admin, RBAC, Kubernetes, SOC 2 kickoff, white-label branding. Evaluate Jambonz 0.9.x vs custom stack at volume.

---

## 9. Open Questions

1. Telnyx Premium AMD accuracy sufficient for v1, or budget for ML self-host?
2. DialerJazz fork vs custom React console — how much UI to reuse?
3. First CRM connector: Jobber (pool clients) or Salesforce (enterprise)?
4. Pricing: per-seat + per-minute hybrid, or Dialpad-style AI credit pool?
5. SOC 2 before or after second client?
6. Legal review of "run copyleft at arm's length" + Twenty AGPL carve-out before Phase 1.
7. **(NEW)** Bare-metal vs cloud VM for FreeSWITCH — cost of colo in Ashburn vs GCP us-east4 latency tradeoff?
8. **(NEW)** Multi-region expansion: when does a second GCP region (or AWS us-east-1) become necessary for DR?

---

## 10. Appendix: Repo Quick-Reference

**MIT/BSD/Apache:**
- github.com/moazzam-07/DialerJazz — MIT power dialer, Telnyx WebRTC
- github.com/PatterAI/Patter — MIT voice-AI SDK
- github.com/pipecat-ai/pipecat — BSD-2
- github.com/livekit/livekit + github.com/livekit/sip — Apache-2.0
- github.com/dograh-hq/dograh — BSD-2
- github.com/bolna-ai/bolna — MIT
- github.com/AgentLineHQ/AgentLine — MIT
- github.com/Taering365/mod_fcc — MIT
- github.com/rixwankhan/whisper-vm-finetune — MIT AMD
- github.com/vmhunterhq/vmhunter-acoustic-amd — MIT AMD

**MPL:**
- github.com/signalwire/freeswitch

**AGPL (study only):**
- github.com/goautodial, github.com/ictinnovations/ictdialer, github.com/dev0miky/Outbound-calling-system-P1-Broadcast

**Twenty:**
- github.com/twentyhq/twenty — AGPLv3 with app-layer carve-out
- github.com/CybrFlux/cybrflux-crm — your fork
- github.com/CybrFlux/next-level-life — call-flow overlay + life-desk journal

---

*v1.2 adds Section 7 (Hosting & Infrastructure): three-layer split, Telnyx Ashburn anchoring, GCP us-east4 as default app-layer provider with AWS/Azure/Fly.io/DigitalOcean rationale, bare-metal FreeSWITCH recommendation, pilot footprint, network-proximity rules, and anti-single-point-of-failure guidance. Two new open questions on colo vs cloud and multi-region DR. Prior versions retained for history.*
