# 🛡️ Personal Homelab & Cybersecurity Infrastructure

An Infrastructure as Code (IaC) repository for my self-hosted home laboratory. Built to replace proprietary cloud services (iCloud) while maintaining strict security controls, reverse proxying, and active intrusion prevention.

## 🏗️ System Architecture

- **Hypervisor:** Proxmox VE 8.2 (Bare-metal on Lenovo IdeaPad)
- **Compute:** Ubuntu Server 24.04 LTS (Isolated VM)
- **Storage:** MergerFS pool pooling multiple USB drives into a single logical volume.
- **Routing & DNS:** Pi-hole (LXC Container) for local DNS resolution and ad-blocking.

```mermaid
graph TD
    %% Style Definition (Colors)
    classDef internet fill:#1a1a1a,stroke:#333,stroke-width:2px,color:#fff
    classDef router fill:#005c99,stroke:#003d66,stroke-width:2px,color:#fff
    classDef lxc fill:#2eb82e,stroke:#1f7a1f,stroke-width:2px,color:#fff
    classDef vm fill:#4d0099,stroke:#26004d,stroke-width:2px,color:#fff
    classDef docker fill:#0db7ed,stroke:#0991bd,stroke-width:2px,color:#fff
    classDef storage fill:#ffcc00,stroke:#b38f00,stroke-width:2px,color:#000
    classDef security fill:#cc0000,stroke:#800000,stroke-width:2px,color:#fff

    %% Nodi Esterni
    Internet((Internet / 4G)):::internet
    DuckDNS[DuckDNS.org<br/>Dynamic DNS]:::internet

    %% Rete Domestica
    Router[TIM Hub Router<br/>192.168.1.1]:::router

    %% Proxmox Host
    subgraph PVE [Proxmox VE Host 192.168.1.254]
        
        PiHole[Pi-hole LXC<br/>192.168.1.253<br/>DNS & AdBlock]:::lxc

        subgraph VM [Ubuntu Server VM 192.168.1.252]
            
            %% Storage & Security (Host-level)
            CrowdSec{{CrowdSec IPS / Firewall}}:::security
            MergerFS[(MergerFS Pool 3TB<br/>/mnt/storage)]:::storage
            
            %% Docker Network
            subgraph Docker [Docker Container Network]
                DuckDNSContainer[DuckDNS Updater]:::docker
                NPM[Nginx Proxy Manager<br/>Ports 80/443]:::docker
                
                subgraph ImmichStack [Immich Stack]
                    ImmichServer[Immich Server]:::docker
                    ImmichML[Immich ML Engine]:::docker
                    DB[(Postgres + Redis<br/>SSD NVMe 40GB)]:::storage
                end
            end
        end
    end

    %% External Traffic Flow
    Internet -- HTTPS (Port 443) --> Router
    Router -- Port Forwarding --> CrowdSec
    CrowdSec -- Secure Traffic --> NPM
    NPM -- Reverse Proxy --> ImmichServer
    
    %% Immich Internal Interactions
    ImmichServer -.-> ImmichML
    ImmichServer --> DB
    ImmichServer --> MergerFS
    
    %% Updates and SSL
    DuckDNSContainer -. Update Public IP .-> DuckDNS
    NPM -. Request/Renew SSL .-> Internet
    
    %% DNS Resolution
    Router -. Assign DNS Server .-> PiHole
    PiHole -. Resolution .-> Internet
    
    %% CrowdSec Interaction
    NPM -. Send logs for analysis.-> CrowdSec
```

## 🔒 Security Posture

Security is a primary focus of this deployment:
1. **Zero Open Ports Internally:** Applications are isolated within Docker networks.
2. **Reverse Proxy:** Nginx Proxy Manager handles all external traffic and enforces HTTPS (Let's Encrypt SSL).
3. **Web Application Firewall (WAF):** CrowdSec is deployed on the host to analyze proxy logs in real-time, detecting and dropping malicious IPs (port scanning, brute-force) at the kernel level (`iptables`).

## 🚀 Deployed Services

### Core
* **Nginx Proxy Manager:** Traffic routing and SSL termination.
* **CrowdSec:** IPS/WAF protecting exposed endpoints.

### Applications
* **Immich:** High-performance, self-hosted photo and video backup (iCloud alternative) utilizing local machine learning for facial recognition.

## 🛠️ Deployment

Each service is defined in its own `docker-compose.yml`. 
To deploy a service:
1. Copy `.env.example` to `.env` and populate secrets.
2. Run `docker compose up -d`.
