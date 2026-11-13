# Day 12: Wireshark Protocol Analysis

Captured on the `Ethernet` adapter of the Win11 lab VM (192.168.40.101), behind the pfSense LAN (192.168.40.1). TCP and HTTP tests used `http://httpforever.com` (104.21.4.210), which stays on plain HTTP (port 80).

## ARP Analysis

**Filter:** `arp`. **Generated with** (admin PowerShell): `arp -d *`, then `ping 192.168.40.1`.

| Device | IP | MAC |
|---|---|---|
| Win11 VM (requester) | 192.168.40.101 | 08:00:27:97:3a:7e |
| pfSense LAN (gateway) | 192.168.40.1 | 08:00:27:1b:9b:21 |

| Frame | Time (s) | Source | Destination | Info |
|---|---|---|---|---|
| 2 | 3.2557 | 08:00:27:97:3a:7e | Broadcast | Who has 192.168.40.1? Tell 192.168.40.101 |
| 3 | 3.2562 | 08:00:27:1b:9b:21 | 08:00:27:97:3a:7e | 192.168.40.1 is at 08:00:27:1b:9b:21 |

![ARP request and reply](../screenshots/day12-arp.png)

**Key points**
- **Opcode** `1` = request, `2` = reply. The request's target MAC is `00:00:00:00:00:00` because that is what it asks for.
- The request is a **broadcast**; the reply is **unicast**. ARP is Layer 2 and only works on the **same subnet**.
- MAC prefix `08:00:27` is the VirtualBox virtual NIC vendor.
- ARP has **no authentication**, which is why ARP spoofing works.

**Security relevance:** two MACs claiming one IP, many gratuitous replies, or bursts of requests sweeping a subnet can indicate spoofing or scanning.

---

## DNS Analysis

**Filter:** `dns`. **Generated with:** `nslookup github.com` (via the pfSense resolver, 192.168.40.1).

| Frame | Time (s) | Direction | Info |
|---|---|---|---|
| 82 / 83 | 235.4234 / 235.4245 | VM ↔ pfSense | PTR `1.40.168.192.in-addr.arpa` → `pfSense.home.arpa` |
| 84 / 86 | 235.4300 / 236.2496 | VM ↔ pfSense | A `github.com.home.arpa` → **No such name** |
| 89 | 236.2623 | VM → pfSense | Query `0x0004` **A `github.com`** |
| 90 | 237.1222 | pfSense → VM | Response `0x0004`: **`github.com` → 4.159.108.160** |
| 91 | 237.1286 | VM → pfSense | AAAA `github.com` (IPv6 lookup) |

![DNS query and response](../screenshots/day12-dns.png)

**Key points**
- **UDP**, server port `53`, random high client port. No handshake.
- The **transaction ID** (`0x0004`) matches response 90 to query 89. `A` = IPv4, `AAAA` = IPv6.
- Latency was about 0.86 s (cache miss); a cached repeat answers in about 1 ms.
- `nslookup` first does a **PTR lookup of the DNS server** (frames 82/83), then Windows tries the **search suffix** (`github.com.home.arpa`, NXDOMAIN) before the bare name.
- DNS is **plaintext**, so anyone on the path can see which domains a host looks up.

**Security relevance:** DNS logs reveal C2 domains; floods of NXDOMAIN suggest DGA malware; long or encoded subdomains suggest tunnelling; answers from an unexpected server suggest spoofing.

---

## DHCP Analysis

**Filter:** `bootp`. **Generated with** (admin PowerShell): `ipconfig /release`, then `ipconfig /renew`.

| Frame | Time (s) | Source → Destination | Packet | Transaction ID |
|---|---|---|---|---|
| 242 | 602.9424 | 192.168.40.101 → 192.168.40.1 | **Release** | `0x402d3ed1` |
| 258 | 607.5493 | 0.0.0.0 → 255.255.255.255 | **Discover** | `0x5dd6b0d4` |
| 260 | 608.5559 | 192.168.40.1 → 192.168.40.101 | **Offer** | `0x5dd6b0d4` |
| 261 | 608.5617 | 0.0.0.0 → 255.255.255.255 | **Request** | `0x5dd6b0d4` |
| 262 | 608.5636 | 192.168.40.1 → 192.168.40.101 | **ACK** | `0x5dd6b0d4` |

