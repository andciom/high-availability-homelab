# Architecture

## Overview

The **Highly Available Secure Homelab Infrastructure** project is a multi-host Linux infrastructure environment designed to provide resilient DNS resolution, encrypted DNS services, HTTPS reverse proxying, containerized applications, monitoring, and supporting infrastructure services.

The environment is built primarily around two single-board computers:

- **Raspberry Pi 5** — Primary infrastructure host
- **Orange Pi 4 Pro** — Secondary infrastructure and redundancy host

Critical network services are distributed between both systems so that the failure or maintenance of a single host does not completely eliminate DNS availability for clients on the network.

The project emphasizes:

- High availability
- DNS resiliency
- Secure name resolution
- Containerization
- Service isolation
- TLS encryption
- Infrastructure documentation
- Backup and recovery
- Multi-host administration

---

## Design Goals

The infrastructure was designed around several primary goals.

### High Availability

Critical network services should not depend entirely on a single physical host.

DNS infrastructure is therefore deployed across both the Raspberry Pi and Orange Pi so that clients have access to more than one resolver.

### Secure DNS Resolution

DNS traffic is protected and validated through technologies including:

- AdGuard Home
- Unbound
- DNSSEC
- DNS-over-TLS
- DNS-over-HTTPS

The design reduces dependency on external public recursive DNS providers by using local recursive resolution through Unbound.

### Containerized Infrastructure

Most infrastructure services run as Docker containers.

Containerization provides:

- Reproducible deployments
- Dependency isolation
- Easier upgrades
- Simplified configuration management
- Persistent storage separation
- Easier backup and recovery

### Service Redundancy

Services that are important to basic network functionality are duplicated when practical.

The primary example is DNS, which is provided independently by both infrastructure hosts.

### Secure Internal Service Access

Internal web applications are accessed through HTTPS reverse proxying rather than exposing every application's native management port directly to clients.

### Recoverability

Configuration files, persistent application data, and infrastructure documentation are maintained so that services can be reconstructed following hardware, storage, or software failure.

---

# Physical Architecture

The environment consists of the following major components:

```text
                         Internet
                            │
                            │
                            ▼
                    ┌───────────────┐
                    │ Home Router   │
                    │ / Firewall    │
                    └───────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
     ┌─────────────────┐        ┌─────────────────┐
     │ Raspberry Pi 5  │        │ Orange Pi 4 Pro│
     │                 │        │                 │
     │ Primary         │        │ Secondary       │
     │ Infrastructure  │        │ Infrastructure  │
     └────────┬────────┘        └────────┬────────┘
              │                          │
              └────────────┬─────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Local NAS       │
                  │ Backup Storage  │
                  └─────────────────┘
```

---

# Hardware

## Raspberry Pi 5

The Raspberry Pi functions as the primary infrastructure host.

### Hardware

- Raspberry Pi 5
- 16 GB RAM
- SSD-based operating system storage
- Gigabit Ethernet

### Primary Responsibilities

The Raspberry Pi provides several central infrastructure services, including:

- Primary DNS filtering
- Recursive DNS resolution
- DNS-over-TLS
- DNS-over-HTTPS
- HTTPS reverse proxying
- Container administration
- Logging and monitoring
- Supporting homelab applications

Because this host provides several central services, critical network functionality such as DNS is duplicated on the Orange Pi.

---

## Orange Pi 4 Pro

The Orange Pi functions as the secondary infrastructure host and provides redundancy for critical services.

### Hardware

- Orange Pi 4 Pro
- 12 GB RAM
- NVMe-based operating system storage
- Gigabit Ethernet

### Primary Responsibilities

The Orange Pi provides:

- Secondary DNS filtering
- Independent recursive DNS resolution
- Secondary DNS-over-TLS service
- Configuration synchronization
- Container management agents
- Supporting infrastructure services

The Orange Pi is designed to remain independently capable of answering DNS queries if the Raspberry Pi becomes unavailable.

---

## Network Attached Storage

A Western Digital NAS provides local network storage used for infrastructure backups.

The NAS is separate from both compute hosts, allowing backup data to remain available even if the storage device of one infrastructure server fails.

---

# Logical Architecture

At a high level, the infrastructure can be divided into several functional layers.

