# Cilium CNI ADD Command Flow

Source: `plugins/cilium-cni/cmd/cmd.go` — `func (cmd *Cmd) Add(args *skel.CmdArgs)`

---

## Overview

CNI ADD is invoked by kubelet when a pod is created. The cilium-cni binary is short-lived — it runs, configures networking, and exits. The long-running cilium-agent daemon does the heavy lifting (IPAM, BPF program loading, endpoint lifecycle).

```
kubelet
  └─ exec cilium-cni binary (ADD)
       ├─ connect to cilium-agent (Unix socket)
       ├─ allocate IP via agent
       ├─ create veth pair (host ↔ container)
       ├─ configure container netns (IP, routes, sysctl)
       ├─ POST /v1/endpoint → agent creates BPF programs
       └─ print CNI result JSON → stdout
```

---

## Step-by-Step Flow

### 1. Parse CNI Input

```go
n, err := types.LoadNetConf(args.StdinData)
cniArgs := &types.ArgsSpec{}
cniTypes.LoadArgs(args.Args, cniArgs)
```

- `args.StdinData` = CNI config JSON (NetConf) piped by kubelet
- `cniArgs` extracts `K8S_POD_NAME`, `K8S_POD_NAMESPACE` from the CNI args env
- File: `plugins/cilium-cni/types/types.go`

### 2. Connect to cilium-agent

```go
c, err := client.NewDefaultClientWithTimeout(defaults.ClientConnectTimeout)
conf, err := getConfigFromCiliumAgent(c)
```

- Connects via Unix socket at `/var/run/cilium/cilium.sock`
- Fetches `DaemonConfigurationStatus` — IPAM mode, MTU, datapath mode, addressing, etc.
- All subsequent decisions depend on this config object

### 3. Chaining Mode Check

```go
if len(n.NetConf.RawPrevResult) != 0 {
    chainAction.Add(ctx, c)  // delegate entirely
}
```

- If `PrevResult` is present, Cilium is running on top of another CNI (AWS CNI, Flannel, Azure CNI, generic-veth)
- In chaining mode, the upstream CNI already created the interface and allocated the IP
- Cilium only programs BPF policy — it does NOT create a veth or allocate IPs
- Chaining implementations: `plugins/cilium-cni/chaining/`

### 4. Open Pod Network Namespace

```go
ns, err := netns.OpenPinned(args.Netns)
```

- `args.Netns` = path like `/var/run/netns/<id>` — pinned by container runtime
- Namespace is held open for the duration of ADD; operations enter it via `ns.Do(func)`

### 5. Allocate IP Address

Two paths based on IPAM mode from daemon config:

**Path A — Cilium agent IPAM** (cluster-pool, kubernetes, ENI, Azure, etc.)
```go
ipam, releaseIPsFunc, err = allocateIPsWithCiliumAgent(c, cniArgs, epConf.IPAMPool())
// calls: c.IPAMAllocate("", "namespace/podname", poolName, true)
// agent REST: POST /v1/ipam
```

**Path B — Delegated plugin IPAM** (host-local, etc.)
```go
ipam, releaseIPsFunc, err = allocateIPsWithDelegatedPlugin(ctx, conf, n, args.StdinData)
// calls: cniInvoke.DelegateAdd(ctx, netConf.IPAM.Type, stdinData, nil)
```

On any later failure, `releaseIPsFunc` is deferred to return the IP to the pool.

### 6. Prepare Endpoint Model

```go
state, ep, err := epConf.PrepareEndpoint(ipam)
```

- Builds a `models.EndpointChangeRequest` struct with IP, container ID, labels, etc.
- `state` holds IP addresses and routes to configure on the interface
- File: `plugins/cilium-cni/cmd/endpoint.go`

### 7. Create Veth Pair

```go
linkConfig := connector.LinkConfig{
    EndpointID:    cniID,
    PeerIfName:    epConf.IfName(),   // container-side name (e.g. "eth0")
    PeerNamespace: ns,
    DeviceMTU:     int(conf.DeviceMTU),
    // GRO/GSO max sizes, headroom, tailroom...
}
linkPair, err := connector.NewLinkPair(logger, linkMode, linkConfig, sysctl)
```

- Creates a veth pair: one end in host netns, one end moved to container netns
- `linkMode` = `veth` (default) or `ipvlan` based on datapath mode
- Host-side name generated from container ID (truncated)
- File: `pkg/datapath/connector/`

### 8. Build CNI Result Interfaces

```go
res.Interfaces = append(res.Interfaces, &cniTypesV1.Interface{
    Name: hostLinkAttrs.Name,
    Mac:  hostLinkAttrs.HardwareAddr.String(),
})
```

- Records host-side interface in CNI result
- MAC addresses from link attributes

### 9. Prepare IP Config and Routes

```go
ipConfig, routes, err = prepareIP(ep.Addressing.IPv4, state, int(conf.RouteMTU))
ipv6Config, routes, err = prepareIP(ep.Addressing.IPv6, state, int(conf.RouteMTU))
```

- Parses IP from IPAM response into `netip.Addr`
- Calls `connector.IPv4Routes` / `connector.IPv6Routes` to compute routes via host gateway
- Routes stored in `state.IP4routes` / `state.IP6routes` for later netns application

### 10. Host-Side Routes (Cloud IPAM modes only)

```go
if needsEndpointRoutingOnHost(conf) {
    interfaceAdd(ipConfig, ipam.IPv4, conf)  // installs /32 host route
}
```

