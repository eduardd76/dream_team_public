# Glossary

Acronyms + concepts used throughout the platform.

## Network protocols

| Term | Meaning |
|---|---|
| **EVPN** | Ethernet VPN. BGP-based control plane for Layer 2 over Layer 3. |
| **VXLAN** | Virtual Extensible LAN. The data-plane encapsulation used with EVPN. |
| **BGP-EVPN** | A BGP address family (L2VPN EVPN) carrying EVPN routes. |
| **VTEP** | VXLAN Tunnel Endpoint. The device that encapsulates / decapsulates VXLAN. Identified by its loopback IP. |
| **EVI** | EVPN Instance. One logical service (~one VNI's control state). |
| **VNI** | VXLAN Network Identifier. The 24-bit segment ID in a VXLAN header. |
| **RT** | Route Target. BGP extended community that controls EVPN route import/export. `65000:10010` style. |
| **RD** | Route Distinguisher. Makes EVPN routes unique across VRFs. |
| **NVE** | Network Virtualization Edge. NX-OS-speak for the VTEP interface. |
| **OSPF** | Open Shortest Path First. The underlay IGP we use in lab setups. |
| **BGP** | Border Gateway Protocol. Used for underlay routing in fabric designs + EVPN control plane. |
| **PCV** | Post-change Verifier (this platform's component). |
| **CRO** | Change Risk Officer (this platform's component). |
| **BRF** | Blast Radius Forecaster (this platform's component). |

## Platform-internal terms

| Term | Meaning |
|---|---|
| **ACP** | Approval Control Protocol. IBM's REST protocol used between Layer 1 (UI) and Layer 2 (Team Leader). |
| **A2A** | Agent-to-Agent. Google's protocol used between Layer 2 and Layer 3, implemented over Redis inboxes. |
| **MCP** | Model Context Protocol. Anthropic's protocol used between Layer 3 (specialists) and Layer 4 (MCP server). |
| **KG** | Knowledge Graph. Neo4j graph holding the live topology + state. |
| **DT** | Digital Twin. The validation surface for proposed changes (Layer 1 = graph; Layer 2 = containerlab; Layer 3 = persistent shadow). |
| **ValidateNode** | The 7-pass validation pipeline component in `agents/shared/validate_node.py`. |
| **Pass N** | One of the 7 ValidateNode stages (Confidence, Fault Correctness, Graph Sim, Containerlab, CRO, BRF, PCV). |
| **fault_type** | Catalogued root-cause class (`ospf_neighbor_loss_dead_timer`, `evpn_peer_admin_shut`, etc.). |
| **tier** | Layer of the fault (`underlay`, `overlay`, `transport`, `platform`, `security`, `service`). |
| **pattern_id** | Recovery-pattern identifier (`evpn_peer_admin_shut`, `interface_recover_no_shut`, etc.). Used by Pass 2. |
| **affected_component** | Subject of the change (a device, interface, peer IP). Cited in the approval card. |
| **skip_reason** | Named reason a Pass didn't run to a passed/failed verdict (e.g., `containerlab_not_installed`). |

## Vendor terms

| Term | Meaning |
|---|---|
| **NX-OS** | Cisco's Nexus operating system. Powers the NX9K data-center switches. |
| **EOS** | Arista's network OS. Powers vEOS / cEOS-lab. |
| **SR Linux** | Nokia's data-center NOS. Runs as `ghcr.io/nokia/srlinux` container. |
| **IOS-XE** | Cisco's enterprise routing/switching OS. Powers CSR1000v in our lab. |
| **vEOS** | Arista virtual EOS (qcow2 image for EVE-NG). |
| **cEOS** | Arista containerized EOS (docker image for containerlab). |
| **vrnetlab** | Open-source project that packages qcow2 vendor images into docker containers. |
| **containerlab** | Open-source container-orchestration tool for network topology testbeds. |

## Test + ops terms

| Term | Meaning |
|---|---|
| **SSM** | AWS Systems Manager. We use Run Command for code deploys to EC2 (SSH keys rotated). |
| **EVE-NG** | Emulated Virtual Environment NG. Hypervisor-style lab platform running on `eveng-prod` EC2. |
| **dry-run** | Pipeline runs end-to-end but `change-executor` never applies the fix to a real device. The platform's pilot default. |
| **golden parity** | Test gate that compares an adapter's output to the previous-vendor-baseline output. Used during vendor-abstraction migrations. |
| **MCP_USE_ADAPTER** | Env flag toggling between legacy parsers and new vendor adapters. Default `false` (legacy). |

## Workshop terms

| Term | Meaning |
|---|---|
| **AutoCon5** | The Cisco DevNet automation conference where this platform was built for. 2026-06-09. |
| **AI_Dream_Team** | The AWS EC2 instance hosting the app stack. `<INSTANCE_ID>`. |
| **the box** | Shorthand for `AI_Dream_Team`. |
