# GCP PSC Flow With AKS Consumer

```mermaid
flowchart TB
    a1["PSC consumer"] --> a2["Local VPC firewall"]
    a2 --> a3["PSC provider NIC 1 - non-routable"]
    a3 --> a4["NIC 0 - routable"]
    a4 --> a5["Host VPC firewall - network tags"]
    a5 --> a6["On-prem"]

    b1["PSC consumer"] --> b2["PSC Endpoint"]
    b2 --> b3["PSC Service Attachment"]
    b3 --> b4["ILB"]
    b4 --> b5["Instance Group"]
    b5 --> b6["Provider VM NIC 1"]
    b6 --> b7["NIC 0 - routable"]
    b7 --> b8["Host VPC firewall - network tags"]
    b8 --> b9["On-prem"]
```

## Explanation (GCP terms for AKS communication)

### Flow A - Direct provider NIC path (top)
- The AKS workload acts as the PSC consumer and reaches the provider VPC over private connectivity.
- Traffic passes the local VPC firewall and lands on the provider VM's non-routable NIC (NIC 1).
- The VM forwards traffic to the routable NIC (NIC 0), then through host VPC firewall rules (network tags) to on-prem.

### Flow B - PSC endpoint to ILB path (bottom)
- The AKS workload connects to a PSC endpoint in the consumer VPC.
- The PSC endpoint maps to a PSC service attachment in the provider VPC.
- The service attachment fronts a GCP Internal HTTP(S) Load Balancer (ILB).
- Host-based routing happens at the ILB using a URL map (Host header/SNI rules) to choose the backend service.
- The selected backend (instance group / NEG) forwards via provider VM NIC 1 → NIC 0 (routable) → VPC firewall → on-prem.

### Host-based routing summary
- PSC only provides private reachability; it does not do host routing.
- The Internal HTTP(S) Load Balancer performs host-based routing via URL map rules.
- AKS just sends HTTPS with the correct Host header; the ILB picks the backend.
