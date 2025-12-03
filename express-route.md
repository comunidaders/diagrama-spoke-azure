# Azure Multi-Hub Spoke Connectivity

## Cenário

Uma VNet **Spoke** precisa se conectar a **dois Hubs** em subscrições diferentes, cada um com seu próprio gateway (VPN ou ExpressRoute), e rotear o tráfego para o Hub correto conforme o destino.

### ⚠️ Limitação do VNet Peering Nativo

A opção **"Use Remote Gateways"** só pode ser habilitada em **um único peering por VNet**. Isso significa que uma Spoke não consegue usar nativamente os gateways de dois Hubs simultaneamente via peering tradicional.

---

## Opção 1: Azure Virtual WAN

### Visão Geral

O **Azure Virtual WAN** é uma solução gerenciada pela Microsoft que fornece conectividade transitiva global entre hubs, spokes, VPNs e ExpressRoute de forma nativa. Todos os hubs são conectados em full mesh automaticamente via backbone da Microsoft.

### Diagrama de Arquitetura

```mermaid
flowchart TB
    subgraph OnPrem["🏢 On-Premises"]
        DC1["Datacenter 1<br/>10.10.0.0/16"]
        DC2["Datacenter 2<br/>10.20.0.0/16"]
    end

    subgraph Azure["☁️ Azure Cloud"]
        subgraph VWAN["Azure Virtual WAN"]
            subgraph HubA["Virtual Hub A - East US"]
                direction TB
                AddrA["Address Space: 10.100.0.0/23"]
                VPNGW1["VPN Gateway"]
                ERGW1["ExpressRoute Gateway"]
                FW1["Azure Firewall<br/>(opcional)"]
            end

            subgraph HubB["Virtual Hub B - West US"]
                direction TB
                AddrB["Address Space: 10.200.0.0/23"]
                VPNGW2["VPN Gateway"]
                ERGW2["ExpressRoute Gateway"]
                FW2["Azure Firewall<br/>(opcional)"]
            end
        end

        subgraph SubA["Subscription A"]
            SpokeA1["Spoke VNet 1<br/>10.1.0.0/16"]
            SpokeA2["Spoke VNet 2<br/>10.2.0.0/16"]
        end

        subgraph SubB["Subscription B"]
            SpokeB1["Spoke VNet 3<br/>10.3.0.0/16"]
            SpokeB2["Spoke VNet 4<br/>10.4.0.0/16"]
        end
    end

    DC1 -->|"VPN Site-to-Site"| VPNGW1
    DC1 -->|"ExpressRoute"| ERGW1
    DC2 -->|"VPN Site-to-Site"| VPNGW2
    DC2 -->|"ExpressRoute"| ERGW2

    HubA -->|"VNet Connection"| SpokeA1
    HubA -->|"VNet Connection"| SpokeA2
    HubB -->|"VNet Connection"| SpokeB1
    HubB -->|"VNet Connection"| SpokeB2

    HubA <-->|"Global Transit<br/>Microsoft Backbone"| HubB

    style VWAN fill:#0078D4,color:#fff
    style HubA fill:#50E6FF,color:#000
    style HubB fill:#50E6FF,color:#000
    style OnPrem fill:#FFB900,color:#000
```

### Fluxo de Tráfego

```mermaid
sequenceDiagram
    participant Spoke as Spoke VNet
    participant HubA as Virtual Hub A
    participant HubB as Virtual Hub B
    participant DC1 as Datacenter 1
    participant DC2 as Datacenter 2

    Note over Spoke,DC2: Tráfego para DC1 (10.10.0.0/16)
    Spoke->>HubA: Pacote destino 10.10.x.x
    HubA->>DC1: Via VPN/ExpressRoute

    Note over Spoke,DC2: Tráfego para DC2 (10.20.0.0/16)
    Spoke->>HubA: Pacote destino 10.20.x.x
    HubA->>HubB: Inter-hub transit (automático)
    HubB->>DC2: Via VPN/ExpressRoute
```

### Componentes Necessários

