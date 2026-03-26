---
title: "Services"
weight: 57
description: >
  kube-vip services reconciliation.
---

kube-vip is able to reconcile Kubernetes `Service` objects to configure networking and load balancing for those resources. Here you can find some info on specific configurations regarding services usage.

## Enabling services processing

Service reconciliation can be enabled with the `svc_enable: true` flag or with the `--services` CLI flag.

## Leader election modes

For each kube-vip's mode (`ARP`, `BGP`, `Routing Table` and `WireGuard`) various configurations of leader election can be applied. Generally, services can be processed without leader election (processing will happen on all kube-vip instances), with global leader election (only one kube-vip instance will process all services) or with per-service leader election (each service will be reconciled on a leader node elected separately for each service). Leader election can be enabled using the `svc_election` env variable/`servicesElection` CLI flag and the `vip_leaderelection` env variable/`leaderElection` CLI flag (the latter only for Routing Table mode). Below you can find all valid configurations for each mode and their meaning.

### ARP mode

| svc_election | Description                                                              |
| ------------ | ------------------------------------------------------------------------ |
| true         | Per-service leader election (leader will be elected for each service)    |
| false        | Global leader election (all services reconciled by the same leader node) |


### BGP mode

| svc_election | Description                                                              |
| ------------ | ------------------------------------------------------------------------ |
| true         | Per-service leader election (leader will be elected for each service)    |
| false        | Leader election disabled (services reconciled by all kube-vip instances) |



### Routing Table mode

| svc_election | vip_leaderelection | Description                                                              |
| ------------ | ------------------ | ------------------------------------------------------------------------ |
| true         | not used           | Per-service leader election (leader will be elected for each service)    |
| false        | true               | Global leader election (all services reconciled by the same leader node) |
| false        | false              | Leader election disabled (services reconciled by all kube-vip instances) |

### WireGuard mode

| svc_election | Description                                                           |
| ------------ | --------------------------------------------------------------------- |
| true         | Per-service leader election (leader will be elected for each service) |
| false        | Services reconciliation disabled                                      |


## Loadbalancer IP

Loadbalancer IP for the service can be specified in 3 separate ways. The precedence of those methods is as follows:

1. Annotation `kube-vip.io/loadbalancerIPs` - IP address can be specified manually. IPv4 and IPv6 can be specified for dualstack services, separated with a comma (e.g. "10.0.0.1,fe80:0000::1").
1. 'spec.loadBalancer.IP` field - can be specified manually, but can only contain one IPv4 or IPv6 address.
1. `status.loadBlancer.ingress` field - can be used if set by e.g. cloud controller.

Additionally, hostname can be used to obtain IP address using DHCP service with the annotation `kube-vip.io/loadbalancerHostname`.

## Requesting IP with DHCP

IP address for service can be also requested from DHCP server with the following annotations:

- `kube-vip.io/hwaddr` - specifies hardware address (MAC) of the interface that the service should be bound to.
- `kube-vip.io/requestedIP` - the IP address that should be requested from the DHCP server.

## DDNS

Services reconciled with kube-vip can use DDNS to update IP address assigned to the provided hostname. This can be enabled with service annotation `kube-vip.io/ddns=true`.

## External cluster policy

kube-vip can reconcile services with both `externalClusterPolicy: Cluster` and `externalClusterPolicy: local`.

- `externalClusterPolicy: Cluster` - will reconcile service if any endpoint is available in the cluster.
- `externalClusterPolicy: Local` - will reconcile service only if an endpoint is available on the same node as the kube-vip pod is deployed on.

## Common lease feature (e.g. sharing VIP address among multiple services)

kube-vip's leader election leases can be shared to force kube-vip's components to be reconciled by selected node. When using global leader election the name of the lease can be specified with the `svc_leasename` env variable or `servicesLeaseName` CLI flag. If it will be configured to be the same as control-plane lease (set by the `vip_leasename` variable or `leaseName` flag), single leader election will be performed for both control-plane and services reconciler.

While using per-service leader election, by default kube-vip will create a separate lease for each service automatically. One can, however, override the lease name with service annotation:

```
kube-vip.io/leaseName: example-lease
```

If multiple services use the same lease, all of them will be reconciled by the same leader node. This makes it possible to share the same IP address (set with`kube-vip.io/loadbalancerIPs`) among multiple services, as the VIP address will be present only on the selected leader node.

## Egress