![DHCP exchange](../screenshots/day12-dhcp.png)

**Key points**
- Discover and Request come from `0.0.0.0` to the broadcast address, because the VM has no IP yet. One transaction ID ties the four DORA packets together. UDP ports: client `68`, server `67`.
- The Offer proposed `192.168.40.101` again (pfSense remembered the MAC).
- The exchange took about **1.0 s**, mostly Discover to Offer, because pfSense pinged the address first to check it was free (see ICMP).
- DHCP options include subnet mask, router, DNS server and lease time.

**Security relevance:** a **rogue DHCP server** (two Offers for one Discover) can push a malicious gateway or DNS; **starvation** floods Discovers from spoofed MACs; DHCP logs map an **IP to a device at a point in time**.

---

## ICMP Analysis

**Filter:** `icmp`. **Generated with:** `ping 192.168.40.1` and `ping 8.8.8.8`.

| Frame | Time (s) | Source → Destination | Info |
|---|---|---|---|
| 4 | 3.2562 | 192.168.40.101 → 192.168.40.1 | Echo request, id `0x0001`, seq 17, ttl 128 |
| 5 | 3.2567 | 192.168.40.1 → 192.168.40.101 | Echo reply, id `0x0001`, seq 17, ttl 64 |
| 1418 | 1474.1445 | 192.168.40.101 → 8.8.8.8 | Echo request, id `0x0001`, seq 21, ttl 128 |
| 1419 | 1474.3035 | 8.8.8.8 → 192.168.40.101 | Echo reply, id `0x0001`, seq 21, ttl 118 |

![ICMP echo](../screenshots/day12-icmp.png)

**Key points**
- **Type** `8` = Echo Request, `0` = Echo Reply. Identifier and sequence number match each reply to its request.
- **TTL:** Windows sends 128, pfSense (FreeBSD) replies 64, and 8.8.8.8 arrived at 118 (about 10 hops).
- **RTT:** about 0.5 ms to the gateway, about 159 ms to 8.8.8.8.
- **Payload:** 32 bytes (`abcdefghijklmnopqrstuvwabcdefghi`), the standard Windows ping pattern.
- **Frame 259:** pfSense sent an Echo request to the VM's address during DHCP to check it was free. No reply came, which explains the 1 s gap.
- **Frames 1226 to 1239:** Destination unreachable (Host unreachable) from pfSense, carrying the header of the failed packet.

**Security relevance:** sweeps of Echo Requests indicate host discovery; large or odd payloads can indicate tunnelling; bursts of Destination unreachable can reveal scanning; TTL helps estimate OS and distance.

---

## TCP Analysis

**Filters:** `tcp.flags.syn == 1 && ip.addr == 104.21.4.210`. **Generated by** browsing to `http://httpforever.com` (or `Test-NetConnection 104.21.4.210 -Port 80`).

| Step | Frame | Time (s) | Flags | Direction | Seq / Ack (relative) | Raw values |
|---|---|---|---|---|---|---|
| 1 | 5295 | 417.8564 | SYN | 192.168.40.101:60104 → 104.21.4.210:80 | Seq=0 | Seq `0xe082e728` |
| 2 | 5298 | 417.8636 | SYN, ACK | 104.21.4.210:80 → 192.168.40.101:60104 | Seq=0, Ack=1 | Seq `0xf7ff19f3`, Ack `0xe082e729` |
| 3 | `<fill in>` | | ACK | 192.168.40.101 → 104.21.4.210 | Seq=1, Ack=1 | Ack should be `0xf7ff19f4` |

The final ACK has no SYN flag, so find it with `tcp.port == 60104`.

| Option | Client SYN | Server SYN-ACK |
|---|---|---|
| Window | 65535 | 65535 |
| MSS | 1460 | 1400 |
| Window scale | 256 | 8192 |
| SACK_PERM | yes | yes |

The browser also opened a second port 80 connection and two port **443** connections to the same IP, probing for HTTPS.