| Componente | Descrição | SKU Recomendado |
|------------|-----------|-----------------|
| Virtual WAN | Recurso pai que agrupa os hubs | **Standard** |
| Virtual Hub | Hub gerenciado por região | /23 address space mínimo |
| VPN Gateway | Gateway S2S no hub | Scale Units conforme throughput |
| ExpressRoute Gateway | Gateway ER no hub | Scale Units conforme throughput |
| Azure Firewall | Inspeção de tráfego (opcional) | Premium |

### Configuração no Portal Azure

#### Passo 1: Criar Virtual WAN
1. Portal Azure → **Create a resource** → **Virtual WAN**
2. Configurar:
   - Nome: `vwan-global`
   - Tipo: **Standard** (necessário para multi-hub)
   - Resource Group e Região

#### Passo 2: Criar Virtual Hubs
1. Dentro do Virtual WAN → **Hubs** → **+ New Hub**
2. Para cada Hub configurar:
   - Região (ex: East US, West US)
   - Address space: `/23` (mínimo recomendado)
   - Virtual hub capacity: conforme necessidade

#### Passo 3: Adicionar Gateways aos Hubs
1. Dentro de cada Hub:
   - **VPN (Site to site)** → Criar gateway
   - **ExpressRoute** → Criar gateway
2. Configurar scale units conforme throughput necessário

#### Passo 4: Conectar VNets aos Hubs
1. Virtual WAN → **Virtual network connections** → **+ Add connection**
2. Configurar:
   - Connection name
   - Hubs: selecionar o hub apropriado
   - Subscription: pode ser diferente
   - Virtual network: selecionar a Spoke
   - Propagate to none: No
   - Associate Route Table: Default

#### Passo 5: Configurar Sites VPN / ExpressRoute
1. **VPN Sites** → adicionar sites on-premises
2. **ExpressRoute circuits** → conectar circuitos existentes
3. O roteamento entre hubs é **automático**

### Vantagens

| ✅ Vantagem | Descrição |
|-------------|-----------|
| Conectividade transitiva nativa | Todos os hubs e spokes se comunicam automaticamente |
| Roteamento automático | Não precisa de UDRs manuais |
| Full mesh entre hubs | Via backbone Microsoft |
| Escalabilidade gerenciada | Microsoft gerencia a infraestrutura |
| Integração Azure Firewall | Routing Intent para inspeção centralizada |
| Cross-subscription nativo | Conecta VNets de qualquer subscription |

### Desvantagens

| ❌ Desvantagem | Descrição |
|----------------|-----------|
| Custo elevado | Mais caro que hub-spoke tradicional |
| Menos flexibilidade | Customizações avançadas limitadas |
| Migração complexa | Se já existir infraestrutura tradicional |
| Vendor lock-in | Solução exclusiva Azure |

---

## Opção 2: NVA com Azure Route Server

### Visão Geral

Usar **Network Virtual Appliances (NVAs)** com **Azure Route Server** permite injetar rotas dinamicamente via BGP, eliminando a necessidade de UDRs manuais. O Route Server aprende rotas dos NVAs e as propaga automaticamente para as VMs.

### Diagrama de Arquitetura

