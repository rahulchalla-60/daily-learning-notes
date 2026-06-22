# Day 007 — Network Buzzwords

---

## 1. IP Address

**Every device on a network needs a unique address — that's an IP address.**

An IP (Internet Protocol) address is a unique identifier assigned to every device connected to a network. It's how the internet knows where to send data.

```
Examples:
  192.168.1.10   ← private (local network)
  8.8.8.8        ← public (Google's DNS server)
```

**Analogy:** Your home postal address. Without it, a delivery person (the internet) has no idea where to bring your package.

**In system design:** When you open YouTube, your device sends a request from its IP address to YouTube's server IP. The response comes back to your IP.

---

## 2. DNS — Domain Name System

**DNS translates human-readable names into IP addresses.**

Computers communicate using IP addresses. Humans remember names. DNS bridges that gap.

```
www.netflix.com
      ↓  DNS lookup
52.14.23.105
      ↓
Netflix Server
```

**Analogy:** Your phone contacts list. You tap "Mom" — your phone looks up the actual number behind the name. DNS does the same thing for websites.

> Without DNS, you'd have to type `142.250.80.46` instead of `google.com` every time.

### How a DNS Lookup Works
```
1. You type google.com
2. Browser checks local DNS cache
3. If not cached → asks DNS resolver
4. DNS resolver → finds IP address
5. Browser connects to that IP
```

---

## 3. Client and Server

**Client asks. Server answers.**

| Role | What It Does | Examples |
|---|---|---|
| **Client** | Sends requests | Browser, mobile app, desktop app |
| **Server** | Processes requests and returns responses | YouTube server, Amazon API, Netflix CDN |

**Analogy:** A restaurant. You (customer = client) place an order. The kitchen (server) prepares and delivers it.

```
Browser
   │  HTTP Request (GET /home)
   ▼
Server
   │  HTTP Response (HTML, JSON, etc.)
   ▼
Browser renders the page
```

---

## 4. Protocols

**A protocol is a set of rules that defines how devices communicate.**

Without agreed-upon rules, computers wouldn't understand each other — like two people speaking different languages.

**Analogy:** Traffic rules. Red = stop, green = go. Everyone follows the same rules, so traffic flows.

**Common protocols in system design:**

| Protocol | Purpose |
|---|---|
| TCP | Reliable data delivery |
| UDP | Fast, best-effort delivery |
| HTTP / HTTPS | Web browser ↔ server communication |
| WebSocket | Real-time, persistent two-way connection |
| gRPC | High-performance internal service communication |

---

## 5. TCP — Transmission Control Protocol

**TCP guarantees delivery. Every byte arrives, in order.**

TCP establishes a connection before sending data, confirms receipt of each packet, and retransmits anything that's lost.

**Analogy:** Sending important documents via registered courier — you get a delivery confirmation, and if lost, it's resent.

```
Sender → [data packet] → Receiver → [ACK: received] → Sender
                                         ↑
                             If no ACK → retransmit
```

| ✅ What TCP guarantees | ❌ What TCP sacrifices |
|---|---|
| Data arrives at destination | Speed (overhead from acknowledgements) |
| Data arrives in correct order | Latency (connection setup required) |
| Lost data is retransmitted | |

**Use TCP for:** web browsing, banking, email, file transfers — anything where correctness matters more than speed.

---

## 6. UDP — User Datagram Protocol

**UDP is fast but fires and forgets. No delivery guarantee.**

UDP sends packets without establishing a connection or confirming receipt. If a packet is lost, it's gone.

**Analogy:** A live radio broadcast. If you miss a word, the station doesn't repeat it. The show goes on.

| ✅ What UDP gives you | ❌ What UDP doesn't guarantee |
|---|---|
| Very low latency | Delivery |
| High throughput | Ordering |
| No connection overhead | Retransmission |

**Use UDP for:** online gaming, video calls, live streaming, DNS queries — anything where speed matters more than perfection.

### TCP vs UDP — Side by Side

| Feature | TCP | UDP |
|---|---|---|
| Reliable delivery | ✅ Yes | ❌ No |
| Ordered packets | ✅ Yes | ❌ No |
| Speed | Slower | Faster |
| Retransmission | ✅ Yes | ❌ No |
| Best for | Banking, email, web | Gaming, streaming, calls |

---

## 7. HTTP — HyperText Transfer Protocol

**HTTP is the protocol your browser uses to talk to web servers.**

Every time you open a website, your browser sends an HTTP request. The server sends back an HTTP response.