![TCP handshake](../screenshots/day12-tcp-handshake.png)

**Key points**
- **Flags:** SYN (`0x002`), SYN+ACK (`0x012`), ACK, FIN, RST.
- Each side acknowledges the other's sequence number **plus 1** (server Ack `0xe082e729` = client Seq `0xe082e728` + 1). Wireshark shows **relative** numbers by default; real initial numbers are random.
- The server offered a smaller **MSS** (1400), and both sides agreed to use **SACK**.
- SYN to SYN-ACK took about **7.2 ms** (the round-trip time).

**Security relevance:** many SYNs with no final ACK suggest a **SYN flood** or half-open scan; an RST reply means the port is closed.

---

## UDP Analysis

**Filters:** `udp`, `udp.port == 53` (235 of 5909 packets). **Generated with:** `ipconfig /flushdns`, then `nslookup httpforever.com` and `nslookup github.com`.

| Frame | Time (s) | Info |
|---|---|---|
| 5866 | 894.0650 | Query `0x0004` A `httpforever.com` (**no response seen**) |
| 5867 / 5869 | 896.0773 / 897.8700 | AAAA `httpforever.com` → `2606:4700:3036::…` |
| 5876 / 5877 | 902.2860 / 903.1921 | A `github.com` → **20.26.156.215** |
| 5878 / 5879 | 903.1986 / 904.0358 | AAAA `github.com` → SOA only (no IPv6 address) |

![UDP DNS traffic](../screenshots/day12-udp-dns.png)

**Frame 5879 (AAAA response):** 154 bytes; pfSense `08:00:27:1b:9b:21` → VM `08:00:27:97:3a:7e`; 192.168.40.1 → 192.168.40.101, protocol `17`; UDP source port **53**, destination port **49956**, length 120 (8-byte header + 112-byte DNS message), checksum `0x56fe`; DNS ID `0x0005`, flags `0x8180` (response, no error).

**Key points**
- **No handshake:** one query, one response. TCP needed three packets before any data moved.
- **No built-in reliability:** the first `httpforever.com` query got no response, and `nslookup` moved on about 2 s later ("DNS request timed out"). The application decides whether to retry.
- The UDP header has four fields (**source port, destination port, length, checksum**), 8 bytes versus at least 20 for TCP.
- Transaction IDs restart in each `nslookup` run, so a query and response are matched by ID **and** client port.

| | TCP | UDP |
|---|---|---|
| Connection | Handshake first | None |
| Reliability | Acknowledged, retransmitted | Best effort |
| Typical use | HTTP, SMB, SSH | DNS, DHCP, streaming |

---

## HTTP and TLS (HTTPS) Analysis

**Filters:** `http`, `tls`. For HTTP I browsed to `http://httpforever.com`, then right-clicked the `GET /` packet → **Follow → TCP Stream**. The TLS example below is background Windows diagnostic traffic to a Microsoft service, but the handshake is the same.

**HTTP:** the request (`GET / HTTP/1.1`, `Host`, `User-Agent`) and response are fully **readable**.

![TCP stream](../screenshots/day12-tcp-stream.png)

**TLS 1.2 handshake:** VM `192.168.40.101:51344` → `20.42.65.92:443`, SNI `v20.events.data.microsoft.com`

| Frame | Time (s) | Direction | Length | Info |
|---|---|---|---|---|
| 7496 | 1305.1955 | VM → server | 266 | **Client Hello** (SNI) |
| 7500 | 1305.2846 | server → VM | 1454 | **Server Hello, Certificate, Server Key Exchange, Server Hello Done** |
| 7501 | 1305.2918 | VM → server | 212 | **Client Key Exchange, Change Cipher Spec, Encrypted Handshake Message** |
| 7502 | 1305.3964 | server → VM | 105 | **Change Cipher Spec, Encrypted Handshake Message** |
| 7503 on | 1305.3964 + | both | 92 to 2095 | **Application Data** (encrypted) |

![TLS handshake](../screenshots/day12-tls.png)

