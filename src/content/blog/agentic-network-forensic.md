---
title: "Agentic Network Forensic — Autonomous Threat Hunting with LLMs"
slug: "agentic-network-forensic"
date: "2026-03-10"
description: "Building an AI agent that autonomously investigates 52 GB of network captures, processes 600K+ Suricata alerts, and produces full forensic reports."
tags: ["security", "network-forensics", "agentic-AI", "project"]
---

## Context

This is my **final project for SC4063 Network Forensics**. The brief was straightforward: given a massive PCAP dataset, produce a forensic investigation report. Part 1 was done manually. For Part 2, I built an autonomous AI agent that does the entire investigation on its own — from ingesting raw PCAPs to generating a PDF report with MITRE ATT&CK mappings.

## The Problem

Network forensic investigations are tedious. You're staring at Wireshark, grepping through Zeek logs, cross-referencing Suricata alerts, and trying to piece together what happened. The dataset I was working with was 52 GB of PCAPs, 600K+ Suricata alerts, and 30 GB of Zeek logs. Doing this manually takes days. I wanted to see if an LLM agent could do it autonomously.

## What I Built

**Agentic Network Forensic** is a tool-calling LLM agent that follows a strict 13-step investigation protocol. It streams through massive files without loading them into memory, calls specialized analysis tools, cross-references findings, and generates comprehensive forensic reports.

The agent isn't just a wrapper around an LLM — it has guardrails against hallucination, mandatory tool-call requirements, and a findings pipeline that reconstructs results directly from raw evidence (bypassing LLM context window truncation).

## Tech Stack & Why

**Google Gemini (primary LLM)** — The 1M token context window was critical. With 600K+ alerts and 30 GB of logs, I needed a model that could hold substantial context. Gemini's free tier made it practical for repeated benchmark runs. I also added Groq (128K tokens, LLaMA 3.3 70B) and Ollama (local, Qwen3:14b) as alternatives — Groq for speed, Ollama for fully offline analysis where sensitive network data shouldn't leave the machine.

**tshark + nfstream** — I needed to process 20 GB+ PCAPs without loading them into memory. tshark (Wireshark CLI) lets you stream packet fields with `-c` count limits for chunked extraction. nfstream is a C-based flow engine that computes flow-level features in a streaming fashion — critical for beaconing and exfiltration detection at scale.

**Custom Zeek & Suricata parsers** — The Zeek logs were ECS-wrapped JSON from Filebeat (30 GB). Off-the-shelf parsers would try to load the entire file. I wrote streaming parsers that process line-by-line, bucketing alerts by severity and category, deduplicating ET/ETPRO rule pairs, and tracking per-source-IP statistics — all with constant memory footprint.

**YARA + ThreatFox IOCs** — For malware signature scanning and known-bad indicator matching. I embedded IOC lists for Cobalt Strike, SocGholish, cryptominers, and other common threats so the agent can flag known infrastructure immediately.

**Flask + SocketIO** — Real-time web dashboard that streams agent logs, shows protocol charts, top talkers, findings tables, and MITRE ATT&CK visualizations as the analysis runs.

**ReportLab** — PDF generation for the final forensic report with formatted sections, tables, timelines, and MITRE ATT&CK mappings.

## How the Agent Works

The agent follows a mandatory 13-step investigation protocol. It cannot finish until all required tools are called — this prevents the LLM from jumping to conclusions.

1. Parse Suricata alert summary (severity breakdown, top rules)
2. Extract C2 IOCs from Suricata alerts
3. Identify affected hosts
4. Deep-dive Suricata alerts (Cobalt Strike, exploits, credentials, DGA, cryptominers)
5. Analyze Zeek DCE/RPC for lateral movement
6. Parse Zeek connections
7. DNS analysis (beaconing detection, DGA campaigns)
8. HTTP analysis
9. TLS/SSL analysis
10. File transfer analysis
11. Generate MITRE ATT&CK mapping
12. Save structured JSON findings
13. Generate PDF report

After every tool call, a checkpoint is saved — so if the agent hits a rate limit or crashes, it resumes without re-running the entire analysis.

## Detection Highlights

On the test dataset (52 GB PCAPs, 600K alerts, 30 GB Zeek logs), the agent found:

- **Cobalt Strike beacon** at 10.128.239.57 with hourly callbacks to 179.60.146.34:3389
- **.click DGA campaign** — 197,684 NXDOMAIN queries across 287 unique domains from a single host
- **TacticalRMM compromise** — 27+ internal hosts querying icanhazip.tacticalrmm.io (45,482 queries)
- **WebLogic exploits** (CVE-2020-2551, CVE-2018-2893) targeting a domain controller
- **Virut DGA + cryptominer** on 10.128.239.98 connecting to herominers pools
- **WPAD poisoning surface** — 314,095 unanswered WPAD queries across 20+ hosts
- **Active Directory enumeration** — 22 workstations making 90–307 DRSCrackNames calls each (BloodHound-style recon)

## Interesting Technical Challenges

**Beaconing detection with coefficient of variation.** Detecting C2 beacons means finding regular timing patterns in DNS queries. I track all query timestamps per (source IP, domain) pair, compute the inter-query intervals, then check if the coefficient of variation (std dev / mean) is below 0.05. A CV that low means the timing is almost perfectly regular — a dead giveaway for automated C2 callbacks. This flagged wallhaven.ufcfan.org at 78.1s ± 0.1s mean interval over 3 days.

**Suricata rule deduplication.** Suricata's ET and ETPRO rule sets often have mirrored rules for the same event (e.g., "ET Trojan.X" and "ETPRO Trojan.X"). Without deduplication, alert counts double. I wrote a normalizer that collapses these pairs so the severity counts and top-rule rankings reflect reality.

**Hallucination prevention.** LLMs hallucinate IPs, invent alerts, and claim certainty where none exists. The guardrails layer extracts all IPs from LLM output and cross-references them against raw evidence. It also flags overconfident language ("definitely", "confirmed attack") and downgrades findings below 0.6 confidence from critical/high to medium. The findings pipeline reconstructs the final report directly from raw tool outputs, so even if the LLM's context window truncates data, the report stays accurate.

**Memory management at scale.** 52 GB of PCAPs can't fit in memory. Every tool in the pipeline streams: tshark extracts fields in chunks, nfstream processes flows via its C engine, Zeek logs are parsed line-by-line, Suricata alerts are bucketed on-the-fly. The entire analysis runs with constant memory regardless of input size.

**LLM context window limits.** Even with Gemini's 1M tokens, 600K alerts don't fit. The solution: specialized tools pre-aggregate and summarize before the LLM sees anything. The agent gets statistics, top-N lists, and flagged anomalies — not raw data. The full evidence is preserved separately for report generation.

## What I Learned

Building an agentic system for security forensics taught me that the hard part isn't the LLM — it's the tooling around it. The parsers, streaming architecture, guardrails, and evidence pipeline are where the real engineering lives. The LLM is just the orchestration layer that decides what to look at next.

The mandatory tool-call protocol was essential. Without it, the agent would run 2-3 tools and declare "investigation complete." Forcing it through all 13 steps consistently produced findings it would have otherwise missed.

The code is available on [GitHub](https://github.com/fearyj/Agentic_Network_Forensic).