**Analogy:** A restaurant order slip. Customer writes the order (request). Kitchen sends food back (response).

```
Browser  →  GET /profile HTTP/1.1   →  Server
         ←  200 OK { "name": "Rahul" }  ←
```

### HTTP Methods

| Method | Purpose |
|---|---|
| GET | Retrieve data |
| POST | Submit / create data |
| PUT | Update data |
| DELETE | Remove data |

### HTTP vs HTTPS

HTTPS = HTTP + TLS encryption. Data is encrypted in transit so no one can intercept it. Always use HTTPS in production.

---

## 8. WebSocket

**WebSocket keeps a connection open so both sides can send messages anytime.**

Standard HTTP is request-response — the client asks, the server answers, connection closes. WebSocket keeps the channel open, allowing real-time two-way communication.

**Analogy:**
```
HTTP:      Call → talk → hang up → call again → talk → hang up
WebSocket: Call once → keep the line open → talk any time
```

```
HTTP:      Client ──request──► Server
                  ◄──response──

WebSocket: Client ◄────────────► Server  (both directions, anytime)
```

### HTTP vs WebSocket

| Feature | HTTP | WebSocket |
|---|---|---|
| Connection | Opens and closes per request | Stays open |
| Direction | One-way (request/response) | Bidirectional |
| Real-time | ❌ Poor | ✅ Excellent |
| Chat apps | ❌ Inefficient | ✅ Perfect |
| Live updates | ❌ Requires polling | ✅ Server pushes instantly |

**Use WebSocket for:** chat apps (WhatsApp, Slack), live stock prices, sports scores, multiplayer games, collaborative tools.

---

## 9. Forward Proxy

**A forward proxy sits in front of clients and hides their identity from the internet.**

The server on the other end sees the proxy's IP, not the original client's.

```
Client → Forward Proxy → Internet → Server
                 ↑
         Server only sees proxy, not client
```

**Analogy:** An assistant who sends emails on your behalf. The recipient sees the assistant's address, not yours.

**Common uses:**
- Hide client IP for privacy
- Bypass geo-restrictions (VPN works this way)
- Content filtering in corporate networks
- Caching frequently accessed content for an office

**Example:** Company employees route all internet traffic through a corporate proxy for monitoring and filtering.

---

## 10. Reverse Proxy

**A reverse proxy sits in front of servers and hides them from clients.**

Clients talk to the reverse proxy — they have no idea which server actually handled the request.

```
Client → Reverse Proxy → Server 1
                       → Server 2
                       → Server 3
         ↑
  Client only sees the proxy
```

**Analogy:** A hospital reception desk. Patients talk to reception — reception routes them to the right doctor. Patients never go directly to a doctor's office.

**Common uses:**
- Load balancing across multiple servers
- SSL/TLS termination (handles HTTPS so servers don't have to)
- Caching static content
- Hiding server infrastructure from the public
- DDoS protection

**Tools:** Nginx, HAProxy, AWS ALB, Cloudflare

---

## Forward Proxy vs Reverse Proxy

| | Forward Proxy | Reverse Proxy |
|---|---|---|
| Sits in front of | Clients | Servers |
| Hides | Client identity | Server identity |
| Who benefits | Client | Server / infrastructure |
| Common use | VPN, corporate filtering | Load balancing, SSL termination |
| Example | Corporate internet gateway | Nginx before app servers |

```
Forward:  Client → [Proxy] → Internet

Reverse:  Internet → [Proxy] → Servers
```

---

## Quick Reference — All Terms

| Term | One-Line Definition |
|---|---|
| IP Address | Unique address of a device on a network |
| DNS | Translates domain names into IP addresses |
| Client | The machine that sends requests |
| Server | The machine that handles requests and returns responses |
| Protocol | A set of rules for how devices communicate |
| TCP | Reliable, ordered, connection-based protocol |
| UDP | Fast, connectionless, no delivery guarantee |
| HTTP | Protocol for browser ↔ server communication |
| HTTPS | HTTP with TLS encryption |
| WebSocket | Persistent, bidirectional real-time connection |
| Forward Proxy | Proxy in front of clients — hides client identity |
| Reverse Proxy | Proxy in front of servers — hides server, adds load balancing |

---

## One-Line Formula

```
IP → identifies devices
DNS → maps names to IPs
TCP → reliable delivery
UDP → fast delivery
HTTP → web communication
WebSocket → real-time communication
Forward Proxy → protects clients
Reverse Proxy → protects and scales servers
```
