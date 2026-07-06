# Lab 2 — Virtual Networks & Secure Storage with Terraform

A single-apply Terraform project that builds a segmented virtual network and locks a storage account away from the public internet entirely, reachable only through a private endpoint inside that network.

**Stack:** Terraform (azurerm + random) · Azure CLI · four-file layout (`main.tf`, `variables.tf`, `outputs.tf`, `terraform.tfvars`)
**AZ-104 domains:** Implement & manage virtual networking · Implement & manage storage · Deploy with infrastructure as code

---

## The Business Problem

A storage account holding customer files, backups, or application data has a public endpoint by default — reachable from anywhere on the internet with the right key. The security team's nightmare is simple: a leaked key, and the data is one HTTPS request away from being pulled out by anyone, anywhere.

The professional fix isn't a stronger key policy — it's removing the public door entirely. Put the storage account behind a private endpoint inside your own virtual network, disable public access, and let only resources on that network reach it. A leaked key is then useless from the outside, because there's no network path in.

This lab builds exactly that pattern: a VNet split into purpose-built subnets, a network security group that allows only intended traffic, a storage account with public access disabled, and a private endpoint that becomes the only way in — all as Terraform, so the topology is peer-reviewable and redeployable rather than something clicked together once and forgotten.

## Architecture

```
Terraform (main.tf • azurerm)
        │ provisions all
        ▼
Virtual Network (10.0.0.0/16)
├── Subnet: workload (10.0.1.0/24)
│   └── Workload VM ← filtered by NSG (allow-listed rules)
└── Subnet: endpoints (10.0.2.0/24)
    └── Private Endpoint ──private link──► Storage Account (public access OFF)
                                                    ▲
                                            Public Internet ──blocked──┘
```

Terraform provisions the whole topology in one apply. The storage account's public access is disabled from creation; the private endpoint in its own dedicated subnet is the only path to it, and a private DNS zone makes the storage account's hostname resolve to that private IP from inside the VNet.

## What I Did

1. **Scoped the network up front.** Picked the address space before writing any code: VNet `10.0.0.0/16`, workload subnet `10.0.1.0/24`, private-endpoint subnet `10.0.2.0/24` — kept in a dedicated subnet rather than sharing with workloads, which is the recommended pattern.
2. **Built the four-file Terraform project**, same layout as Lab 1: resource group, VNet and two subnets, an NSG allowing SSH only from my own IP (via `my_ip_cidr` in `terraform.tfvars`), a storage account created with `public_network_access_enabled = false` from the start, and a private endpoint + private DNS zone wired to the storage account's blob subresource.
3. **Hit a parser error from a PDF-sourced line.** The `provider "azurerm"` block failed to parse — traced it to smart/curly quotes copied from the lab PDF instead of straight quotes. Retyping the line by hand fixed it. Filed this as a standing gotcha for any future lab content copied from a PDF.
4. **Deployed a test VM into the workload subnet**, deliberately with no public IP, so the only way to reach it was Serial Console — reinforcing that the private endpoint, not a public address, was the intended access path. Used Serial Console with boot diagnostics enabled and a temporary password (`az vm user update`) rather than Bastion, to avoid the extra subnet and cost.
5. **Tested connectivity from inside the VNet.** From the VM, resolved the storage account's blob hostname to a `10.0.2.x` private IP and reached it successfully via `nc`.
6. **Tested — and initially mis-tested — connectivity from outside.** My first pass used `Test-NetConnection` and `curl`/`Invoke-WebRequest` against the blob URL from my laptop and both "succeeded," which looked like a failure of the lockdown. It wasn't: Azure's shared frontend accepts the TCP connection on port 443 before account-level network rules are ever evaluated, so a raw connectivity test proves nothing about whether the storage account itself is reachable. The real test is `az storage container list --account-name <name> --auth-mode login`, which forces actual data-plane evaluation — from my laptop this returned an explicit network-rules-block error, which is the correct, provable negative result.
7. **Captured screenshots** across apply success, state list, the VNet/subnet layout, NSG rules, storage networking showing public access disabled, the private endpoint connection, the VM-side `nslookup`/`nc` success, the laptop-side blocked `az storage container list`, and `terraform destroy` success — using the `lab2-NN-description.png` convention so they sort in build order.
8. **Ran `terraform destroy`** promptly once screenshots were captured, since the private endpoint carries a small hourly charge on the free-tier subscription.

## What I'd Do Differently in Production

- **Restrict by resource, not just network.** Public access is disabled here, which is the right first move, but a production setup would layer storage-account-level firewall rules (or Private Link exclusively, with no fallback network rule) and use Azure Policy to *deny* any storage account created with public access enabled in the first place, rather than relying on each engineer remembering the setting.
- **Remote state, not local.** This lab's state lives on disk. In a team setting I'd back this with a remote backend (Azure Storage with state locking) so multiple engineers can't corrupt each other's plans, and so the state isn't sitting on one laptop.
- **NSG rules scoped further.** My rule allows SSH from one IP — fine for a solo lab. A production NSG would typically deny SSH entirely in favor of Bastion or a jump host, since even a single allow-listed IP is a standing inbound door that has to be managed as people's IPs change.
- **Multiple private endpoints, one per subresource, with approval workflows.** Real environments often connect several services (blob, file, queue, table) via separate private endpoints, and in a shared subscription the connection approval would go through a formal request/approve flow rather than `is_manual_connection = false`.
- **DNS zone shared across subscriptions.** A single organization typically centralizes private DNS zones in a hub subscription and links every spoke VNet to it, rather than creating a new zone per lab/project as this config does — avoids DNS fragmentation as the environment grows.
- **Don't rely on connectivity tests that stop at TCP.** The `Test-NetConnection` false-positive here is a good lesson: production runbooks and automated tests should validate the actual data plane (e.g., an authenticated data-plane call) rather than treating a successful TCP handshake as proof that an access restriction is working.
- **Tag and budget this like Lab 1's VM.** This lab didn't carry over the budget alert or required-tag policy from Lab 1 — in a real environment, every resource group would get those governance controls by default, not just the ones built in the "governance" lab.