```text
┌─────────────────────────────────────────────┐
│                Client Layer                 │
│                                             │
│ Windows • macOS • Linux • Mobile • IoT      │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│               Network Layer                 │
│                                             │
│ Router • DHCP • DNS Distribution            │
└──────────────────────┬──────────────────────┘
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
┌─────────────────────┐ ┌─────────────────────┐
│ Raspberry Pi        │ │ Orange Pi           │
│                     │ │                     │
│ AdGuard Home        │ │ AdGuard Home        │
│ Unbound             │ │ Unbound             │
│ HAProxy             │ │ HAProxy             │
└─────────┬───────────┘ └──────────┬──────────┘
          │                        │
          └───────────┬────────────┘
                      ▼
             Recursive DNS
                  Resolution
```

---

# DNS Architecture

DNS is one of the most important components of the environment and is designed with redundancy in mind.

Each infrastructure host contains its own DNS stack.

```text
             PRIMARY DNS STACK

Client
   │
   ▼
AdGuard Home
   │
   ▼
Unbound
   │
   ▼
Authoritative DNS Infrastructure


            SECONDARY DNS STACK

Client
   │
   ▼
AdGuard Home
   │
   ▼
Unbound
   │
   ▼
Authoritative DNS Infrastructure
```

Neither DNS server depends on the other server for recursive resolution.

This means that if either physical host becomes unavailable, the remaining host can continue resolving DNS independently.

---

## AdGuard Home

AdGuard Home operates as the client-facing DNS service.

Its responsibilities include:

- DNS filtering
- Blocking known advertising and tracking domains
- Local DNS records
- DNS query logging
- Client request handling
- Forwarding unresolved requests to Unbound

Clients communicate with AdGuard Home rather than directly communicating with Unbound.

---

## Unbound

Unbound functions as the recursive DNS resolver.

Instead of forwarding all DNS queries to a public resolver such as Google or Cloudflare, Unbound recursively resolves DNS queries by communicating with the DNS hierarchy.

A typical resolution path is:

```text
Client
   │
   ▼
AdGuard Home
   │
   ▼
Unbound
   │
   ├── Root DNS Servers
   │
   ├── TLD Servers
   │
   └── Authoritative DNS Servers
   │
   ▼
Resolved Answer
```

This provides greater control over DNS resolution and reduces reliance on external recursive DNS providers.

---

## DNSSEC

DNSSEC validation is enabled through the recursive DNS infrastructure.

DNSSEC provides cryptographic validation of supported DNS records and helps protect against attacks involving forged DNS responses.

The resolver validates DNSSEC information before returning validated responses to clients.

---

# Encrypted DNS

The infrastructure supports encrypted DNS transports in addition to traditional DNS.

## DNS-over-TLS

HAProxy provides TLS termination for DNS-over-TLS.

```text
DNS Client
    │
    │ TLS / TCP 853
    ▼
 HAProxy
    │
    ▼
AdGuard Home
    │
    ▼
 Unbound
```

HAProxy handles the TLS connection and forwards the DNS request to the internal DNS service.

Both infrastructure hosts can provide this functionality.

---

## DNS-over-HTTPS

DNS-over-HTTPS is provided through the HTTPS reverse proxy.

```text
Client
   │
   │ HTTPS
   │ /dns-query
   ▼
 Caddy
   │
   ▼
AdGuard Home
   │
   ▼
Unbound
```

This allows supported clients to perform DNS queries through encrypted HTTPS connections.

---

# Reverse Proxy Architecture

Caddy functions as the primary reverse proxy for internal web-based services.

Instead of accessing applications using combinations of IP addresses and ports such as:

```text
http://server-address:port
```

services can be accessed through internal DNS hostnames and HTTPS.

Conceptually:

```text
Browser
   │
   │ HTTPS
   ▼
Caddy Reverse Proxy
   │
   ├── AdGuard Home
   ├── Portainer
   ├── Monitoring Services
   ├── Automation Services
   └── Other Internal Applications
```

This provides:

- Centralized HTTPS
- Easier service access
- Cleaner internal DNS names
- Reduced need to expose application ports directly
- Centralized TLS certificate management

---

# TLS Certificate Architecture

TLS certificates are centrally managed for internal HTTPS and encrypted DNS services.

A wildcard certificate is used for the internal service namespace.

Conceptually:

```text
Wildcard TLS Certificate
          │
          ├──────────► Caddy
          │
          └──────────► HAProxy
```

The certificate required by the secondary infrastructure host is securely synchronized from the primary host.

Private certificate material is **not stored in this GitHub repository**.

Certificate files, API credentials, and private keys are excluded from version control.

---

# Container Architecture

Most services are deployed through Docker and Docker Compose.

Each host maintains its own Docker environment.

