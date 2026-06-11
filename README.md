# fleet-free5gc-upf

Standalone edge UPF Fleet bundle for free5GC split-deployment.

## Architecture

```
CORE CLUSTER                    EDGE CLUSTER
AMF, SMF, NRF, UDM...          UPF (this bundle) + gNB
       |                               |
       | N4 (PFCP/UDP 8805)            | N3 (GTP-U, local)
       | cross-cluster                 | N6 (internet egress)
       +------- NodePort ------------->|
```

## Interface layout

| Interface | Transport | Where |
|-----------|-----------|-------|
| N4 (PFCP) | Pod network → NodePort | Cross-cluster (core SMF → edge UPF) |
| N3 (GTP-U) | Multus ipvlan on enp1s0 | Local edge (gNB ↔ UPF) |
| N6 (DN) | Multus ipvlan on enp1s0 | Local edge (UPF → internet) |

## Platform contract

**Producer endpoint (N4/PFCP):**
- Service: `upf-n4`
- Namespace: `free5gc`
- Port: `8805/UDP`
- Type: `NodePort` (auto-assigned)
- Reachable at: `<edge-node-IP>:<nodePort>`

The remote SMF must be configured with this UPF's N4 at `<edge-node-IP>:<nodePort>`.

## Critical: address mismatch warning

PFCP and GTP-U carry IP addresses **inside their payloads** (PFCP Node IDs / F-TEIDs),
not just in packet headers. NodePort rewrites the header, not the payload. Therefore:

- The SMF's `userplaneInformation` must list this UPF's N4 at the **cross-cluster-reachable**
  address (edge node IP + nodePort), not the Multus 10.100.50.x address.
- The UPF's N3 address must be the **gNB-reachable** edge-local address (10.100.50.233).
- If PFCP association establishes but no user data flows, check for address mismatches here.

## Dependencies
- `gtp5g-installer` — UPF uses the in-kernel GTP-U datapath
- `multus-cni` — N3 and N6 interfaces

## Verify deployment
```bash
# NodePort service exists
kubectl get svc upf-n4 -n free5gc

# UPF pod is running
kubectl get pods -n free5gc -l nf=upf

# PFCP listening on 0.0.0.0
kubectl logs -n free5gc -l nf=upf | grep "pfcp\|0.0.0.0\|8805"

# UDP reachability from another pod
kubectl run -it --rm probe --image=busybox --restart=Never -- \
  nc -u -w1 <node-ip> <nodePort>
```