```mermaid
flowchart TB
    subgraph OnPrem["🏢 On-Premises"]
        DC1["Datacenter 1<br/>10.10.0.0/16"]
        DC2["Datacenter 2<br/>10.20.0.0/16"]
    end

    subgraph Azure["☁️ Azure Cloud"]
        subgraph SubHub1["Subscription Hub A"]
            subgraph HubA["Hub VNet A - 10.100.0.0/16"]
                VPNGW1["VPN Gateway<br/>ASN 65010"]
                ARS1["Azure Route Server<br/>ASN 65515"]
                NVA1["NVA / Firewall<br/>IP: 10.100.1.4<br/>ASN 65001"]
            end
        end

        subgraph SubHub2["Subscription Hub B"]
            subgraph HubB["Hub VNet B - 10.200.0.0/16"]
                VPNGW2["VPN Gateway<br/>ASN 65020"]
                ARS2["Azure Route Server<br/>ASN 65515"]
                NVA2["NVA / Firewall<br/>IP: 10.200.1.4<br/>ASN 65002"]
            end
        end

        subgraph SubSpoke["Subscription Spoke"]
            subgraph Spoke["Spoke VNet - 10.50.0.0/16"]
                ARS3["Azure Route Server<br/>ASN 65515"]
                VM["Workloads"]
            end
        end
    end

    DC1 <-->|"VPN / ExpressRoute"| VPNGW1
    DC2 <-->|"VPN / ExpressRoute"| VPNGW2

    VPNGW1 <-->|"iBGP"| ARS1
    ARS1 <-->|"eBGP"| NVA1
    VPNGW2 <-->|"iBGP"| ARS2
    ARS2 <-->|"eBGP"| NVA2

    HubA <-->|"VNet Peering"| Spoke
    HubB <-->|"VNet Peering"| Spoke

    ARS3 <-->|"eBGP Peering"| NVA1
    ARS3 <-->|"eBGP Peering"| NVA2

    VM -.->|"Tráfego 10.10.0.0/16"| NVA1
    VM -.->|"Tráfego 10.20.0.0/16"| NVA2

    style HubA fill:#0078D4,color:#fff
    style HubB fill:#0078D4,color:#fff
    style Spoke fill:#50E6FF,color:#000
    style OnPrem fill:#FFB900,color:#000
    style ARS1 fill:#00BCF2,color:#000
    style ARS2 fill:#00BCF2,color:#000
    style ARS3 fill:#00BCF2,color:#000
    style NVA1 fill:#FF6B6B,color:#fff
    style NVA2 fill:#FF6B6B,color:#fff
```

### Fluxo de Propagação de Rotas BGP

```mermaid
flowchart LR
    subgraph Step1["1️⃣ On-Premises anuncia rotas"]
        OnPrem["On-Prem<br/>10.10.0.0/16"]
    end

    subgraph Step2["2️⃣ Gateway aprende via BGP"]
        GW["VPN/ER Gateway"]
    end

    subgraph Step3["3️⃣ Route Server distribui"]
        RS["Route Server<br/>Hub"]
    end

    subgraph Step4["4️⃣ NVA aprende e re-anuncia"]
        NVA["NVA/Firewall"]
    end

    subgraph Step5["5️⃣ Route Server Spoke aprende"]
        RS2["Route Server<br/>Spoke"]
    end

    subgraph Step6["6️⃣ VMs recebem rotas"]
        VMs["Workloads"]
    end

    OnPrem -->|"eBGP"| GW
    GW -->|"iBGP"| RS
    RS -->|"eBGP"| NVA
    NVA -->|"eBGP<br/>via peering"| RS2
    RS2 -->|"Injeta nas<br/>effective routes"| VMs
```

### Componentes Necessários

| Componente | Localização | Função | Subnet Necessária |
|------------|-------------|--------|-------------------|
| Azure Route Server | Hub A | Troca rotas BGP com NVA e Gateway | RouteServerSubnet (/24) |
| Azure Route Server | Hub B | Troca rotas BGP com NVA e Gateway | RouteServerSubnet (/24) |
| Azure Route Server | Spoke | Recebe rotas dos NVAs | RouteServerSubnet (/24) |
| NVA/Firewall | Hub A | Next-hop para DC1, faz peering BGP | Subnet dedicada |
| NVA/Firewall | Hub B | Next-hop para DC2, faz peering BGP | Subnet dedicada |
| VPN/ER Gateway | Hub A | Conectividade on-premises | GatewaySubnet (/27) |
| VPN/ER Gateway | Hub B | Conectividade on-premises | GatewaySubnet (/27) |

### Configuração no Portal Azure

#### Passo 1: Preparar Subnets

Em cada VNet (Hub A, Hub B, Spoke), criar:

| Subnet | CIDR | Observação |
|--------|------|------------|
| RouteServerSubnet | /24 ou /25 | Nome exato obrigatório |
| GatewaySubnet | /27 | Apenas nos Hubs |
| NVA-Subnet | /28 ou maior | Apenas nos Hubs |

#### Passo 2: Criar Route Servers

1. Portal Azure → **Create a resource** → **Route Server**
2. Para cada Route Server (Hub A, Hub B, Spoke):
   - Selecionar a VNet correspondente
   - Selecionar a RouteServerSubnet
   - Criar IP público (Standard SKU)
