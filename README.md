# Local Home NVR

A documentation of my decision to replace Ring with a fully local, self-hosted network video recorder built around a Reolink PoE doorbell camera, Frigate NVR, and TrueNAS storage. 

This repo documents the hardware choices, the networking work required to make it possible, and the privacy reasoning behind all of it.

---

## Table of Contents

- [Why I Replaced Ring](#why-i-replaced-ring)
- [What I Built](#what-i-built)
- [Hardware](#hardware)
- [Software Stack](#software-stack)
- [The Networking Work](#the-networking-work)
- [Integration with Home Security Lab](#integration-with-home-security-lab)
- [What This Enables](#what-this-enables)
- [Next Steps](#next-steps)

---

## Why I Replaced Ring

I decided to replaced Ring because I became unwilling to accept the specific tradeoffs Ring was making on my behalf without my meaningful consent.

The first thing that pushed me over the edge was Ring's relationship with Flock Safety — a surveillance company whose cameras have been deployed by law enforcement agencies across the country and whose security practices have been extensively documented as deeply problematic. 

I came across Benn Jordan, a musician and independent security researcher, who spent months investigating Flock's infrastructure and published a series of YouTube videos exposing its vulnerabilities: cameras livestreaming openly to the internet, physical access exploits achievable in under 30 seconds, unencrypted credential transmission, and no 2FA requirement for law enforcement access. His work, done in collaboration with researcher Jon Gaines and journalists at 404 Media, is worth watching for anyone who owns a smart home camera or lives near Flock hardware.

Ring's planned integration with Flock, which would allow law enforcement to request Ring footage through Flock's platforms, was the specific trigger. The fact that Ring framed footage sharing as "optional" didn't address the underlying concern: my doorbell camera data would be part of an infrastructure I had no visibility into and no real control over.

The second push was Ring's Search Party feature, which introduced crowd-sourced surveillance framing into what had been marketed as a personal security product.

_(Note: Ring did cancel the Flock partnership in early 2026 following public backlash. The broader concerns about Ring's relationship with law enforcement and third-party data access remain.)_

This is part of a broader effort in my household to be more intentional about IoT devices and where data flows. I deleted my social media last year for similar reasons. The goal isn't to avoid technology — it's to use it with intentionality, and to choose local control over third-party dependency wherever that's genuinely feasible.

**Further reading and viewing:**

- [Benn Jordan's YouTube channel](https://www.youtube.com/@bennjordan) — his Flock Safety series is the place to start
- [404 Media: Flock Exposed Its AI-Powered Cameras to the Internet](https://www.404media.co/how-benn-jordan-discovered-flocks-cameras-were-left-streaming-to-the-internet/)
- [Privacy Guides: Ben Jordan Exposes Severe Security Vulnerabilities in Flock Surveillance Cameras](https://www.privacyguides.org/news/2025/11/17/ben-jordan-exposes-severe-security-vulnerabilities-in-flock-surveillance-cameras/)

---

## What I Built

A fully local doorbell camera system: a Reolink PoE doorbell camera connected via dedicated ethernet to a TP-Link PoE switch, feeding into Frigate NVR running as a Docker container on a Mac Mini server, with recordings stored on a dedicated partition in TrueNAS. Motion detection and object recognition run entirely on-device. 

```
┌─────────────────────────────────────────────────────────┐
│                     HOME NETWORK                        │
│                                                         │
│  ┌─────────────────┐                                    │
│  │ Reolink Video   │                                    │
│  │ Doorbell PoE    │                                    │
│  └────────┬────────┘                                    │
│           │ ethernet (PoE)                              │
│           ▼                                             │
│  ┌─────────────────┐                                    │
│  │ TP-Link PoE     │                                    │
│  │ Switch          │──────────────────────┐             │
│  └─────────────────┘                      │             │
│                                           │ ethernet    │
│                                           ▼             │
│                              ┌────────────────────────┐ │
│                              │   2014 Mac Mini        │ │
│                              │   (Central Node)       │ │
│                              │                        │ │
│                              │  • Frigate NVR         │ │
│                              │    (Docker)            │ │
│                              │  • Motion detection    │ │
│                              │  • Object recognition  │ │
│                              └────────────┬───────────┘ │
│                                           │             │
│                                           ▼             │
│                              ┌────────────────────────┐ │
│                              │   TrueNAS              │ │
│                              │   (Storage)            │ │
│                              │                        │ │
│                              │  • Dedicated security  │ │
│                              │    partition           │ │
│                              │  • Local recordings    │ │
│                              │  • No cloud backup     │ │
│                              └────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## Hardware

|Device|Role|Notes|
|---|---|---|
|Reolink Video Doorbell PoE|Doorbell camera|PoE powered, RTSP stream, no cloud account required for local use|
|TP-Link PoE Switch|Powers doorbell, distributes ethernet|Runs to first floor living and media rooms as well|
|Mac Mini (2014)|Frigate host|Runs Frigate as a Docker container alongside other home services|
|TrueNAS|Storage|Security footage stored on a dedicated partition separate from media and general storage|

---

## Software Stack

**Frigate NVR** runs as a Docker container on the 2014 Mac Mini. It receives the RTSP stream from the Reolink doorbell, handles motion detection and object recognition (CPU-based for now), and writes recordings to the TrueNAS security partition.

Frigate was chosen specifically because it is designed for local processing — it does not require any cloud account, does not phone home, and gives full control over retention policies, detection zones, and recording rules. The web UI is accessible only on the local network.

**TrueNAS** handles storage with partitions segmented by use case: media, security footage, general file storage, and backups. Keeping security footage on its own partition makes retention management and access control straightforward.

**No cloud dependency.** The Reolink doorbell supports RTSP streaming natively without requiring a Reolink cloud account. Frigate connects directly to that stream. Recordings never leave the local network.

---

## The Networking Work

Getting a wired PoE connection to the front door required running ethernet through the house, which is something I had never done before. This turned into a larger project: running ethernet from a central hub in the basement up through the walls to a Zyxel switch on the second floor, then distributing from there to devices throughout the house. The TP-Link PoE switch serves the doorbell as well as first floor living and media room drops, with unmanaged switches at those endpoints for connecting media devices.

This involved learning to:

- Plan cable runs through walls and between floors
- Crimp RJ45 connectors and terminate custom-length cables
- Install wall plates and keystone jacks
- Configure static IP assignments for PoE devices
- Manage switch port assignments for IoT isolation

The ethernet infrastructure that now runs through the house is now the physical foundation that will make a locally controlled home network possible.

---

## Integration with Home Security Lab

Frigate logs are ingested into the centralized logging pipeline documented in the [home-security-lab](https://github.com/CamBrown-GRC/home-security-lab) repo. Motion detection events and object recognition activity flow into Loki alongside firewall, DNS, and network traffic data, making camera activity queryable alongside the rest of the network's telemetry.

A dedicated Grafana dashboard for Frigate events is planned as part of Phase 3 of the home security lab.

---

## What This Enables

- **Full data ownership** — recordings are stored locally on TrueNAS and never transmitted to a third party
- **No subscription required** — no ongoing cloud storage cost, account to manage, or Terms of Service changes that affect what happens to footage
- **Network visibility** — camera traffic is visible in the home security monitoring pipeline like any other network device
- **IoT segmentation** — the doorbell camera sits on a segmented IoT network, isolated from trusted devices
- **Auditability** — all access to footage is local and logged; no third party can access recordings without physical access to the network

---

## Next Steps

- **Coral TPU** — adding a Google Coral USB accelerator to offload object detection from CPU, reducing latency and resource usage. This becomes more important as additional cameras are added.
- **Additional cameras** — expanding coverage to other areas of the house. The ethernet infrastructure is already in place for this.
- **Grafana dashboard** — dedicated dashboard for Frigate motion and detection events, integrated with the broader home security monitoring pipeline
- **Retention policy documentation** — documenting the TrueNAS retention configuration for reproducibility

---

_Part of a broader effort to build a privacy-first home network. See also: [home-security-lab](https://github.com/CamBrown-GRC/home-security-lab)_