**Key points**
- The TLS handshake runs on top of the TCP handshake, on port `443`.
- The **Client Hello** sends the **SNI** in plaintext, plus supported ciphers and ALPN (`h2`, `http/1.1`).
- In TLS 1.2 the server's **certificate** is also sent in the clear. **Change Cipher Spec** marks the point where everything becomes encrypted.
- Client Hello to Server Hello took about **89 ms**; the full handshake about **201 ms**.
- In TLS 1.3 the certificate is encrypted too, but the SNI stays plaintext unless Encrypted Client Hello (ECH) is used.

**Key takeaway:** HTTP exposes everything (URLs, headers, form data, credentials). TLS encrypts the payload, but the handshake still leaks the **server name (SNI)** and certificate details. An observer can see **who** the VM talks to, but not **what** is said.

**Security relevance:** credentials over HTTP can be read by anyone on the path. For HTTPS, analysts use metadata (SNI, certificate, IP, packet sizes and timing) and watch for expired or self-signed certificates, name/SNI mismatches, very old TLS versions, or a missing SNI.

---

## SMB Analysis

**Filter:** `smb2` (56 of 1833 packets). Client: Win11 VM (192.168.40.101, port `57115`). Server: WinSrv-Lab (192.168.40.10, port `445`). WinSrv-Lab was first on a bridged network only, so I added an internal adapter with static IP `192.168.40.10` on the VM's network.

**Generated with**
- Server: `mkdir C:\TestShare`, a `test.txt` inside it, and `New-SmbShare -Name TestShare -Path C:\TestShare -FullAccess Everyone`
- VM: `net use * /delete`, `net use Z: \\192.168.40.10\TestShare`, `dir Z:\`, `type Z:\test.txt`, `"new file" | Out-File Z:\from-win11.txt` (using the IP forces NTLM)

Typical flow: Negotiate Protocol (agree on dialect) → Session Setup (NTLM / Kerberos) → Tree Connect (connect to share) → Create / Read / Write / Close.

| Frame | Time (s) | Direction | Info |
|---|---|---|---|
| 1794 | 1484.9439 | Server → VM | **Create Response**, `from-win11.txt` |
| 1795 / 1796 | 1484.9491 / 1484.9499 | both | **Write** request (length 22, offset 0) / response |
| 1797 / 1798 | 1484.9506 / 1484.9515 | both | **GetInfo** request / response |
| 1799 / 1802 | 1484.9518 / 1484.9884 | both | **Close** request / response |
| 1822 / 1823 | 1495.1465 / 1495.1471 | both | **Close** request / response, `test.txt` |
| 1824 / 1825 | 1495.1474 / 1495.1478 | both | **Tree Disconnect**, `\\192.168.40.10\IPC$` |

![SMB2 file operations](../screenshots/day12-smb.png)

Share path: `\\192.168.40.10\TestShare`. Dialect: `<fill in>` (expand **Negotiate Protocol Response** → SMB2 header → **Dialect**).

**Key points**
- Every file action is visible, so an analyst can see who touched which file and when.
- File names are **readable in the clear**, so this traffic was **not encrypted**. SMB 3.x supports encryption, but it is not always enabled.
- The 22-byte write matches the file I created (`Out-File` writes UTF-16). The write was answered in about 0.8 ms.
- `IPC$` is the hidden inter-process communication share Windows connects to alongside the real one.

**Security relevance:** SMB is used for **lateral movement** and ransomware spread. Watch for many Session Setup failures, access to admin shares (`C$`, `ADMIN$`), unusual workstation-to-workstation SMB, and one host rapidly writing many files.

---

## Host Filter and Evidence

- **Host filter:** `ip.addr == 192.168.40.101` shows every packet to or from the VM, the main filter for investigating one host.
- **Evidence file:** `wireshark/day12-mixed-capture.pcapng` (includes `httpforever.com` HTTP traffic), reviewed for credentials or personal browsing before committing.
- **Screenshots:** `day12-arp.png`, `day12-dns.png`, `day12-dhcp.png`, `day12-icmp.png`, `day12-tcp-handshake.png`, `day12-udp-dns.png`, `day12-tcp-stream.png`, `day12-tls.png`, `day12-smb.png`