3. Após criação, habilitar **Branch-to-branch**: Yes

#### Passo 3: Configurar VNet Peering

**⚠️ IMPORTANTE: NÃO habilitar "Use Remote Gateways"**

| Configuração | Spoke → Hub | Hub → Spoke |
|--------------|-------------|-------------|
| Allow virtual network access | ✅ Enabled | ✅ Enabled |
| Allow forwarded traffic | ✅ Enabled | ✅ Enabled |
| Allow gateway transit | ❌ Disabled | ❌ Disabled |
| Use remote gateways | ❌ Disabled | N/A |

Repetir para ambos os Hubs (A e B).

#### Passo 4: Configurar BGP Peering nos Route Servers

**No Route Server do Hub A:**
1. Route Server → **Peers** → **+ Add**
2. Adicionar peering com o NVA local:
   - Peer name: `nva-hub-a`
   - Peer ASN: `65001` (ASN do NVA)
   - Peer IP: `10.100.1.4` (IP do NVA)

**No Route Server do Hub B:**
- Mesmo processo com NVA do Hub B (ASN 65002)

**No Route Server da Spoke:**
1. Adicionar peering com NVA do Hub A:
   - Peer ASN: `65001`
   - Peer IP: `10.100.1.4`
2. Adicionar peering com NVA do Hub B:
   - Peer ASN: `65002`
   - Peer IP: `10.200.1.4`

#### Passo 5: Configurar NVAs

Os NVAs precisam ser configurados para:

1. **Estabelecer BGP** com os Route Servers:
   - Peer com Route Server local (2 IPs - HA)
   - Peer com Route Server da Spoke (2 IPs - HA)

2. **Anunciar rotas** apropriadas:
   - NVA Hub A anuncia: `10.10.0.0/16` (DC1)
   - NVA Hub B anuncia: `10.20.0.0/16` (DC2)

3. **Configurar AS-Path** para controle de preferência

#### Passo 6: Verificar Rotas Aprendidas

Na Spoke, verificar effective routes das VMs:
1. VM → **Networking** → **Effective routes**
2. Deve mostrar:

| Prefixo | Next Hop Type | Next Hop | Source |
|---------|---------------|----------|--------|
| 10.10.0.0/16 | Virtual Appliance | 10.100.1.4 | Route Server |
| 10.20.0.0/16 | Virtual Appliance | 10.200.1.4 | Route Server |
| 10.100.0.0/16 | VNet peering | - | System |
| 10.200.0.0/16 | VNet peering | - | System |

### Requisitos do NVA para BGP

| Requisito | Descrição |
|-----------|-----------|
| Suporte BGP | Deve suportar eBGP (maioria dos firewalls enterprise) |
| Multi-hop BGP | Route Server usa eBGP multi-hop |
| ASN único | Cada NVA precisa de ASN diferente do 65515 (reservado) |
| IP Forwarding | Habilitar na NIC do NVA |
| AS Override | Necessário em cenários multi-região |

### NVAs Compatíveis

- Azure Firewall (via Route Server integration)
- Palo Alto Networks
- Fortinet FortiGate
- Cisco CSR/FTD
- Check Point
- Barracuda
- Qualquer appliance com suporte BGP

### Vantagens

| ✅ Vantagem | Descrição |
|-------------|-----------|
| Roteamento dinâmico | BGP elimina UDRs manuais |
| Flexibilidade | Usa qualquer NVA do mercado |
| Failover automático | BGP detecta falhas e reconverge |
| Menor custo | Mais barato que Virtual WAN |
| Controle granular | Políticas BGP customizáveis |
| Infraestrutura existente | Aproveita firewalls já licenciados |

### Desvantagens

| ❌ Desvantagem | Descrição |
|----------------|-----------|
| Complexidade | Requer conhecimento de BGP |
| Gestão de NVAs | Updates, HA, monitoramento |
| Route Server custo | ~$365/mês por instância |
| Troubleshooting | Mais difícil que Virtual WAN |

---

## Comparativo Final

