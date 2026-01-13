# Homelab Infrastructure

## Overview

This documentation presents the architecture of a personal homelab with containerized services, redundant network infrastructure, and gaming services.

## 1. Global Network Architecture

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e1f5ff','primaryTextColor':'#000','primaryBorderColor':'#0288d1','lineColor':'#555','secondaryColor':'#ffe1f5','tertiaryColor':'#90EE90','noteTextColor':'#000','noteBkgColor':'#fff3cd'}}}%%
graph TB
    WAN([WAN])
    ISP[ISP Router]
    Router[Main Router<br/>Managed Switch]
    Server1[Main Server<br/>x86-64]
    Server2[ARM Server<br/>SBC Aarch64]
    Orbi[WiFi Router<br/>Relay Mode]
    Satellite[WiFi Satellite]
    Devices[Wired Devices]
    WiFi[WiFi Devices]
    
    WAN --> ISP
    ISP --> Router
    Router --> Server1
    Router --> Server2
    Router --> Orbi
    Router --> Devices
    Orbi --> Satellite
    Satellite -.WiFi.-> WiFi
    
    style Server1 fill:#e1f5ff,stroke:#0288d1,stroke-width:2px,color:#000
    style Server2 fill:#ffe1f5,stroke:#c2185b,stroke-width:2px,color:#000
    style WAN fill:#90EE90,stroke:#388e3c,stroke-width:2px,color:#000
    style ISP fill:#f5f5f5,stroke:#666,color:#000
    style Router fill:#f5f5f5,stroke:#666,color:#000
    style Orbi fill:#f5f5f5,stroke:#666,color:#000
    style Satellite fill:#f5f5f5,stroke:#666,color:#000
    style Devices fill:#f5f5f5,stroke:#666,color:#000
    style WiFi fill:#f5f5f5,stroke:#666,color:#000
```

## 2. Services per Machine

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e1f5ff','primaryTextColor':'#000','primaryBorderColor':'#0288d1','lineColor':'#555','secondaryColor':'#ffe1f5','tertiaryColor':'#fff4e1','noteTextColor':'#000'}}}%%
graph 
    subgraph Server1["🖥️ Main Server (x86-64)"]
        direction LR
        D1[<a href="https://gitlab.isc.org/isc-projects/kea">DHCP</a> - Backup]
        D2[<a href="https://github.com/AdguardTeam/AdGuardHome">DNS</a> - Backup]
        D3[<a href="https://github.com/caddyserver/caddy">Caddy Reverse Proxy</a>]
        D8[<a href="https://github.com/wg-easy/wg-easy">WireGuard VPN</a>]
        D9[<a href="https://gitlab.com/samba-team/samba">SMB Server</a><br/>+ Time Machine]
        D10[<a href="https://github.com/louislam/uptime-kuma">Monitoring</a>]

        subgraph MC_Stack["🎮 Gaming Stack"]
            D4[<a href="https://github.com/papermc/velocity">Velocity Proxy</a>]
            D5[<a href="https://github.com/Vianpyro/OxideVault">Discord Bot</a>]
            D6[<a href="https://github.com/itzg/docker-mc-backup">Backup Service</a>]
            D7[<a href="https://github.com/itzg/docker-mc-backup">Restore Service</a>]

            subgraph Services2["🌳 Minecraft Servers"]
                M1[<a href="https://github.com/itzg/docker-minecraft-server">Minecraft Test Server</a>]
                M2[<a href="https://github.com/itzg/docker-minecraft-server">Minecraft Main Server</a>]
            end
        end

        subgraph Services1["🌐 Exposed Web Services"]
            W1[<a href="https://github.com/Vianpyro/Penny-Game">Penny Game</a> - Web App]
            W2[<a href="https://github.com/granny/Pl3xMap">Pl3xmap</a> - Minecraft Map]
            W3[File Server<br/>with authentication]
        end
        
        subgraph Future["📅 Future Services"]
            F1[SSO Authentication]
            F2[Personal Cloud]
        end
    end
    
    subgraph Server2["🔶 ARM Server (SBC)"]
        direction TB
        O1[<a href="https://gitlab.isc.org/isc-projects/kea">DHCP</a> - Master]
        O2[<a href="https://github.com/AdguardTeam/AdGuardHome">DNS</a> - Master]
    end
    
    D3 --> W1
    D3 --> W2
    D3 --> W3

    D4 --> M1
    D4 --> M2
    
    M2 --> D6
    D7 --> M2

    style Server1 fill:#e1f5ff,stroke:#0288d1,stroke-width:2px,color:#000
    style Server2 fill:#ffe1f5,stroke:#c2185b,stroke-width:2px,color:#000
    style Services1 fill:#fff4e1,stroke:#f57c00,stroke-width:2px,color:#000
    style MC_Stack fill:#fff4e1,stroke:#f57c00,stroke-width:2px,color:#000
    style Future fill:#f0f0f0,stroke:#666,stroke-width:2px,stroke-dasharray: 5 5,color:#000
```

