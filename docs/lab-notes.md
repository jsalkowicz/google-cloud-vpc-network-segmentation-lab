# Detailed Lab Notes — Working with Multiple VPC Networks

## Task 1 — Create Custom VPC Networks and Firewall Rules

### Task 1A — `managementnet`

**Action:** Created a custom-mode VPC through the Google Cloud Console.

**Configuration**
- Network: `managementnet`
- Subnet: `managementsubnet-us`
- Region: `us-west1`
- CIDR: `10.130.0.0/20`

**Observation:** `managementnet` appeared in Custom mode while the preconfigured `default` and `mynetwork` networks used Auto mode.

**What I learned:** Custom-mode VPCs let me explicitly control where subnets are created and what IP ranges they use.

**Security relevance:** Deliberate subnet placement and addressing support intentional segmentation and network design.

### Task 1B — `privatenet`

**Action:** Created a custom-mode VPC and two regional subnets with `gcloud`.

```bash
gcloud compute networks create privatenet --subnet-mode=custom

gcloud compute networks subnets create privatesubnet-us \
  --network=privatenet \
  --region=us-west1 \
  --range=172.16.0.0/24

gcloud compute networks subnets create privatesubnet-notus \
  --network=privatenet \
  --region=asia-east1 \
  --range=172.20.0.0/20
```

**Observation:** The custom VPC contained only the subnets explicitly created.

**What I learned:** VPCs are global resources while subnets are regional.

**Security relevance:** Custom network design provides tighter control over segmentation and IP allocation.

### Task 1C/1D — Firewall Rules

Created matching ingress firewall rules for `managementnet` and `privatenet` allowing ICMP, SSH, and RDP as required by the lab.

**Observation:** The rules used `0.0.0.0/0`.

**What I learned:** Firewall behavior is defined by direction, source, target, protocol, and port.

**Security relevance:** Broad administrative exposure is appropriate only for this controlled training exercise. Production SSH/RDP exposure would normally be more restrictive.

### Task 1E — Verification

Used `gcloud` to verify matching custom VPC firewall policies.

**What I learned:** CLI-based verification makes it easier to review multiple configurations consistently.

---

## Task 2 — Compute Engine VM Deployment

### Task 2A — `managementnet-us-vm`

Created through the Console.

- Zone: `us-west1-c`
- Machine type: `e2-medium`
- Network: `managementnet`
- Subnet: `managementsubnet-us`
- Internal IP: `10.130.0.2`
- Interface: `nic0`

**Observation:** The IP was assigned from the subnet's `10.130.0.0/20` range.

**What I learned:** A VM's network interface determines its VPC membership, subnet, and internal addressing.

### Task 2B — `privatenet-us-vm`

Created through the CLI:

```bash
gcloud compute instances create privatenet-us-vm \
  --zone=us-west1-c \
  --machine-type=e2-medium \
  --subnet=privatesubnet-us
```

- Internal IP: `172.16.0.2`
- Status: Running

**What I learned:** Compute Engine deployment can be performed directly and repeatably from `gcloud`.

---

## Task 3 — Connectivity Testing

### External IP Testing

External ICMP connectivity succeeded to all tested instances when the applicable firewall rules allowed it.

**Security relevance:** Public reachability must be evaluated separately from VPC segmentation.

### Internal Same-VPC Testing

`mynet-us-vm` successfully reached `mynet-notus-vm` using an internal IP even though the instances were in different regions.

**What I learned:** Google Cloud VPC networks are global.

### Internal Cross-VPC Testing

From `mynet-us-vm`:
- `managementnet-us-vm` — 100% packet loss
- `privatenet-us-vm` — 100% packet loss

**Key takeaway:** VPC membership, rather than same-zone placement, determined default private connectivity in the lab.

**Security relevance:** Separate VPCs provided network segmentation by default.

---

## Task 4 — Multi-NIC VM

Created `vm-appliance` with:

| Interface | Network | Subnet | Internal IP |
|---|---|---|---|
| `nic0` | `privatenet` | `privatesubnet-us` | `172.16.0.3` |
| `nic1` | `managementnet` | `managementsubnet-us` | `10.130.0.3` |
| `nic2` | `mynetwork` | `mynetwork` | `10.138.0.3` |

Used:

```bash
sudo ifconfig
```

to inspect Linux interfaces and:

```bash
ip route
```

to examine host routing.

### Connectivity Results

- `privatenet-us-vm` — Successful
- `managementnet-us-vm` — Successful
- `mynet-us-vm` — Successful
- `mynet-notus-vm` — Failed

**Observation:** Directly connected subnets had usable routes. Traffic without a matching specific route followed the primary interface's default route.

**What I learned:** Being attached to a VPC is not the same as having a usable route to every subnet in that VPC.

**Security relevance:** Multi-NIC security appliances require deliberate routing, firewall, interface, and DNS design.

### Evidence Note

The timed Google Cloud Skills Boost environment expired after the work was completed and before terminal screenshots of `sudo ifconfig`, Task 4 connectivity testing, and `ip route` were preserved. The multi-NIC deployment screenshot was preserved.

---

## Overall Takeaways

- VPCs are global; subnets are regional.
- Custom mode provides deliberate network and CIDR design.
- Separate VPCs provide default isolation.
- Firewall rules control allowed network traffic.
- External reachability is distinct from private segmentation.
- VM interfaces define network membership and internal addresses.
- Multi-NIC systems still depend on host routing for actual traffic paths.
