# Detailed methodology

This document breaks down the commands and reasoning behind each step of the analysis. All examples use **sample** IP/MAC addresses (not the real ones from the audited network). For the full narrative of findings, see [`01-audit-report.md`](./01-audit-report.md).

## Work environment

- Linux (host), no virtual machine — ARP spoofing needs raw access to the Wi-Fi adapter, and bridged mode for a VM over Wi-Fi is unreliable due to limitations of the 802.11 standard itself.
- Wireshark with `dumpcap` using Linux capabilities (`cap_net_admin`, `cap_net_raw`) instead of running everything as `sudo`. Principle of least privilege: a small, auditable binary gets elevated privileges, while the protocol dissectors (which process untrusted network data) run without them.
- Nmap, TShark, `arpspoof` (from the `dsniff` package), Python 3 with the standard `socket` module.

## 1. Asset discovery

```bash
sudo nmap -sn 192.168.1.0/24
```

`-sn` is a "ping scan" — it doesn't touch any ports, it just confirms which hosts are active. The equivalent of "taking roll call": it shouldn't cause any perceptible interference on the network.

**Classify by MAC, not by IP.** With DHCP, a device's IP can change across reboots or lease renewals. The MAC (absent randomization) is stable. To distinguish a real factory MAC from one generated randomly by software (common on modern phones for privacy), compute the **universal/local (U/L) bit**: the second least-significant bit of the MAC's first octet.

```
First octet in binary: xxxxxx1x  →  U/L bit = 1  →  software-generated (random) MAC
First octet in binary: xxxxxx0x  →  U/L bit = 0  →  factory MAC (assigned by the manufacturer, real OUI)
```

Example (fictional): `3E:A1:9C:40:12:7B` → first octet `3E` = `00111110` in binary → second least-significant bit = `1` → random MAC.

## 2. Port fingerprinting

```bash
sudo nmap -sS -sV -p- 192.168.1.50
```

- `-sS`: SYN scan — sends only the first packet of the three-way handshake (SYN) and never completes the connection. Faster at scale and leaves less of a trail in the target's logs than a full-connect scan.
- `-sV`: attempts to identify the service/version behind each open port (banner grabbing + known fingerprint database).
- `-p-`: scans all 65,535 ports, not just the "common" ones.

**Important note on false positives:** Nmap identifies services by byte-pattern matching against a fingerprint database, not by semantic certainty. In this project, two fingerprints (a server-monitoring protocol and a strategy-game engine) turned out to be coincidental byte matches with no real relation to the devices' actual function. Rule of thumb: if the fingerprint doesn't fit the device's plausible *role* (e.g., an IoT client reporting as if it were a monitoring server), treat it as suspicious until verified.

For unknown services, the next step is direct inspection:

```bash
echo -n "<query phrase>" | nc -u -w2 192.168.1.50 <port>
```

## 3. ARP spoofing (authorized active interception)

> Requires explicit authorization from the owners of the intercepted devices. See [`scope-and-authorization.md`](./scope-and-authorization.md).

**Why it's needed:** on a switch (unlike a hub), each port only receives traffic addressed to its own MAC, plus broadcast/multicast. A machine connected to the same switch as a phone and an IoT device **cannot see** the unicast conversation between the two, unless it deliberately places itself in the middle.

**Mechanism:** ARP has no authentication — any machine can announce "I am this IP" and others believe it without verification. The attack consists of:

1. Enable kernel packet forwarding, so as not to cut off the victims' real connectivity (and avoid an accidental DoS):
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   ```
2. Poison victim A's ARP cache, impersonating victim B (in one terminal):
   ```bash
   sudo arpspoof -i <interface> -t <victim_A_IP> <victim_B_IP>
   ```
3. Poison victim B's ARP cache, impersonating victim A (in another terminal, simultaneously):
   ```bash
   sudo arpspoof -i <interface> -t <victim_B_IP> <victim_A_IP>
   ```
4. Capture with Wireshark in parallel, filtering by the IP of interest:
   ```
   ip.addr == <target_IP>
   ```

**The two `arpspoof` instances are independent attacks, one per direction of the conversation.** If one of the two has an error (a typo, for instance), only that half of the traffic is lost, not the whole interception — a useful differential diagnosis if a capture shows one-way traffic only.

### Environment issues that can look like an "attack failure"

1. **The attacking machine's own firewall.** A host firewall enabled by default (e.g., `ufw` on Debian/Ubuntu-based distros) typically ships with `iptables`'s `FORWARD` chain set to `DROP` policy. This silently blocks forwarding third-party packets through the attacking machine, **with no visible error** — it looks like the attack "isn't working" or the victim "lost internet," when in reality the attacking machine never forwards what it intercepts.

   Diagnosis:
   ```bash
   sudo iptables -L FORWARD -n -v
   ```
   If the policy is `DROP` and packets are counted there, that's the cause. Specific fix (without disabling the firewall entirely), for the `ufw` tool:
   ```bash
   sudo ufw route allow in on <interface> out on <interface>
   ```

2. **ICMP Redirect.** When a machine forwards a packet back out the same interface it arrived on (exactly what a MITM attack does), the Linux kernel automatically generates an ICMP Redirect message to the original sender, suggesting "talk directly, you don't need to go through me." If the victim heeds it, the attack silently breaks. Mitigation:
   ```bash
   sudo sysctl -w net.ipv4.conf.all.send_redirects=0
   sudo sysctl -w net.ipv4.conf.<interface>.send_redirects=0
   ```

3. **Incorrect restoration when stopping the attack.** When `arpspoof` is interrupted, the tool tries to restore the victims' real ARP cache. In some prior-error scenarios it can restore pointing to an all-zeros MAC instead of the real one, temporarily disconnecting the device. Cleanly restarting the process usually fixes this.

**Troubleshooting checklist when MITM "isn't working":**
- Is `ip_forward` set to 1?
- Do both `arpspoof` instances have the correct IPs (no typos)?
- Does the `iptables` `FORWARD` chain let traffic through (not just ICMP)?
- Were outgoing ICMP Redirects disabled?
- Does the problem affect both traffic directions equally, or just one? (a problem affecting only one direction is almost never "the victim winning an ARP race" — rule out your own firewall first)

## 4. Reverse-engineering an unknown binary protocol

General, reusable process, applied to this project's IoT control channel:

1. **Capture multiple samples** of the same message type, of different sizes if possible.
2. **Look for constant bytes** at the start of every message → candidate for a "magic number" (protocol identifier), not a credential.
3. **Look for fields that change predictably** between consecutive messages (e.g., a counter incrementing by 1) → usually sequence metadata, not authentication.
4. **Isolate the length field**, if one exists, by comparing messages with different payload sizes.
5. **For the trailing byte or "odd" block:** try the simplest hypothesis before a complex one. For a verification byte, suggested order of hypotheses:
   - XOR of all preceding bytes.
   - Sum of all preceding bytes, modulo 256 (simple additive checksum).
   - Only if none of the above fits several independent samples, consider a standard CRC or an HMAC (in which case the field *would* be a real security measure, not a cosmetic one).
6. **Confirm with at least 3–4 independent samples** before trusting a formula.
7. **Controlled experiment to identify a data field's meaning:** change a single variable at a time (e.g., one device action per capture) and compare which bytes appear only when that action occurred.
8. **Validate with an active proof of concept:** hand-build a message with the reconstructed fields and send it directly to the device. If the device responds/acts as expected, the protocol model is confirmed, not just a hypothesis about bytes.
9. **Test the protocol's limits, not just the happy path:** for example, send the same command with a sequence/counter field out of the expected order, to check whether the receiver validates that field or ignores it (a replay vulnerability if it's ignored).

### Proof-of-concept example (generic structure, without the real confirmed values)

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("<device_IP>", "<port>"))

# hand-built payload following the reconstructed frame format:
# MAGIC(4) + TYPE(2) + COUNTER(2) + LENGTH(2) + DATA(LENGTH) + CHECKSUM(1)
payload = bytes.fromhex("<reconstructed frame — real values omitted from this publication>")
sock.send(payload)

response = sock.recv(1024)
print(response.hex())
sock.close()
```

See [`exploit/README.md`](../exploit/README.md) for the full explanation of which parts of the real code were omitted from this repository, and why.

## 5. Metadata analysis in encrypted traffic

Without breaking any encryption, there are three typical sources of visible metadata even when content travels over HTTPS/TLS:

| Wireshark filter | What it reveals |
|---|---|
| `dns` | Resolved domains in plaintext (before the encrypted connection exists) |
| `tls.handshake.type == 1` | The SNI (*Server Name Indication*) field of the Client Hello — the first packet of any TLS connection, indicating which domain it's connecting to, unencrypted |
| Statistics → Conversations | Data volume and duration of each connection — suggests the type of activity (e.g., sustained high volume = likely video) without revealing specific content |

Complementary: Statistics → Protocol Hierarchy gives an aggregate view of which protocols dominate a full capture — useful for spotting anomalies (industrial or old game protocols appearing on a normal phone are almost always dissector false positives, not real findings; confirm or rule them out by following the corresponding stream with "Follow Stream").