```text
Raspberry Pi
│
├── Docker Engine
│
├── DNS Stack
│   ├── AdGuard Home
│   ├── Unbound
│   └── HAProxy
│
├── Reverse Proxy
│   └── Caddy
│
├── Management
│   ├── Portainer
│   └── Dozzle
│
└── Additional Services


Orange Pi
│
├── Docker Engine
│
├── DNS Stack
│   ├── AdGuard Home
│   ├── Unbound
│   └── HAProxy
│
├── Synchronization
│   └── AdGuardHome-Sync
│
├── Management
│   └── Portainer Agent
│
└── Additional Services
```

The infrastructure intentionally avoids duplicating every application.

Services are duplicated primarily when redundancy provides an operational benefit.

---

# AdGuard Configuration Synchronization

The two AdGuard Home servers operate independently but share configuration through **AdGuardHome-Sync**.

```text
Primary AdGuard Home
        │
        │ Configuration Sync
        ▼
AdGuardHome-Sync
        │
        ▼
Secondary AdGuard Home
```

Synchronization helps maintain consistent settings between DNS servers while preserving independent DNS processing on both hosts.

Examples of synchronized configuration may include:

- Filtering rules
- Custom DNS configuration
- Client settings
- General AdGuard settings

Certain host-specific values may remain unique between systems.

---

# High Availability Design

The primary high-availability objective of the current architecture is maintaining DNS availability.

The router distributes multiple DNS resolver addresses to clients.

```text
                     Client
                       │
              DNS Server Configuration
                       │
            ┌──────────┴──────────┐
            ▼                     ▼
     Primary DNS             Secondary DNS
     Raspberry Pi             Orange Pi
            │                     │
            ▼                     ▼
     AdGuard + Unbound      AdGuard + Unbound
```

If one infrastructure host becomes unavailable, clients can continue using the other DNS resolver.

---

## Failure Scenarios

### Raspberry Pi Failure

If the Raspberry Pi becomes unavailable:

- Primary DNS becomes unavailable.
- Secondary DNS remains operational.
- Orange Pi continues performing recursive DNS resolution.
- Services hosted only on the Raspberry Pi become unavailable.

### Orange Pi Failure

If the Orange Pi becomes unavailable:

- Secondary DNS becomes unavailable.
- Raspberry Pi continues servicing DNS requests.
- Primary reverse proxy and infrastructure services remain operational.

### NAS Failure

If the NAS becomes unavailable:

- Active infrastructure services continue operating.
- Local backups become temporarily unavailable.
- Compute and DNS services remain unaffected.

---

# Service Availability Scope

The term **highly available** in this project primarily refers to critical network infrastructure that has intentionally been duplicated.

Not every application in the homelab is currently redundant.

| Service | Redundant |
|---|---|
| AdGuard Home | Yes |
| Unbound | Yes |
| DNS Resolution | Yes |
| DNS-over-TLS | Yes |
| AdGuard Configuration | Synchronized |
| Caddy Reverse Proxy | Architecture evolving |
| NAS Storage | No |
| Individual Application Containers | Usually no |

This distinction is intentional.

Duplicating every container would consume additional resources without necessarily providing meaningful operational benefit.

Services are duplicated based on their importance to overall infrastructure availability.

---

# Network Design

Infrastructure hosts use stable network addressing so that clients and infrastructure services can reliably locate them.

The public repository intentionally does not document production IP addresses.

Example addressing used in documentation:

```text
Router:               192.168.x.1

Raspberry Pi:         192.168.x.10
Orange Pi:            192.168.x.11
NAS:                  192.168.x.20

Primary DNS:          192.168.x.10
Secondary DNS:        192.168.x.11
```

Actual production addressing is intentionally excluded from the repository.

---

# Service Naming

Internal services use DNS names rather than requiring users to remember IP addresses and application ports.

Example:

```text
service.lab.example.com
```

instead of:

```text
https://192.168.x.x:9443
```

This improves:

- Usability
- TLS certificate management
- Infrastructure organization
- Service migration flexibility
- Documentation clarity

The actual domain may be replaced with example values in public configuration files where appropriate.

---

# Docker Networking

Docker networks are used to allow containers to communicate internally without exposing every service directly to the physical network.

A typical service path is:

```text
Client
   │
   ▼
Caddy
   │
   │ Docker Network
   ▼
Application Container
```

Where possible, backend application ports remain accessible only within Docker networks.

Only services requiring direct client access are published to the host network.

This reduces unnecessary network exposure.

---

# Management Architecture

Infrastructure management is distributed across several tools.

## Portainer

Portainer provides centralized Docker administration.

The primary Portainer instance can manage containers running locally and connect to remote container hosts through the Portainer Agent.

