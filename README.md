# 🖥️ Automated Infrastructure & Monitoring System

Personal Home Lab Project | RHEL 9 | 2025–2026

A production-grade home lab environment demonstrating real-world skills in infrastructure design, automation, high availability, and full-stack observability — built entirely from scratch on virtual machines.

🌐 [**Project Website**](https://automatedinfra.lovable.app/)  |  📓 [**Interactive Notebook (NotebookLM)**](https://notebooklm.google.com/notebook/06558014-6927-4665-8372-b8a4e9686a0b)  |  📄 [**Full Report (PDF)**](https://github.com/Ruslan-JS/Automated-Infrastructure-Monitoring-System/blob/9f05f7c8fa2fe31f1c78cf75d0802a735035fb34/Project%20Report.pdf)  
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
| Web / Proxy | Nginx (containerised) |
| Database | PostgreSQL (Primary \+ Replica) |
| HA / Failover | Keepalived (Virtual IP) |
| Monitoring | Zabbix Server \+ Agents, Grafana |
| Logging | Graylog \+ OpenSearch \+ rsyslog |
| DNS | BIND9 (Master/Slave, Forward/Reverse zones) |
| Automation | Ansible (Playbooks \+ Jinja2 Templates) |
| Security | SELinux, firewalld, TLS-PSK (Zabbix) |

---

## 🔧 Key Components

### 1\. Nginx Reverse Proxy (Containerised)

- Routes `/app` → `192.168.50.3:8081` and `/data` → `192.168.50.2:8080`  
- Custom `default.conf` with structured log format for monitoring  
- Built into a custom image with embedded health-check scripts  
- Managed as systemd services for reboot persistence

### 2\. Live System Health Dashboard

- Bash script polls all backend services every 5 seconds via `curl`  
- Auto-generates and updates `status.html` with service name, IP: port, HTTP status, and response time

### 3\. PostgreSQL Primary/Replica Replication

- Primary has full read/write; Replica is a read-only streaming replica  
- Verified by inserting test data on Primary and reading instantly from Replica

### 4\. Virtual IP — Keepalived HA

- VIP `192.168.50.100` floats between Primary and Replica DB VMs  
- Health-check script uses `podman exec ... pg_isready` to test DB liveness  
- On Primary failure, VIP migrates to the replica automatically — no manual steps

### 5\. Zabbix \+ Grafana Monitoring

- Agents on all VMs report to the central Zabbix server  
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

.  
├── ansible.cfg  
├── host\_vars  
│   └── web  
│       └── main.yml  
├── inventory  
│   ├── group\_vars  
│   └── production  
├── playbooks  
│   ├── deploy\_containers.yml  
│   └── patch.yml  
├── roles  
│   ├── common  
│   │   └── tasks  
│   │       └── main.yml  
│   ├── db  
│   ├── dns-server  
│   │   ├── handlers  
│   │   │   └── main.yml  
│   │   ├── tasks  
│   │   │   └── main.yml  
│   │   └── templates  
│   │       ├── monitoring.local.forward  
│   │       ├── monitoring.local.reverse.11  
│   │       ├── monitoring.local.reverse.50  
│   │       └── named.conf.j2  
│   ├── firewall  
│   │   └── tasks  
│   │       └── main.yml  
│   ├── graylog\_forward  
│   │   ├── handlers  
│   │   │   └── main.yml  
│   │   ├── tasks  
│   │   │   └── main.yml  
│   │   └── templates  
│   │       └── graylog.conf.j2  
│   ├── podman\_stack  
│   ├── web  
│   │   ├── files  
│   │   │   └── template.zip  
│   │   ├── handlers  
│   │   │   └── main.yml  
│   │   └── tasks  
│   │       └── main.yml  
│   └── zabbix\_agent  
└── site.yml

A single `ansible-playbook site.yml` run configures Edge VM as a NAT gateway, pushes DNS to all VMs, sets subnet-specific gateways, builds the Mini Cloud local package mirror, and deploys Graylog forwarding and web templates across the fleet.

---

## 🐛 Notable Challenges Solved

| Problem | Root Cause | Solution |
| :---- | :---- | :---- |
| Containers dying on reboot | Podman is daemonless | `podman generate systemd --new` → systemctl service |
| `Podman stop` is conflicting with systemd | Two controllers owning the same container | Only use `systemctl` — never raw `podman stop/start` |
| Log files created as symlinks | Wrong `COPY` path in Dockerfile | Fixed destination path in Dockerfile |
| Keepalived exit code 127 | `podman` called without an absolute path | Use `/usr/bin/podman --url unix:///run/podman/podman.sock` |
| SELinux `exec_died` on VIP script | Missing SELinux context on script | `restorecon -v /usr/local/bin/check_pg.sh` \+ socket perms |
| VIP is not migrating on failover | Missing `mcast_src_ip` in keepalived config | Added `mcast_src_ip` for the correct VRRP multicast source |
| Graylog connection refused | Container-bound UDP-only, input on TCP | Redeploy with both `/tcp` and `/udp` on both ports |
| Zabbix is showing inverted 0/1 | Raw `ActiveState` is text, not numeric | JSONPath `$.ActiveState.text` \+ regex `active` → `1` |

---

## 🔗 Resources

| Resource | Link |
| :---- | :---- |
| 🌐 Project Website | [automatedinfra. lovable.app](https://automatedinfra.lovable.app/) |
| 📓 Interactive Notebook | [NotebookLM](https://notebooklm.google.com/notebook/06558014-6927-4665-8372-b8a4e9686a0b) |
| 📄 Full Weekly Report | [Project\_Report.pdf](http://./Project_Report.pdf) |

---

## 👤 Author

**Ruslan Rustamzada**  
Junior System Administrator | Infrastructure Automation  
📧 [ruslanrustemzade03@gmail.com](mailto:ruslanrustemzade03@gmail.com)  
📍 Baku, Azerbaijan | Open to Remote Roles  
