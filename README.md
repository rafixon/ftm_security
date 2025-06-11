# ftm_security
# FTM Spoofing & Replay Toolkit
> Practical experiments that falsify Wi-Fi Fine Timing Measurement (FTM) ranging with commodity NICs.

![Build](https://img.shields.io/github/actions/workflow/status/your-org/ftm-spoof/ci.yml?label=CI)
![License](https://img.shields.io/github/license/your-org/ftm-spoof)
![Last Commit](https://img.shields.io/github/last-commit/your-org/ftm-spoof)

## Table of Contents
- [Overview](#overview)
- [Attack Catalogue](#attack-catalogue)
  - [Spoofing FTM Responses](#spoofing-ftm-responses)
  - [Replaying FTM Sessions & Responses](#replaying-ftm-sessions--responses)
  - [Replaying PHY-Modified FTM Responses](#replaying-phy-modified-ftm-responses)
  - [In-Session Injection](#in-session-injection)
  - [Configuration Side-Channels & Bias Attacks](#configuration-side-channels--bias-attacks)
  - [Bandwidth-Mismatch Offsets](#bandwidth-mismatch-offsets)
  - [Channel-Swap MITM Attacks](#channel-swap-mitm-attacks)
  - [Forced Downgrade to Unencrypted FTM](#forced-downgrade-to-unencrypted-ftm)
- [Key Takeaways](#key-takeaways)
- [Quick Start](#quick-start)
- [Requirements](#requirements)
- [Usage](#usage)
- [Defensive Research Directions](#defensive-research-directions)
- [Responsible Disclosure](#responsible-disclosure)
- [Citing This Work](#citing-this-work)
- [License](#license)

---

## Overview
802.11mc Fine Timing Measurement (FTM) lets Wi-Fi devices estimate distance by exchanging timestamped frames.  
While 802.11az strengthens this with Protected FTM (PASN + Secure-LTF + Sequence Authentication Code), many commercial stacks silently fall back to legacy, unprotected FTM—leaving room for manipulation.

This repository collects **proof-of-concept code, pcap traces, and analysis notebooks** for eight attack primitives that bias or fully spoof FTM ranging on off-the-shelf hardware.

> **Ethics notice:** The code is for academic and defensive research only. Never test on networks you do not own or have explicit permission to audit.

---

## Attack Catalogue

### Spoofing FTM Responses
A rogue device advertises itself as an FTM responder, then forges the timestamp pairs (`t₁/t₄`) or optional LCI/LCR fields in every FTM-Response it emits. Because pre-az frames carry **no MIC**, any station that starts ranging accepts the bogus data and derives an attacker-chosen distance or absolute position.  
*Practical demo:* monitor/inject capable NICs achieve **metre-level bias** without breaking association security.  
[[aanjhan.com](https://aanjhan.com)]

### Replaying FTM Sessions & Responses
An eavesdropper records a full ranging burst (request → multiple responses) at **location A** and later replays it verbatim at **location B**. Legacy FTM lacks freshness, so the initiator re-uses stale timestamps and still believes it is at A. Replaying just one captured response inside a live burst is enough to corrupt multilateration engines.  
[[aanjhan.com](https://aanjhan.com)]

### Replaying PHY-Modified FTM Responses
Following Schepers *et al.*: replay a previously captured response but **narrow the channel** (e.g., 80 MHz → 20 MHz). The longer OFDM symbol stretches the perceived flight-time, adding **tens of metres** to the computed range—all with one frame.  
[[winlab.rutgers.edu](https://winlab.rutgers.edu)]

### In-Session Injection
During an ongoing burst the adversary spoofs the responder’s MAC and injects a crafted response **before** the legitimate one arrives. Most chipsets accept the **first** frame matching the current 1-byte dialog token; the forged timestamps are processed while the genuine frame is discarded as a duplicate. No jamming required.  
[[petsymposium.org](https://petsymposium.org)] [[northeastern.edu](https://repository.library.northeastern.edu)]

### Configuration Side-Channels & Bias Attacks
Initiators and responders leak static features—fixed burst-size, dialog-token patterns, missing MAC randomisation—enabling long-term device tracking. Malformed FTM frames further reveal firmware versions.  
[[researchgate.net](https://researchgate.net)] [[pmc.ncbi.nlm.nih.gov](https://pmc.ncbi.nlm.nih.gov)]

### Bandwidth-Mismatch Offsets
If two legitimate devices silently fall back from 40/80 MHz to 20 MHz while the peer still assumes the original width, the reported RTT shifts by **≈ 6–8 m**. An active adversary can induce the mismatch via selective jamming, biasing distance *without* spoofing any frame.  
[[winlab.rutgers.edu](https://winlab.rutgers.edu)] [[sciencedirect.com](https://sciencedirect.com)]

### Channel-Swap MITM Attacks
Early 802.11az drafts omitted the **Operating-Class Information (OCI)** element from the PASN handshake. A man-in-the-middle relays authentication on channel 1, then silently moves one party to channel 6 for the ranging exchange. The two sides measure ToF on different frequencies; the distance passes integrity checks yet is meaningless. (Fixed by Comment LB-253.)  
[[mentor.ieee.org PDF](https://mentor.ieee.org)]

### Forced Downgrade to Unencrypted FTM
802.11az keeps backward compatibility: if either side advertises only legacy FTM, most stacks quietly downgrade. Attackers jam or corrupt PASN until the estimator retries without security, **re-opening all spoofing and replay avenues**. Advisories label this a live threat on Wi-Fi 7 phones and mesh APs.  
[[thehackernews.com](https://thehackernews.com)] [[usenix.org](https://usenix.org)] [[netally.com](https://cyberscope.netally.com)]

---

## Key Takeaways
- **PASN + Secure-LTF + SAC** harden ranging, but only when enabled *and* enforced.
- Sophisticated adversaries exploit **PHY quirks, configuration leaks, or downgrade paths** to bias distance without triggering integrity checks.
- Defensive work should focus on **downgrade prevention, PHY-parameter sanity checking, and cross-sensor consistency tests.**

---

## Quick Start
```bash
git clone https://github.com/your-org/ftm-spoof.git
cd ftm-spoof
pip install -r requirements.txt          # analysis notebooks
./scripts/build.sh                       # optional: compile the C/CPP PoC injectors