```text
Portainer
   │
   ├── Raspberry Pi Docker
   │
   └── Orange Pi
          │
          ▼
     Portainer Agent
```

---

## Dozzle

Dozzle provides browser-based Docker container log viewing.

Multi-host support allows logs from infrastructure systems to be viewed through a centralized interface.

---

# Backup Architecture

Infrastructure configuration and application data are backed up separately from the active services.

```text
Raspberry Pi ──────┐
                   │
                   ├────► Backup Process
                   │
Orange Pi ─────────┘
                           │
                           ▼
                     Local NAS Storage
                           │
                           ▼
                     Off-Site Backup
```

Backups focus on data that cannot simply be recreated by pulling container images.

Examples include:

- Docker Compose files
- Application configuration
- Persistent Docker data
- Reverse proxy configuration
- DNS configuration
- Automation scripts
- Documentation

Container images themselves generally do not require backup because they can be pulled again from their respective registries.

Detailed backup and recovery procedures are documented separately in:

```text
docs/backup-restore.md
```

---

# Security Architecture

Security is incorporated into several layers of the infrastructure.

```text
                 Security Layers
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
   Network          Transport        Application
   Security         Security          Security
       │               │                │
   Firewall          HTTPS          AdGuard
   Isolation          DoT           DNS Filtering
   Limited Ports      DoH           Authentication
                      TLS
```

Security controls include:

- DNSSEC validation
- DNS-over-TLS
- DNS-over-HTTPS
- HTTPS for internal services
- TLS certificates
- Local recursive DNS
- Restricted port exposure
- Docker network isolation
- Environment variables for sensitive configuration
- Exclusion of credentials from Git
- Backup encryption
- Separate infrastructure hosts

Additional security details are documented in:

```text
docs/security.md
```

---

# Repository Architecture

The GitHub repository mirrors the logical infrastructure design where practical.

```text
high-availability-homelab/
│
├── README.md
├── LICENSE
├── .gitignore
│
├── docs/
│   ├── architecture.md
│   ├── deployment.md
│   ├── dns-flow.md
│   ├── failover.md
│   ├── security.md
│   └── backup-restore.md
│
├── raspberry-pi/
│   ├── docker-compose.yml
│   ├── caddy/
│   ├── haproxy/
│   ├── adguard/
│   └── unbound/
│
├── orange-pi/
│   ├── docker-compose.yml
│   ├── haproxy/
│   ├── adguard/
│   ├── unbound/
│   └── adguard-sync/
│
├── scripts/
│   ├── health-check.sh
│   ├── dns-test.sh
│   └── certificate-sync.sh
│
└── config/
    ├── .env.example
    └── hosts.example
```

Production secrets and sensitive files are intentionally excluded.

---

# Known Limitations

This project is a homelab environment designed for experimentation, learning, and practical infrastructure experience.

It should not be interpreted as a complete enterprise high-availability architecture.

Current limitations include:

- Some application services remain single-host services.
- Local NAS storage is not currently redundant.
- Certain management components may still depend on the primary infrastructure host.
- Client DNS failover behavior can vary by operating system and network implementation.
- Hardware redundancy is limited to the equipment available within the homelab.
- Some services require manual recovery or intervention following a host failure.

Documenting these limitations is intentional and helps identify areas for future improvement.

---

# Future Improvements

The architecture continues to evolve.

Potential future improvements include:

- Highly available reverse proxy services
- Virtual IP-based service failover
- Automated health checks
- Automated failover testing
- Infrastructure deployment with Ansible
- Infrastructure monitoring and alerting
- Centralized metrics collection
- Additional network segmentation
- Improved VLAN architecture
- Automated certificate synchronization
- Automated backup verification
- Infrastructure-as-code deployment
- Expanded disaster recovery testing

---

# Summary

This architecture provides a practical multi-host infrastructure platform focused on secure networking, containerization, service redundancy, and recoverability.

The Raspberry Pi serves as the primary infrastructure platform while the Orange Pi provides independent redundancy for critical DNS services.

Together, the environment demonstrates practical experience with:

- Linux administration
- Docker
- Docker Compose
- DNS
- Recursive DNS resolution
- DNSSEC
- DNS-over-TLS
- DNS-over-HTTPS
- HAProxy
- Caddy
- TLS
- High availability
- Reverse proxying
- Network storage
- Backup and recovery
- Multi-host infrastructure management

The project is intentionally designed as an evolving environment where new technologies and infrastructure improvements can be implemented, tested, documented, and evaluated.