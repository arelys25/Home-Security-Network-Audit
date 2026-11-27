# Scope and authorization

This project was carried out on a real home network, with **prior, explicit, written authorization** from the family members who own the audited devices, signed before any capture or active interception began. The original consent document (in Spanish, with the family's real names and dates) is not published in this repository for privacy reasons; its content is summarized below.

## What was authorized

- Discovery scanning of the network (identifying which devices are connected).
- Traffic capture with Wireshark to analyze what kind of data flows.
- **ARP spoofing**: a technique that temporarily redirects a device's traffic through the auditor's machine before it continues on its normal path, in order to observe it. It was explicitly noted that internet might feel slightly slower during those moments, without it being expected to fully cut off.

## What could and couldn't be seen (explained to the family in non-technical terms)

Most of today's internet traffic (social media, streaming, banking, search engines) uses HTTPS encryption — the equivalent of a letter inside a tamper-proof sealed envelope: even though the letter physically passes through the interceptor's hands, the envelope can't be opened to read what's inside.

**What could be seen, even in apps that encrypt their content:**
- Which sites or apps a device connected to (the domain, not the specific page or post).
- Roughly how long a connection was active and how much data it transferred.

**What could never be seen:**
- The specific content viewed, posted, or searched for.
- The content of private messages.
- Passwords for HTTPS sites.

It's the equivalent of seeing a phone bill (who was called and for how long) rather than listening to the call.

Where complete, unencrypted information could actually be found was on devices with weak security by design — IP cameras, smart bulbs or plugs, printers — which sometimes send some of their traffic with no encryption at all. That was, in fact, one of the project's real goals: detect whether the house had any such device leaking information without the family knowing.

## Explicit commitments from the auditor

- Not storing, sharing, or using any passwords or personal data that appeared in the captures. Any sensitive data is hidden or removed before appearing in any report.
- Not modifying, blocking, or damaging any device.
- Nothing captured leaves the home network or gets shared outside the project — what is published publicly (this repository) went through an additional redaction process, described in the main README and in [`exploit/README.md`](../exploit/README.md).

## Timeframe and opt-out

Fieldwork was carried out in discrete sessions during July and August 2026, announced to the family in advance. Any family member had the option to exclude their device from the analysis with no consequences; that option remained open throughout the project.

## Note on this publication

This repository is a **sanitized** version of the original work, intended for an academic portfolio and methodology write-up. The following were removed or replaced with generic placeholders: real IP and MAC addresses, the Wi-Fi network name (SSID), the exact brand and model of the vulnerable IoT device, and any data that could identify family members or their specific usage habits (beyond what's strictly necessary to explain a finding, such as "a family member" instead of a name). See also the responsible disclosure note in the main README regarding the vulnerable IoT device.