- Only for ENI, Azure, AlibabaCloud, or DelegatedPlugin with `InstallUplinkRoutesForDelegatedIPAM`
- Installs a `/32` (or `/128`) host route on the node pointing to pod IP

### 11. Configure Container Netns

```go
ns.Do(func() error {
    reserveLocalIPPorts(conf, sysctl)       // net.ipv4.ip_local_reserved_ports
    sysctl.Disable(["net","ipv6","conf","all","disable_ipv6"])  // if IPv6
    macAddrStr, err = configureIface(ipam, epConf.IfName(), state)
    return err
})
```

`configureIface` inside the netns:
1. `netlink.LinkSetUp` — brings container interface UP
2. `netlink.AddrAdd` — assigns IPv4 and/or IPv6 address
3. `netlink.RouteAdd` — installs routes (default route via host gateway, etc.)
4. `route.ReplaceRule` — installs policy routing rules (for multi-homing / source routing)

### 12. Get Netns Cookie

```go
ns.Do(func() error {
    cookie, err = netns.GetNetNSCookie()
})
ep.NetnsCookie = strconv.FormatUint(cookie, 10)
```

- Kernel netns cookie = unique ID stable across namespace reuse
- Passed to agent for BPF map keying

### 13. Create Endpoint in Agent (Synchronous)

```go
ep.SyncBuildEndpoint = true  // block until BPF programs loaded
newEp, err = c.EndpointCreate(ep)
// agent REST: POST /v1/endpoint
```

- Agent receives the endpoint model, assigns a numeric endpoint ID
- Allocates a security identity from labels
- Compiles and loads BPF programs for this endpoint (tc programs on the veth)
- Updates `lxcmap` BPF map with endpoint ID → IP/ifindex mapping
- Updates `policymap` with allowed identities
- `SyncBuildEndpoint = true` means ADD blocks until BPF is ready

### 14. Final Netns Configuration

```go
ns.Do(func() error {
    configurePacketizationLayerPMTUD(conf, sysctl)  // PLPMTUD
    return configureCongestionControl(conf, sysctl)  // cubic TCP CC
})
```

- Optionally sets TCP MTU probing (`net.ipv4.tcp_mtu_probing`)
- Optionally overrides TCP congestion control to `cubic` (host-ns-only BBR mode)

### 15. Return CNI Result

```go
res.Interfaces = append(res.Interfaces, &cniTypesV1.Interface{
    Name:    epConf.IfName(),  // "eth0"
    Mac:     macAddrStr,
    Sandbox: args.Netns,
})
return cniTypes.PrintResult(res, n.CNIVersion)
```

- Prints JSON result to stdout (CNI spec requirement)
- kubelet reads this to confirm pod network is ready

---

## Data Flow Diagram

```
kubelet
  │
  │ exec + stdin: CNI JSON config
  ▼
cilium-cni binary (cmd.Add)
  │
  ├─[1] Parse NetConf + CNI args
  │
  ├─[2] GET /v1/config  ──────────────────► cilium-agent
  │      DaemonConfigurationStatus ◄────────
  │
  ├─[3] Chaining? ─── yes ──► chainAction.Add() → done
  │         │
  │        no
  │
  ├─[4] Open pod netns (pinned fd)
  │
  ├─[5] POST /v1/ipam  ───────────────────► cilium-agent
  │      IPAMResponse (IP, gateway) ◄───────
  │
  ├─[6] PrepareEndpoint (build EP model)
  │
  ├─[7] connector.NewLinkPair
  │      └─ create veth: lxc<id> (host) ↔ eth0 (container netns)
  │
  ├─[8-9] Build CNI result + compute routes
  │
  ├─[10] Install host routes (ENI/Azure only)
  │
  ├─[11] Enter netns: AddrAdd + RouteAdd + ReplaceRule + sysctl
  │
  ├─[12] Get netns cookie
  │
  ├─[13] POST /v1/endpoint ──────────────► cilium-agent
  │       └─ agent: identity alloc
  │              BPF compile + load     (BLOCKS until done)
  │              lxcmap update
  │              policymap update
  │       EndpointResponse ◄──────────────
  │
  ├─[14] Enter netns: PMTUD + congestion control sysctl
  │
  └─[15] Print CNI result JSON → stdout → kubelet
```

---

## Error Handling

| Failure point | Cleanup |
|---|---|
| IPAM allocation succeeds, later step fails | `releaseIPsFunc` deferred — IPs returned to pool |
| veth creation succeeds, later step fails | `linkPair.Delete()` deferred — veth removed |
| Endpoint create fails | IP released, veth deleted — clean slate |

---

## Key Files

| File | Role |
|---|---|
| `plugins/cilium-cni/cmd/cmd.go` | `Add()` implementation |
| `plugins/cilium-cni/cmd/endpoint.go` | `PrepareEndpoint()` — builds agent request |
| `plugins/cilium-cni/types/types.go` | `NetConf`, `ArgsSpec` types |
| `plugins/cilium-cni/chaining/` | Per-CNI chaining ADD implementations |
| `pkg/datapath/connector/add.go` | `NewLinkPair` — veth creation |
| `pkg/datapath/connector/ipam.go` | `IPv4Routes`, `IPv4Gateway` helpers |
| `pkg/client/` | REST client for cilium-agent |
| `pkg/endpoint/` | Endpoint model types |
| `daemon/cmd/endpoint.go` | Agent-side `EndpointCreate` handler |