```mermaid
quadrantChart
    title Comparativo: Virtual WAN vs NVA + Route Server
    x-axis Baixa Complexidade --> Alta Complexidade
    y-axis Baixo Custo --> Alto Custo
    quadrant-1 Não recomendado
    quadrant-2 Enterprise gerenciado
    quadrant-3 Custo-benefício
    quadrant-4 Flexibilidade máxima
    "Virtual WAN": [0.25, 0.80]
    "NVA + Route Server": [0.70, 0.45]
    "Hub-Spoke Tradicional": [0.40, 0.30]
    "Peering Simples": [0.15, 0.15]
```

### Tabela Comparativa

| Critério | Virtual WAN | NVA + Route Server |
|----------|-------------|-------------------|
| **Complexidade de Setup** | 🟢 Baixa | 🟡 Média-Alta |
| **Complexidade Operacional** | 🟢 Baixa | 🟡 Média |
| **Custo Mensal** | 🔴 Alto | 🟡 Médio |
| **Flexibilidade** | 🟡 Média | 🟢 Alta |
| **Roteamento** | Automático | BGP dinâmico |
| **Failover** | Nativo | Via BGP |
| **Escalabilidade** | Gerenciada MS | Manual |
| **Multi-região** | 🟢 Nativo | 🟡 Requer config |
| **Inspeção de tráfego** | Azure Firewall | Qualquer NVA |
| **Migração de existente** | 🟡 Complexa | 🟢 Incremental |
| **Suporte Microsoft** | 🟢 Completo | 🟡 Parcial (NVA é vendor) |

---

## Recomendação por Cenário

### 🏢 Use **Virtual WAN** quando:

- ✅ Projeto **greenfield** (começando do zero)
- ✅ Precisa de **simplicidade operacional**
- ✅ Tem **budget** disponível
- ✅ Quer **suporte unificado** Microsoft
- ✅ Múltiplas regiões com **requisito de full mesh**
- ✅ Equipe com **pouca experiência** em roteamento avançado

### 🔧 Use **NVA + Route Server** quando:

- ✅ Já possui **NVAs/Firewalls licenciados**
- ✅ Precisa de **controle granular** sobre roteamento
- ✅ Quer **otimizar custos**
- ✅ Tem **expertise em BGP** na equipe
- ✅ Cenário **brownfield** (infraestrutura existente)
- ✅ Requisitos específicos de **compliance** com vendor específico

---

## Estimativa de Custos (Referência USD)

### Virtual WAN

| Recurso | Custo/hora | Custo/mês (estimado) |
|---------|------------|---------------------|
| Virtual WAN Standard | $0.05/hub | ~$36/hub |
| VPN Gateway (1 SU) | $0.361 | ~$263 |
| ExpressRoute GW (1 SU) | $0.42 | ~$306 |
| Hub Data Processing | $0.02/GB | Variável |
| **Total mínimo (2 hubs)** | - | **~$1.200+** |

### NVA + Route Server

| Recurso | Custo/hora | Custo/mês (estimado) |
|---------|------------|---------------------|
| Route Server | $0.50 | ~$365 |
| VPN Gateway VpnGw1 | $0.19 | ~$138 |
| NVA (varia por vendor) | - | $200-$1000+ |
| **Total mínimo (3 RS + 2 GW)** | - | **~$800+** |

*Valores aproximados, consulte a calculadora Azure para valores atuais.*

---

## Referências

- [Azure Virtual WAN Overview](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about)
- [Hub-Spoke with Virtual WAN Architecture](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke-virtual-wan-architecture)
- [Azure Route Server Overview](https://learn.microsoft.com/en-us/azure/route-server/overview)
- [Route Injection in Spokes](https://learn.microsoft.com/en-us/azure/route-server/route-injection-in-spokes)
- [Multi-Hub Spoke with Azure Firewall](https://learn.microsoft.com/en-us/azure/firewall/firewall-multi-hub-spoke)
- [VPN Gateway Transit Configuration](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-peering-gateway-transit)
- [Multi-region Networking with Route Server](https://learn.microsoft.com/en-us/azure/route-server/multiregion)
- [Virtual WAN Routing Policies](https://learn.microsoft.com/en-us/azure/virtual-wan/how-to-routing-policies)
