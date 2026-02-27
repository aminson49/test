# GCP PSC Flow With AKS Consumer

```mermaid
flowchart LR
    aks[AKS cluster\n(Azure VNet consumer)] --> nsg[Azure NSG / firewall]
    nsg --> psc_ep[GCP PSC Endpoint\n(in consumer VPC)]
    psc_ep --> sa[PSC Service Attachment\n(provider)]
    sa --> ilb[GCP Internal Load Balancer]
    ilb --> ig[Instance Group / backends]
    ig --> nic1[Provider VM NIC 1\n(non-routable)]
    nic1 --> nic0[Provider VM NIC 0\n(routable)]
    nic0 --> vpc_fw[Provider VPC firewall\n(network tags)]
    vpc_fw --> onprem[On-prem network]
```
