# 🖥️ Automated Infrastructure & Monitoring System

Personal Home Lab Project | RHEL 9 | 2025–2026

A production-grade home lab environment demonstrating real-world skills in infrastructure design, automation, high availability, and full-stack observability — built entirely from scratch on virtual machines.

🌐 [**Project Website**](https://automatedinfra.lovable.app/)  |  📓 [**Interactive Notebook (NotebookLM)**](https://notebooklm.google.com/notebook/06558014-6927-4665-8372-b8a4e9686a0b)  |  📄 [**Full Report (PDF)**](https://github.com/Ruslan-JS/Automated-Infrastructure-Monitoring-System/blob/9f05f7c8fa2fe31f1c78cf75d0802a735035fb34/Project%20Report.pdf)

---

## 📐 Architecture Overview

graph TD

    Internet\["🌐 Internet / NAT\\n192.168.11.x"\]

    Internet \--\> EDGE

    EDGE\["🔷 EDGE VM\\n192.168.11.100 — Public NAT\\n192.168.50.1 — Private Backbone\\nNginx Reverse Proxy · Zabbix Agent · Graylog Relay"\]

    EDGE \--\> APP

    EDGE \--\> DATA

    EDGE \--\> MON

    APP\["📦 APP VM\\n192.168.50.3:8081\\nWeb Application · Zabbix Agent"\]

    DATA\["🗄️ DATA VM — Primary DB\\n192.168.50.2:8080\\nPostgreSQL Primary · Zabbix Agent"\]

    DATA \--\> REPLICA

    REPLICA\["🗄️ Replica VM\\n192.168.50.6\\nPostgreSQL Replica · Zabbix Agent"\]

    DATA \-. "VIP 192.168.50.100\\nKeepalived Failover" .-\> REPLICA

    MON\["📊 Monitoring VM\\n192.168.11.146\\nZabbix Server · Grafana · Graylog · DNS Master"\]

    MON \--\> DNS2\["🔁 DNS Backup VM\\n192.168.11.150\\nBIND9 Slave"\]

    CONTROL\["⚙️ Control VM\\nAnsible Controller\\nMini Cloud — Local Repo"\]

    CONTROL \-- "SSH \+ Playbooks" \--\> EDGE

    CONTROL \-- "SSH \+ Playbooks" \--\> APP

    CONTROL \-- "SSH \+ Playbooks" \--\> DATA

    CONTROL \-- "SSH \+ Playbooks" \--\> REPLICA

    CONTROL \-- "SSH \+ Playbooks" \--\> MON

---

## ✅ Completed Milestones

| Phase | Component | Status |
| :---- | :---- | :---- |
| Week 1 | Internal VLAN \+ Reverse Proxy \+ VIP | ✅ |
| Week 1 | Live System Health Dashboard | ✅ |
| Week 1 | Nginx Access & Error Logging | ✅ |
| Week 2–3 | Zabbix Monitoring \+ Agents on all VMs | ✅ |
| Week 2–3 | PostgreSQL Primary/Replica Replication | ✅ |
| Week 2–3 | Virtual IP — Keepalived HA Failover | ✅ |
| Week 2–3 | Grafana Dashboard (CPU, Memory, Network) | ✅ |
| Week 2–3 | Graylog Centralised Log Aggregation | ✅ |
| Week 3–4 | BIND9 DNS Master/Slave (Forward \+ Reverse Zones) | ✅ |
| Week 3–4 | Ansible Full Automation (5 Playbooks) | ✅ |

---

## 🧱 Stack

| Category | Technology |
| :---- | :---- |
| OS | RHEL 9 |
| Containers | Podman \+ systemd services |
| Web / Proxy | Nginx (containerized) |
| Database | PostgreSQL (Primary \+ Replica) |
| HA / Failover | Keepalived (Virtual IP) |
| Monitoring | Zabbix Server \+ Agents, Grafana |
| Logging | Graylog \+ OpenSearch \+ rsyslog |
| DNS | BIND9 (Master/Slave, Forward/Reverse zones) |
| Automation | Ansible (Playbooks \+ Jinja2 Templates) |
| Security | SELinux, firewalld, TLS-PSK (Zabbix) |

---

## 🔧 Key Components

### 1\. Nginx Reverse Proxy (Containerized)

- Routes `/app` → `192.168.50.3:8081` and `/data` → `192.168.50.2:8080`  
- Custom `default.conf` with structured log format for monitoring  
- Built into a custom image with embedded health-check scripts  
- Managed as systemd services for reboot persistence

### 2\. Live System Health Dashboard

- Bash script polls all backend services every 5 seconds via `curl`  
- Auto-generates and updates `status.html` with service name, IP:port, HTTP status, and response time

### 3\. PostgreSQL Primary/Replica Replication

- Primary has full read/write; Replica is read-only streaming replica  
- Verified by inserting test data on Primary and reading instantly from Replica

### 4\. Virtual IP — Keepalived HA

- VIP `192.168.50.100` floats between Primary and Replica DB VMs  
- Health-check script uses `podman exec ... pg_isready` to test DB liveness  
- On Primary failure, VIP migrates to Replica automatically — no manual steps

### 5\. Zabbix \+ Grafana Monitoring

- Agents on all VMs report to central Zabbix server  
- Custom item: `systemd.unit.get["container-edge.service"]` \+ JSONPath `$.ActiveState.text` → returns `1` or `0`  
- Single Grafana dashboard shows disk, CPU, memory, and network for all VMs  
- Podman socket spoofed as Docker socket so Zabbix agent2's built-in Docker plugin works with Podman

### 6\. Graylog Centralised Logging

- Internal VMs forward logs → Edge VM (dual-homed) → Graylog container  
- Ports: `1514/udp`, `1514/tcp`, `1515/tcp` to prevent protocol lock

### 7\. BIND9 DNS Master/Slave

- Forward zone `monitoring.local` resolves all hosts on both `.11` and `.50` subnets  
- Reverse PTR zones for both subnets  
- Slave DNS auto-pulls all zone files from Master  
- Security: ACL, firewalld, SELinux `named_zone_t`, Ansible-managed patching

### 8\. Ansible Automation

ansible/

├── playbooks/

│   ├── deploy\_containers.yml

│   └── patch.yml

├── roles/

│   ├── dns-server/        ← BIND9 Master/Slave auto-config

│   ├── graylog\_forward/   ← rsyslog forwarding config

│   ├── web/               ← Web template deployment

│   ├── firewall/

│   └── common/

└── site.yml

A single `ansible-playbook site.yml` run configures Edge VM as NAT gateway, pushes DNS to all VMs, sets subnet-specific gateways, builds the Mini Cloud local package mirror, and deploys Graylog forwarding and web templates across the fleet.

---

## 🐛 Notable Challenges Solved

| Problem | Root Cause | Solution |
| :---- | :---- | :---- |
| Containers dying on reboot | Podman is daemonless | `podman generate systemd --new` → systemctl service |
| `podman stop` conflicting with systemd | Two controllers owning same container | Only use `systemctl` — never raw `podman stop/start` |
| Log files created as symlinks | Wrong `COPY` path in Dockerfile | Fixed destination path in Dockerfile |
| Keepalived exit code 127 | `podman` called without absolute path | Use `/usr/bin/podman --url unix:///run/podman/podman.sock` |
| SELinux `exec_died` on VIP script | Missing SELinux context on script | `restorecon -v /usr/local/bin/check_pg.sh` \+ socket perms |
| VIP not migrating on failover | Missing `mcast_src_ip` in keepalived config | Added `mcast_src_ip` for correct VRRP multicast source |
| Graylog connection refused | Container bound UDP-only, input on TCP | Redeploy with both `/tcp` and `/udp` on both ports |
| Zabbix showing inverted 0/1 | Raw `ActiveState` is text, not numeric | JSONPath `$.ActiveState.text` \+ regex `active` → `1` |

---

## 🔗 Resources

| Resource | Link |
| :---- | :---- |
| 🌐 Project Website | [automatedinfra.lovable.app](https://automatedinfra.lovable.app/) |
| 📓 Interactive Notebook | [NotebookLM](https://notebooklm.google.com/notebook/06558014-6927-4665-8372-b8a4e9686a0b) |
| 📄 Full Weekly Report | [Project\_Report.pdf](http://./Project_Report.pdf) |

---

## 👤 Author

**Ruslan Rustamzada**  
Junior System Administrator | Infrastructure Automation  
📧 [ruslanrustemzade03@gmail.com](mailto:ruslanrustemzade03@gmail.com)  
📍 Baku, Azerbaijan | Open to Remote Roles  
