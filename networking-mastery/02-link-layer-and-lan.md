# 2. Link Layer & LANs

Goal: how devices on the *same* local network actually reach each other — the layer everything else (IP, TCP, HTTP) sits on top of. This is mostly infrastructure-level knowledge with no direct HRNS code to point at (application code never touches this layer directly), but it's the necessary foundation for chapter 4 (routing/NAT) and chapter 15 (Docker networking) to make sense.

## 2.1 Ethernet & frames

**Ethernet** is the dominant wired LAN technology (Wi-Fi is its wireless cousin, using mostly-compatible framing). An **Ethernet frame** is the link-layer envelope wrapping every packet sent over a local network segment:

```
[ destination MAC | source MAC | type | payload (an IP packet) | checksum ]
```

The **NIC** (Network Interface Card) — physical hardware, or a virtual one for a VM/container — is what sends and receives these frames. Every NIC has a **MAC address** (Media Access Control address) burned in at manufacture: a 48-bit identifier like `3c:22:fb:1a:2b:9d`, unique per physical interface (in principle — virtualized/container NICs generate them on the fly, which is directly relevant in chapter 15).

## 2.2 Switches — how a LAN actually delivers a frame

A **switch** is the device a LAN's cables physically plug into. It learns which MAC address lives on which physical port (by watching source addresses on incoming frames) and forwards each frame *only* out the port where the destination MAC actually is — not to every device on the network. This matters:

```
Computer A
    │
    ▼
  Switch
  /    \
 ▼      ▼
PC B   PC C     ← a frame from A to B goes ONLY to B's port, not C's
```

Three terms worth having precise: **unicast** (one specific destination — the normal case), **broadcast** (every device on the local network segment — used by ARP, §2.3, and DHCP), **multicast** (a specific *group* of interested devices — less common in typical web app networking). A **collision domain** is the set of devices that could interfere if they transmitted simultaneously (switches eliminate this almost entirely versus older shared-medium hubs); a **broadcast domain** is the set of devices that *receive* a broadcast frame — everything on one LAN segment, by definition, unless separated by a router.

**A switch and a router do genuinely different jobs**, worth being precise about since the terms get used loosely: a switch moves frames *within* one LAN using MAC addresses; a router moves packets *between* different networks using IP addresses (chapter 4). Your home Wi-Fi box is almost always both, bundled into one physical device, which is exactly why the distinction gets blurry in casual conversation.

## 2.3 ARP — resolving an IP address to a MAC address

Every layer-3 (IP) send ultimately has to become a layer-2 (Ethernet) frame with an actual destination MAC address — but application code, and even the IP layer itself, only ever deals in IP addresses. **ARP** (Address Resolution Protocol) is the glue: a broadcast question, "who has this IP address?", answered by whichever device owns it.

```
Broadcast: "Who has 192.168.1.20?"
                      ↓
192.168.1.20 replies: "That's me — MAC xx:xx:xx:xx:xx:xx"
```

The asking device then caches that mapping (the **ARP cache/table**) for a while, so it doesn't have to broadcast-ask again for every single packet. IPv6 replaces ARP with a conceptually similar but more sophisticated protocol, **NDP** (Neighbor Discovery Protocol) — same job, IPv6-native design.

You will essentially never write application code that touches ARP directly — its relevance to you as a full-stack engineer is almost entirely diagnostic: if a device on your LAN is unreachable, "is ARP resolving?" is one of the lower-layer questions in the §1.5 elimination checklist, and tools like `arp -a` (or `ip neigh` on Linux, chapter 17) show you the current cache.

## 2.4 Where this shows up (indirectly) in this platform's world

There's genuinely no application-level code in `HRNS.Platform.Server`/`HRNS.Platform.UI` that operates at this layer — link-layer concerns are handled entirely by the OS/hypervisor/cloud fabric underneath the app. The two places this layer's *consequences* become visible to you as a developer:

- **Docker networking** (chapter 15): each container gets its own **virtual Ethernet interface** (`veth` pair) and its own MAC address on Docker's internal virtual switch (a "bridge" network) — the exact same switch/MAC/ARP concepts from this chapter, just implemented in software instead of physical hardware.
- **Cloud VPCs** (chapter 15): a cloud provider's "subnet" is, underneath the abstraction, still built from these same primitives (virtual switches, virtual NICs) — the cloud provider's dashboard hides ARP and MAC addresses from you entirely, but understanding they're still there underneath is what makes concepts like "why can't this security group reach that subnet" reason-about-able instead of magic.

## Checkpoint

1. Explain, in your own words, why a switch forwarding a frame only to the port with the matching MAC address is more efficient than a hub broadcasting every frame to every port.
2. If your laptop's ARP cache is stale (says a given IP address maps to a MAC address that's no longer accurate — e.g. a device on your network was replaced), what symptom would you actually observe, and at which layer would you initially suspect the problem before realizing it was ARP?
3. Look up (or recall) your own laptop's MAC address (`ipconfig /all` on Windows, `ip link` on Linux/WSL — chapter 17 covers these tools properly). Is it the same every time you check, and would it change if you connected via Wi-Fi vs. a wired connection?
