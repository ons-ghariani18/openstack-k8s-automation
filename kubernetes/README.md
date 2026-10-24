# Kubernetes manifests
- `monitoring/custom-alerts.yaml` – Prometheus alert rules (HighPodCPU, PodCrashLooping, NodeNotReady, HighMemoryUsage) as a ConfigMap.
- `failure-scenarios/crash-test-pod.yaml` – pod that crashes on purpose (CrashLoopBackOff / auto-restart demo).
- `failure-scenarios/cpu-stress-pod.yaml` – pod that burns ~1 CPU core (HighPodCPU alert + Grafana demo).
