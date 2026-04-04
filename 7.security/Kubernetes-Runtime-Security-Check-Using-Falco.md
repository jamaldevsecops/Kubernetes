🛡️ Kubernetes Runtime Security Runbook using Falco

⸻

🎯 Coverage Scope (Very Important)

Falco is a runtime security tool for Kubernetes and Linux.

👉 Unlike kube-bench / kubescape (config) and kube-hunter (probing), Falco:

✅ What Falco detects

🧠 Runtime Threats
	•	Shell spawned inside containers
	•	Privilege escalation attempts
	•	Sensitive file access (e.g., /etc/shadow)
	•	Suspicious process execution

🔐 Kubernetes-Aware Events
	•	Exec into pods
	•	Creation of privileged containers
	•	Changes to ConfigMaps / Secrets
	•	Unexpected API actions (via audit logs)

🐧 Host-Level Signals
	•	System calls (syscalls) via kernel
	•	Process tree anomalies
	•	Network activity (basic visibility)

⸻

❗ What Falco does NOT cover
	•	❌ CIS compliance (use kube-bench)
	•	❌ Static misconfiguration (use kubescape)
	•	❌ External exposure scanning (use kube-hunter)

👉 Falco = Real-time detection & alerting (runtime IDS)

⸻

🧰 Prerequisites
	•	Kubernetes cluster (kubeadm / on-prem)
	•	Helm 3
	•	Kernel support for eBPF or driver (preferred: eBPF)
	•	Internet access OR mirrored images (for restricted env)

⸻

📥 Installation (Recommended: Helm)

Add Helm repo

helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update


⸻

Install Falco (eBPF preferred)

helm install falco falcosecurity/falco \
  -n falco --create-namespace \
  --set driver.kind=ebpf


⸻

Verify

kubectl get pods -n falco

Expected:
	•	falco DaemonSet pod on each node

⸻

📦 Offline / Restricted Environment Notes
	•	Pre-pull images and push to private registry
	•	Set image.repository to your registry
	•	If eBPF not allowed, fallback to kernel module driver

⸻

🚀 Basic Usage

View live logs

kubectl logs -n falco -l app.kubernetes.io/name=falco -f


⸻

Example alert

Warning Shell spawned in container


⸻

📊 Understanding Output

Alert Structure
	•	Rule: What triggered
	•	Priority: Severity
	•	Output: Human-readable message
	•	Fields: container, pod, user, command

⸻

Priority Levels

Level	Meaning
Debug	Noise / debug
Info	Informational
Notice	Suspicious
Warning	Risky
Error	High risk
Critical	Immediate action


⸻

🔍 Common Alerts → Remediation

1) Shell spawned in container

Risk
	•	Possible compromise or manual debugging in prod

Fix
	•	Restrict kubectl exec access
	•	Enforce RBAC
	•	Audit access patterns

⸻

2) Write below /etc

Risk
	•	Container modifying system files

Fix
	•	Use read-only root filesystem

securityContext:
  readOnlyRootFilesystem: true


⸻

3) Privileged container started

Fix

securityContext:
  privileged: false


⸻

4) Sensitive file read (/etc/shadow)

Risk
	•	Credential access attempt

Fix
	•	Restrict container capabilities
	•	Use minimal base images

⸻

5) Unexpected outbound connection

Risk
	•	Possible data exfiltration

Fix
	•	Apply NetworkPolicies
	•	Monitor egress traffic

⸻

⚙️ Custom Rules (Very Important)

Create custom rule file

cat >/etc/falco/rules.d/custom_rules.yaml <<'EOF'
- rule: Detect Curl in Container
  desc: Detect curl usage
  condition: container and proc.name = curl
  output: "Curl executed in container (user=%user.name command=%proc.cmdline)"
  priority: WARNING
EOF


⸻

Apply via Helm values

customRules:
  custom_rules.yaml: |
    - rule: Detect Curl in Container
      desc: Detect curl usage
      condition: container and proc.name = curl
      output: "Curl executed in container"
      priority: WARNING


⸻

🔔 Alert Forwarding (Enterprise)

Enable file output

falco:
  json_output: true
  json_include_output_property: true


⸻

Integrate with SIEM / Webhook
	•	Send logs to:
	•	ELK / OpenSearch
	•	Splunk
	•	Teams / Slack webhook

⸻

🤖 Automation Script (Quick Check)

cat >/usr/local/bin/check-falco.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

kubectl get pods -n falco
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=20
EOF

chmod +x /usr/local/bin/check-falco.sh


⸻

🔁 Jenkins Integration Example

stage('Falco Health Check') {
  steps {
    sh '''
      kubectl get pods -n falco
      kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=50 > falco.log || true
    '''
    archiveArtifacts artifacts: 'falco.log', fingerprint: true
  }
}


⸻

⚠️ Operational Considerations
	•	Tune rules to reduce noise
	•	Validate alerts before enforcement
	•	Monitor performance impact
	•	Keep Falco rules updated

⸻

🔐 Enterprise Best Practices
	•	Run Falco on all nodes (DaemonSet)
	•	Integrate with SIEM
	•	Use custom rules per application
	•	Combine with:
	•	kube-bench
	•	kube-hunter
	•	kubescape
	•	Trivy

⸻

📎 Verification Checklist

✔ Falco pods running on all nodes
✔ Alerts visible
✔ Custom rules applied
✔ Logs exported
✔ Noise tuned

⸻

🎯 Golden Commands

Check pods

kubectl get pods -n falco

View alerts

kubectl logs -n falco -l app.kubernetes.io/name=falco -f


⸻

🚀 Outcome
	•	Real-time threat detection
	•	Runtime visibility
	•	Incident response readiness

⸻

📌 End of Runbook