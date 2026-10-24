# OpenStack Private Cloud (Epoxy) + Automated Kubernetes Platform with AI-assisted Monitoring

> Multi-node OpenStack private cloud (manual, package-based install) with Kubernetes provisioned by **Heat + Ansible**, observability with **Prometheus/Grafana**, AI-based anomaly detection & pod resource recommendations, containerized Spring Boot/Angular/MySQL app and a **Jenkins** CI/CD pipeline.

![OpenStack](https://img.shields.io/badge/OpenStack-Epoxy-ED1944?logo=openstack&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28-326CE5?logo=kubernetes&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-EE0000?logo=ansible&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?logo=jenkins&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)

## 1. Overview
- **OpenStack Epoxy** private cloud: 1 controller + 6 compute nodes (Nova, Neutron/OVS, Cinder-LVM, Swift, Glance, Keystone, Placement, Heat, Octavia, Horizon).
- **Heat** provisions a two-tier network (frontend/backend), a router, a security group and 4 instances with floating IPs; **Ansible** then configures the Kubernetes cluster with a **single command**.
- **Observability**: Prometheus (Alertmanager, node-exporter, kube-state-metrics) + Grafana, custom alert rules, **AI anomaly detector**, **load predictor** with CPU/RAM recommendations, **HPA**.
- **Resilience tests**: pod crash, node failure, automatic rescheduling.
- **Hybrid deployment (OpenStack + Azure)**: the Angular frontend runs on the OpenStack cloud (port `30100` of the Kubernetes master), the Spring Boot + MySQL backend runs in Docker on Azure; an **ngrok** HTTPS tunnel publishes the frontend and connects the two clouds.
- **Containerization**: Docker images (Spring Boot backend, Angular frontend, MySQL).

## 2. Architecture
```mermaid
flowchart TB
    subgraph VM["VMware Workstation lab"]
      subgraph OS["OpenStack Epoxy"]
        C["Controller<br/>Keystone · Glance · Nova · Neutron · Cinder · Swift · Heat · Octavia · Horizon"]
        CN["6 Compute nodes"]
        ST[("Cinder (LVM) + Swift")]
        subgraph K["Kubernetes 1.28 (Calico, containerd)"]
            M["master (m1.tiny)"]
            W1["worker1 (m1.small)"]
            W2["worker2 (m1.small)"]
        end
        FE["Angular frontend"]
      end
    end
    subgraph AZ["Microsoft Azure"]
        BE["Spring Boot backend + MySQL (Docker)"]
    end
    subgraph O["Observability (in Kubernetes)"]
        P["Prometheus + Alertmanager"] --> G["Grafana"]
        AI["Anomaly detector · Load predictor · HPA"]
    end
    ANS["Ansible (one command from the controller)"] --> K
    HEAT["Heat stack: 2 networks, router, SG, 4 VMs"] --> OS
    C --> CN --- ST
    USR(("Users")) -- HTTPS --> NG["ngrok tunnel"]
    NG -- ":30100" --> FE
    FE -- "REST API" --> BE
    K --> P --> AI
```

## 3. Tech stack
| Domain | Technologies |
|---|---|
| Cloud | OpenStack Epoxy, Open vSwitch, Cinder (LVM), Swift, Octavia, Heat |
| Automation | Heat (HOT), Ansible roles, Helm |
| Kubernetes | kubeadm 1.28, containerd, Calico v3.26, metrics-server, HPA |
| App | Spring Boot, Angular, MySQL 8, Ollama |
| Monitoring | Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, Python anomaly detector |
| CI/CD & Security | Jenkins, Docker multi-stage, RBAC, Secrets, NetworkPolicies, non-root containers |

## 4. Infrastructure
| Item | Value |
|---|---|
| Hypervisor | VMware Workstation (nested virtualization lab) |
| OpenStack | Epoxy – manual install on Ubuntu (see [docs](docs/openstack-controller-install.md)) |
| Controller VM | 4 vCPU · 8 GB RAM · 50 GB disk · multiple VMnet adapters ([screenshot](docs/images/09-vmware-controller-resources.png)) |
| Nodes | 1 controller + 6 compute nodes |
| K8s VMs | `master` (m1.tiny), `worker1`, `worker2` (m1.small) – Ubuntu 22.04 |
| Networks | Self-service tenant network `172.16.1.0/24` + provider network (floating IPs `192.168.1.0/24`) |
| Storage | Cinder volumes (LVM backend), Swift object storage |
| Kubernetes | 1.28, Calico CNI, containerd |
| Backend hosting | Microsoft Azure VM (Docker: Spring Boot + MySQL 8) |

## 5. Implementation
1. **OpenStack** – manual deployment of every service (database → user → endpoints → config → db sync). Details and troubleshooting: [`docs/openstack-controller-install.md`](docs/openstack-controller-install.md).
2. **Infrastructure as Code** – [`heat/infrastructure-stack.yaml`](heat/infrastructure-stack.yaml) creates:
   - 2 networks (`10.10.0.0/24` frontend, `10.20.0.0/24` backend) and a router with an external gateway;
   - 1 security group (SSH/HTTP/HTTPS/ICMP, MySQL `3306` restricted to the frontend subnet);
   - 4 instances (2× Ubuntu 22.04, 2× CirrOS) with ports and floating IPs, key-based SSH only;
   - outputs with the floating IPs.
   A volume-enabled variant of the stack `[À COMPLÉTER]`.
3. **Automatic configuration** – once the Heat stack has created the VMs, a **single `ansible-playbook` command** run from the controller configures everything (no manual step on the nodes):
   - `ansible/site.yml` – modular **roles** (common, containerd, kubernetes, control_plane, calico, workers, cluster_verify, demo_apps, helm).
   - `ansible/k8s-ai-monitoring.yml` – complete playbook: cluster + Helm Prometheus/Grafana + AI anomaly detector + load predictor + HPA + HA tests + dashboard.
4. **Monitoring & alerting** – alert rules in [`kubernetes/monitoring/custom-alerts.yaml`](kubernetes/monitoring/custom-alerts.yaml): `HighPodCPU`, `PodCrashLooping`, `NodeNotReady`, `HighMemoryUsage`.
5. **AI-assisted operations** – anomaly detector (`ai-monitoring` namespace) and load predictor that suggests increasing/decreasing pod CPU/RAM.
6. **Application (hybrid)** – Spring Boot backend + MySQL run as Docker containers on Azure; the Angular frontend (FlexBidaya organizer dashboard) runs on OpenStack and is published through an **ngrok** tunnel (public HTTPS URL → `localhost:30100` on the master), which links the OpenStack-hosted frontend with the Azure-hosted backend.

## 6. Demonstrations
**VMs of the Kubernetes cluster running on OpenStack**
![Horizon instances](docs/images/01-horizon-instances.png)

**Object storage (Swift)**
![Swift](docs/images/02-swift-object-storage.png)

**Prometheus scraping the 3 Kubernetes nodes (3/3 up)**
![Targets](docs/images/03-prometheus-targets.png)

**Custom alert rules loaded in Prometheus**
![Alerts](docs/images/04-prometheus-alert-rules.png)

**Pod crash → Kubernetes keeps restarting it (CrashLoopBackOff, restart counter increasing)**
![Crash](docs/images/05-pod-crashloop-restart.png)

**Injected CPU anomaly (`cpu-stress`) visible in Grafana**
![Grafana](docs/images/06-grafana-cpu-stress.png)

**Angular frontend served from the OpenStack cloud**
![Frontend](docs/images/08-angular-frontend-openstack.png)

**ngrok tunnel publishing the frontend (hybrid OpenStack ↔ Azure)**
![ngrok](docs/images/10-ngrok-tunnel.png)

**Containerized application images (Spring Boot backend, Angular frontend, MySQL)**
![Docker](docs/images/07-docker-images.png)

| Scenario | Result |
|---|---|
| High CPU anomaly | `cpu-stress` pod pushes CPU to ~1 core, visible in Grafana, detected by the anomaly detector |
| Pod crash | Automatic restart by Kubernetes |
| Node failure | Automatic pod rescheduling `[À COMPLÉTER: add screenshot]` |
| Resource tuning | Recommendations to raise/lower pod CPU/RAM |

### Additional components (documentation in progress)
Octavia load balancer, custom firewall rules (security groups), Cinder backup, Ollama integration and Jenkins CI/CD were also part of the project; their configuration files will be added to this repository.

## 7. Repository structure
```
.
├── README.md
├── ansible/
│   ├── ansible.cfg · site.yml · k8s-ai-monitoring.yml
│   ├── inventory/hosts.ini · group_vars/all.yml
│   └── roles/ (common, containerd, kubernetes, control_plane, calico, workers, cluster_verify, demo_apps, helm)
├── heat/infrastructure-stack.yaml
├── kubernetes/           # alert rules, failure scenarios (+ app manifests)
├── docker/               # Dockerfiles (multi-stage)
├── jenkins/              # Jenkinsfile
└── docs/
    ├── openstack-controller-install.md
    └── images/
```

## 8. How to reproduce
```bash
# 1. OpenStack ready (see docs/)  2. Create the stack
openstack stack create -t heat/infrastructure-stack.yaml --parameter keypair_name=<your-keypair> --wait infra-stack
openstack stack output show infra-stack --all
# 3. Configure everything with Ansible, from the controller (password is never stored in the repo)
cd ansible
ansible-playbook k8s-ai-monitoring.yml -e vault_grafana_admin_password='<YOUR_PASSWORD>'
# 4. Failure scenarios
kubectl apply -f ../kubernetes/monitoring/custom-alerts.yaml
kubectl apply -f ../kubernetes/failure-scenarios/crash-test-pod.yaml
kubectl apply -f ../kubernetes/failure-scenarios/cpu-stress-pod.yaml
```

## 9. Challenges & solutions
| Challenge | Solution |
|---|---|
| Controller lost external connectivity after Octavia setup (default route injected on `o-bhm0`) | Removed the route and added a dhclient hook ignoring routers on that interface |
| Octavia could not use the amphora image | Image owner did not match the project ID in config; aligned both |
| Heat resource `Create Failed` (port not found, HTTP 400) when creating an instance | `[À COMPLÉTER: your fix]` |

## 10. Future improvements
HA controller, GitOps (ArgoCD), centralized logging (Loki), Terraform, TLS everywhere.

## 11. Authors
**[Your name]** – [LinkedIn](https://linkedin.com/in/...)  ·  Team members: `[À COMPLÉTER si projet d'équipe]`