## 3. Data Flow and Communications

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e1f5ff','primaryTextColor':'#000','primaryBorderColor':'#0288d1','lineColor':'#555','secondaryColor':'#ffe1f5','tertiaryColor':'#f5e1ff','noteTextColor':'#000'}}}%%
graph LR
    subgraph External["🌍 External"]
        WAN([WAN])
        Remote[Remote<br/>Users]
    end
    
    subgraph Network["🏠 Local Network"]
        Clients[Network Clients]
        
        subgraph DHCP_HA["⚡ DHCP High Availability"]
            direction LR
            DHCP_M[DHCP Master<br/>ARM Server]
            DHCP_B[DHCP Backup<br/>x86 Server]
            DHCP_M <-.sync.-> DHCP_B
        end
        
        subgraph DNS_HA["🔍 DNS High Availability"]
            direction LR
            DNS_M[DNS Master<br/>ARM Server]
            DNS_B[DNS Backup<br/>x86 Server]
            DNS_M -.- DNS_B
        end
        
        subgraph MC_Stack["🎮 Gaming Stack"]
            Proxy[Game Proxy]
            
            subgraph Services2["🌳 Minecraft Servers"]
                M1[Minecraft Test Server]
                M2[Minecraft Main Server<br/>+ Voice Chat]
            end

            Discord[Discord Bot]
            Backup[Backup Service]
            Restore[Restore Service]
            StorageSSD[(Fast Storage)]
            StorageHDD[(Long Term Storage)]
        end
        
        RevProxy[Reverse Proxy<br/>+ Certificate Authority]
        Files[File Server]
        App1[Web Application]
        App2[Map Visualization]
        
        VPN[VPN]
    end
    
    Remote --> VPN
    VPN -.-> Clients
    
    Files -.auth.-> StorageHDD
    
    Clients --> DHCP_HA
    Clients --> DNS_HA
    
    WAN --> RevProxy
    WAN --> Proxy
    
    RevProxy --> Files
    RevProxy --> App1
    RevProxy --> App2
    
    Proxy --> M1
    Proxy --> M2
    M2 --> Discord
    M2 --> Backup
    Backup --> StorageSSD
    Restore --> StorageSSD
    Restore --> M2

    StorageSSD -.backup.-> StorageHDD

    style External fill:#90EE90,stroke:#388e3c,stroke-width:2px,color:#000
    style DHCP_HA fill:#ffe1e1,stroke:#d32f2f,stroke-width:2px,color:#000
    style DNS_HA fill:#e1f0ff,stroke:#1976d2,stroke-width:2px,color:#000
    style MC_Stack fill:#f5e1ff,stroke:#7b1fa2,stroke-width:2px,color:#000
    style Network fill:#f9f9f9,stroke:#666,stroke-width:2px,color:#000
