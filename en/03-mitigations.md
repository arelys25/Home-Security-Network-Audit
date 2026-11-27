# Mitigations and recommendations

## Redesigning the vulnerable IoT device's protocol

The two confirmed vulnerabilities in the core finding (see [`01-audit-report.md`](./01-audit-report.md)) are:

1. **Total absence of authentication** on the control channel.
2. **No protection against replay attacks** — the counter field exists in the protocol, but the receiver never validates it.

A reasonable but incorrect first instinct would be to "hide" or obfuscate fields like the magic number or the checksum. **This fixes nothing**: an attacker who has already captured a valid packet doesn't need to understand its structure to replay it as-is — hiding a field doesn't stop a replay or a forgery if the attacker only copies bytes it already saw work. This is the classic mistake of confusing **security through obscurity** with real authentication.

The actual solution combines three complementary pieces:

| Mechanism | What it replaces | What it achieves |
|---|---|---|
| **Pre-shared key**, established during initial pairing between the device and the legitimate app, never transmitted in each message | Nothing — it doesn't exist today | Gives each device a real secret that a passive network observer never sees |
| **HMAC** computed with that key, replacing the current checksum field | The simple additive checksum (sum mod 256), which anyone can compute with no secret | Without the key, it's computationally infeasible to generate a valid HMAC for a new or modified message — a forged packet is detected and rejected |
| **Stateful validation on the receiver**: the device remembers the last accepted counter value and rejects any packet with an equal or lower value | Nothing — the counter exists today but isn't validated | A packet captured once stops being reusable: the receiver, not the message format, is what prevents replay |

With all three together: an attacker who only listens to the network (passive observation) can neither forge new commands (no key) nor replay an old one (the counter was already used and the receiver remembers).

## Additional recommendations (general context, beyond this specific protocol)

- **Network segmentation for IoT.** Separate IoT devices from the rest of the home network via a VLAN or a dedicated guest network. This doesn't fix the protocol's vulnerability, but it drastically limits the blast radius if an IoT device is compromised: an attacker who already controls the bulb shouldn't be able to directly reach laptops, phones, or cameras on the main network from there.
- **Encrypted channel for local control.** Use TLS/DTLS (or an equivalent authenticated channel) for the LAN control channel, instead of a proprietary plaintext binary protocol over TCP.
- **Check whether a public CVE already exists** for the hardware/firmware family involved. This kind of cheap, unauthenticated WiFi control module is a known pattern in the low-cost IoT ecosystem — worth checking whether the finding is new or already documented, rather than treating it as an isolated discovery.
- **Update or replace the device if the manufacturer offers no fix.** If the firmware can't be updated remotely (common in this product category), the practical end-user mitigation is the network segmentation mentioned above, rather than waiting for a patch.

## General home network hygiene recommendations (from the full audit)

- **Metadata is still metadata, even when content is encrypted.** DNS and TLS SNI reveal which services are in use even when HTTPS protects the content. If privacy from other network members (or an ISP) is a concern, consider encrypted DNS (DoH/DoT) — though this alone doesn't hide the SNI unless ECH (*Encrypted Client Hello*) is also used when the server supports it.
- **Don't assume "filtered" means "secure."** A host with all ports filtered may simply have a firewall hiding the real state of its services, not necessarily an absence of vulnerable services.
- **Change the router admin panel's default credentials**, if that hasn't been done yet — outside the scope of this specific audit, but a standard recommendation that applies to any home network.
