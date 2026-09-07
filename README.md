# Highly Available Secure Homelab Infrastructure

## Project Overview

As my homelab expanded, several services became increasingly important to the rest of the environment. DNS resolution, internal HTTPS access, reverse proxying, container networking, and remote administration were no longer isolated services—they had become part of the core infrastructure supporting everything else.

That created an important design problem: **what happens when one of those critical services or hosts becomes unavailable?**

This project focuses on designing and implementing a more resilient homelab infrastructure using multiple Linux-based single-board computers, containerized services, redundant DNS, recursive DNS resolution, internal HTTPS, and centralized service access.

Rather than treating the homelab as a collection of independent Docker containers, the goal was to design it more like a small production environment with consideration for **availability, security, fault tolerance, maintainability, and troubleshooting**.

---

## Project Goals

The primary goals of the project were to:

- Eliminate DNS as a single point of failure.
- Deploy redundant DNS services across separate physical hosts.
- Use local recursive DNS resolution instead of relying exclusively on public DNS resolvers.
- Validate DNSSEC-protected DNS responses.
- Provide human-readable internal DNS names for homelab services.
- Provide HTTPS access to internal applications without accessing them directly by IP address and port.
- Centralize reverse proxy configuration.
- Reduce unnecessary exposure of container ports to the local network.
- Improve visibility into Docker services and infrastructure health.
- Document failure scenarios and verify that redundancy works as expected.
- Create an architecture that can continue expanding as additional networking and security services are added.

---

## Hardware

The environment currently uses two primary Linux hosts:

### Raspberry Pi 5

- 16 GB RAM
- 512 GB SSD boot drive
- Docker Engine
- Docker Compose
- Primary application and infrastructure host

### Orange Pi 4 Pro

- 12 GB RAM
- 1 TB NVMe storage
- Debian Linux
- Docker Engine
- Docker Compose
- Secondary infrastructure and DNS host

Using two separate physical systems allows critical services to remain available even if one Docker host is restarted, updated, or becomes unavailable.

---

## Core Technologies

The project incorporates several technologies working together:

| Technology | Purpose |
|---|---|
| Docker | Containerized application hosting |
| Docker Compose | Declarative container deployment |
| AdGuard Home | Local DNS filtering and client DNS service |
| Unbound | Recursive DNS resolution |
| HAProxy | DNS traffic handling and infrastructure services |
| Caddy | HTTPS reverse proxy |
| DNSSEC | DNS response validation |
| TLS | Encrypted access to internal web applications |
| Portainer | Docker environment management |
| NetBox | Infrastructure documentation and IPAM |
| Tailscale | Secure remote-access experimentation |
| Git / GitHub | Configuration version control and project documentation |

---

## High-Level Architecture

At a high level, client devices send DNS queries to redundant DNS infrastructure hosted across the Raspberry Pi and Orange Pi.

```text
                    Internet
                       │
                 Home Router
                       │
              ┌────────┴────────┐
              │                 │
        Raspberry Pi        Orange Pi
              │                 │
        AdGuard Home        AdGuard Home
              │                 │
           Unbound            Unbound
              │                 │
              └────────┬────────┘
                       │
                Recursive DNS
                       │
                    Internet
```

Internal applications are accessed using DNS names rather than IP addresses and port numbers.

For example:

```text
portainer.lab.example.com
netbox.lab.example.com
n8n.lab.example.com
```

Those requests are resolved internally and forwarded through the reverse proxy:

```text
Client
  │
  ▼
Internal DNS
  │
  ▼
Caddy Reverse Proxy
  │
  ├──► Portainer
  ├──► NetBox
  ├──► n8n
  └──► Other Docker Services
```

This provides a much cleaner user experience than connecting directly to services using addresses such as:

```text
192.168.x.x:9443
192.168.x.x:8000
192.168.x.x:5678
```

It also allows backend application ports to remain unexposed when the application only needs to communicate with the reverse proxy through Docker networking.

---

# Design Decision 1: Redundant DNS

DNS quickly became one of the most important services in my homelab.

Once internal DNS was being used to access applications and hosts by name, losing the DNS server could make otherwise healthy infrastructure appear unavailable.

My original configuration relied on a single AdGuard Home instance running on the Raspberry Pi.

Although this worked, it created a clear single point of failure.

If the Raspberry Pi was:

- rebooted,
- undergoing maintenance,
- experiencing a Docker failure,
- disconnected from the network,

DNS resolution for the entire environment could be interrupted.

To address this, I deployed a second DNS stack on the Orange Pi.

The resulting design provides two independent DNS servers:

```text
              Client
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
 Primary DNS       Secondary DNS
 Raspberry Pi       Orange Pi
        │               │
   AdGuard Home     AdGuard Home
        │               │
     Unbound          Unbound
```

The router distributes both DNS server addresses to clients.

If the primary resolver becomes unavailable, clients can continue using the secondary resolver.

This changes the DNS architecture from:

```text
Clients ──► Raspberry Pi ──► DNS
```

into:

```text
                 ┌──► Raspberry Pi ──► DNS
Clients ─────────┤
                 └──► Orange Pi ─────► DNS
```

The key difference is that DNS availability is no longer dependent on a single physical host.

---

# Design Decision 2: Recursive DNS with Unbound

AdGuard Home provides filtering and DNS management, but I wanted the DNS infrastructure to perform its own recursive resolution rather than forwarding every request to a third-party resolver.