```

## 4. Service Details

### Network Infrastructure

| Service | Machine | Role | Type |
|---------|---------|------|------|
| **DHCP** | ARM Server (Master) | IP Assignment - Primary | Container |
| **DHCP** | x86 Server (Backup) | IP Assignment - Secondary | Container |
| **DNS** | ARM Server (Master) | Resolution + Filtering - Primary | Container |
| **DNS** | x86 Server (Backup) | Resolution + Filtering - Secondary | Container |

### Web Services and Proxy

| Service | Machine | Description | Access |
|---------|---------|-------------|--------|
| **Reverse Proxy** | x86 Server | HTTPS Proxy | Local + External |
| **Penny Game** | x86 Server | Custom web application | Via Proxy |
| **Pl3xmap** | x86 Server | Interactive game map | Via Proxy |
| **File Server** | x86 Server | Accessible files | Via Proxy (auth) |

### Gaming and Communication

| Service | Machine | Description |
|---------|---------|-------------|
| **Game Server 1** | x86 Server | Tests Minecraft server |
| **Game Server 2** | x86 Server | Main Minecraft server |
| **Voice Chat** | x86 Server | Integrated voice plugin |
| **Game Proxy** | x86 Server | Connection proxy |
| **Discord Bot** | x86 Server | Discord integration |
| **Backup Service** | x86 Server | Automatic backups |
| **Restore Service** | x86 Server | Fast restoration |

### Security and Access

| Service | Machine | Description |
|---------|---------|-------------|
| **VPN** | x86 Server | Secure remote access |

### Monitoring and Observability

| Service | Machine | Description |
|---------|---------|-------------|
| **Monitoring** | x86 Server | Service availability monitoring |

### Storage and Sharing

| Service | Machine | Description |
|---------|---------|-------------|
| **Network Share** | x86 Server | Cross-platform file sharing |
| **Apple Backup Support** | x86 Server | Time Machine compatible |

### Planned Future Services

- **SSO Authentication**: Centralized identity management
- **Personal Cloud**: Synchronization and collaboration

## 5. Storage Architecture

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e1f5ff','primaryTextColor':'#000','primaryBorderColor':'#0288d1','lineColor':'#555','secondaryColor':'#90EE90','tertiaryColor':'#FFD700','noteTextColor':'#000'}}}%%
graph TB
    subgraph Storage["💾 Storage Architecture"]
        Fast[Fast Storage<br/>System + Apps]
        Mass[Mass Storage<br/>Data]
        
        subgraph Fast_Data["Fast Storage"]
            OS[OS + Containers]
            Runtime[Applications]
        end
        
        subgraph Mass_Data["Mass Storage"]
            Backups[Backups]
            Shares[Network Shares]
            Archives[Archives]
        end
        
        Fast --> Fast_Data
        Mass --> Mass_Data
    end
    
    style Storage fill:#e1f5ff,stroke:#0288d1,stroke-width:2px,color:#000
    style Fast fill:#90EE90,stroke:#388e3c,stroke-width:2px,color:#000
    style Mass fill:#FFD700,stroke:#f57f17,stroke-width:2px,color:#000
    style Fast_Data fill:#c8e6c9,stroke:#388e3c,color:#000
    style Mass_Data fill:#fff9c4,stroke:#f57f17,color:#000
```

## 6. High Availability

### DHCP High Availability
- **Mode**: Hot-Standby Failover
- **Master**: ARM Server
- **Backup**: x86 Server
- **Synchronization**: Automatic

### DNS High Availability
- **Mode**: Active redundancy
- **Master**: ARM Server
- **Backup**: x86 Server
- Clients can query both servers

## 7. Technology Stack

### Containerization
- **Orchestration**: Docker Compose
- **All services**: Deployed as containers
- **Management**: Via docker-compose.yml files

### Servers
- **Main Server (x86-64)**: Hosts majority of services
- **ARM Server (SBC)**: Critical network services (DHCP/DNS Master)

### Network
- **Topology**: Flat network (no VLANs)
- **WiFi**: Dual-band mesh in relay mode
- **Remote Access**: Secure VPN

## 8. Security

### External Access
- **VPN**: Secure access to all services (direct access to local network)
- **Reverse Proxy**: Controlled exposure of specific services to the Internet
- **Authentication**: Required for sensitive resources

### Best Practices
- High availability for critical services (DHCP/DNS)
- Automatic backups
- Fast restoration service
- System/data storage separation
- Secure remote access via VPN

## 9. Homelab Philosophy

### Principles
- **Full containerization**: All services in Docker
- **High availability**: Redundant network infrastructure
- **Role separation**: ARM for network, x86 for services
- **Automation**: Automated backups and restoration

### Goals
- Learn DevOps and self-hosting technologies
- Personal services independent from cloud providers
- Gaming and collaboration with friends
- Secure remote access

## 10. Future Evolution

### Short Term
- ✅ Operational HA infrastructure
- ✅ Deployed gaming services
- ✅ Automated backups
- ✅ Availability monitoring

### Medium Term
- 🔄 **SSO**: Centralized authentication
- 🔄 **Personal Cloud**: Multi-device synchronization

### Long Term
- Expand monitoring and observability
- Storage expansion
- New services as needed

## Conclusion

This homelab demonstrates a modern approach to self-hosting with:
- Robust and redundant network infrastructure
- Containerized services for easy management
- Intelligent workload separation
- Scalability for future services

The complete stack runs on **2 physical servers** managing **13+ Docker containers** with high availability for critical services.