For that reason, each AdGuard Home instance forwards permitted DNS queries to a local Unbound resolver.

The request path becomes:

```text
Client
  │
  ▼
AdGuard Home
  │
  ├── Blocked domain → Reject
  │
  └── Allowed domain
           │
           ▼
        Unbound
           │
           ▼
       Root DNS
           │
           ▼
        TLD DNS
           │
           ▼
   Authoritative DNS
```

Instead of asking a public recursive resolver for the final answer, Unbound performs the DNS lookup process itself.

This provided an opportunity to better understand:

- DNS recursion
- root servers
- top-level-domain servers
- authoritative DNS
- DNS caching
- DNSSEC
- upstream and downstream DNS relationships

The project therefore became useful not only as infrastructure, but also as a practical networking lab.

---

# Design Decision 3: Internal DNS and HTTPS

Another objective was to stop accessing applications by remembering IP addresses and port numbers.

Instead of:

```text
https://192.168.x.x:9443
```

I wanted to use:

```text
https://portainer.lab.example.com
```

AdGuard Home provides the internal DNS records while Caddy handles the HTTPS connection and reverse proxies the request to the appropriate Docker container.

The traffic flow is:

```text
Browser
   │
   │ HTTPS
   ▼
portainer.lab.example.com
   │
   ▼
Internal DNS
   │
   ▼
Caddy
   │
   ▼
Docker Network
   │
   ▼
Portainer
```

Caddy and the backend containers share a Docker network, allowing Caddy to communicate with applications by their Docker DNS names.

For example:

```text
portainer:9443
netbox:8080
n8n:5678
```

This means many applications do not need their ports published directly to the host network.

---

# Security Considerations

Although this is a homelab environment, I wanted security decisions to be part of the design rather than something added later.

Some of the practices implemented or planned as part of the project include:

- Using HTTPS for internal web applications.
- Avoiding unnecessary Docker port exposure.
- Keeping API keys and credentials outside Git repositories.
- Using `.env` files for sensitive configuration.
- Maintaining sanitized example configuration files for GitHub.
- Using DNS filtering to reduce access to known advertising, tracking, and malicious domains.
- Using DNSSEC validation with Unbound.
- Segmenting trusted devices from IoT devices where practical.
- Limiting remote access to secure methods such as VPN-based connectivity.
- Maintaining backups of important service configuration.

No production credentials, TLS private keys, API tokens, passwords, or sensitive configuration are included in the public project repository.

---

# Testing the Design

A highly available design is only useful if the failure scenarios are actually tested.

One of the most important parts of this project is therefore deliberately creating failures and observing how the environment responds.

Planned and completed testing includes:

### Primary DNS Failure

Stop the primary AdGuard Home container:

```bash
docker compose stop adguard
```

Then verify that DNS resolution continues using the secondary server.

```bash
nslookup example.com
```

or:

```bash
dig example.com
```

### DNSSEC Validation

Verify that a correctly signed domain resolves normally while an invalid DNSSEC domain fails validation.

### Recursive DNS Testing

Inspect DNS resolution to verify that Unbound is performing recursive resolution rather than silently falling back to a public resolver.

### Reverse Proxy Testing

Verify that services remain accessible through their internal HTTPS hostnames.

### Docker Network Testing

Confirm that services without published host ports remain reachable through the reverse proxy while being inaccessible directly from unrelated network clients.

---

# Troubleshooting and Lessons Learned

One of the most valuable parts of building this environment has been troubleshooting the interactions between DNS, Docker networking, reverse proxies, TLS certificates, and Linux hosts.

Several issues encountered during the project included:

- HTTP 502 Bad Gateway responses from reverse-proxied applications.
- Docker containers being unable to resolve other container hostnames.
- Services attached to different Docker networks.
- TLS certificate path and permission problems.
- DNS resolution working correctly on Linux and macOS while behaving differently on Windows.
- Containers appearing healthy while the associated web application remained inaccessible.
- Reverse proxy upstream configuration errors.
- Hostnames resolving correctly from the host but not from inside containers.

These failures forced me to troubleshoot the infrastructure layer by layer rather than simply reinstalling services.

A typical troubleshooting process involved checking:

```text
DNS Resolution
      │
      ▼
Network Connectivity
      │
      ▼
Docker Networking
      │
      ▼
Application Port
      │
      ▼
Reverse Proxy
      │
      ▼
TLS
      │
      ▼
Client
```

This has been one of the most useful lessons from the project: a web application displaying an error does not necessarily mean the web application itself is the problem.

Understanding the complete traffic path makes troubleshooting significantly more systematic.

---

# Current Status

Current capabilities include:

- Multiple Linux Docker hosts.
- Redundant DNS servers.
- AdGuard Home DNS filtering.
- Unbound recursive DNS.
- DNSSEC validation.
- Internal DNS records.
- HTTPS reverse proxying.
- Docker service-to-service networking.
- Centralized container management.
- Infrastructure documentation.

Future improvements will include additional high-availability testing, monitoring, network segmentation, infrastructure automation, centralized logging, and security monitoring.

---

# Skills Demonstrated

This project demonstrates practical experience with:

- Linux administration
- TCP/IP networking
- DNS
- recursive DNS
- DNSSEC
- Docker
- Docker Compose
- container networking
- reverse proxies
- HTTP/HTTPS
- TLS certificates
- high availability
- fault tolerance
- infrastructure troubleshooting
- network security
- documentation
- Git
- GitHub


