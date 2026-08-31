# Networking

The infrastructure layer underneath everything else in this repo — how traffic actually gets
routed to the right machine, and how machines are isolated from each other and from the public
internet. (Service discovery — how one service finds *another service's* current address — is a
microservices-layer concern covered in `architecture-styles/microservices/`, not here; this
folder is about the network substrate underneath that.)

## VPC (Virtual Private Cloud)

An isolated, software-defined network within a cloud provider — your own private IP address
space, not shared with other tenants, with full control over subnetting, routing, and what can
reach it from outside. The foundational isolation boundary cloud infrastructure is built on top
of.

- **Subnets**: a VPC is divided into subnets, typically **public** (has a route to an internet
  gateway — load balancers, bastion hosts) and **private** (no direct inbound route from the
  internet — application servers, databases). The standard pattern: public subnet for the load
  balancer only, everything else in private subnets, reachable only from inside the VPC (or via
  the load balancer) — minimizes what's directly internet-addressable at all.
- **Route tables**: control which subnet's traffic goes where (to the internet gateway, to
  another VPC via peering, to a VPN/Direct Connect for on-prem connectivity).
- **Security groups / NACLs**: security groups are stateful, instance-level firewalls (allow rules
  only — if inbound is allowed, the matching outbound response is automatically allowed);
  Network ACLs are stateless, subnet-level firewalls (explicit allow *and* deny rules, evaluated
  in order). Defense in depth: NACLs as a coarse subnet-level boundary, security groups for
  precise per-instance/service rules.
- **VPC peering / Transit Gateway**: connecting separate VPCs (different environments, different
  accounts, different teams) — peering is point-to-point and doesn't transitively route (A-B and
  B-C peered doesn't give A-C connectivity); a Transit Gateway acts as a hub for many VPCs when
  the topology outgrows pairwise peering.

## Load balancing

- **Layer 4 (transport)**: routes based on IP/port only, no visibility into the actual request
  content — fast, protocol-agnostic (works for anything over TCP/UDP, not just HTTP).
- **Layer 7 (application)**: routes based on request content (HTTP path, host header, headers) —
  enables path-based routing (`/api/*` to one service, `/static/*` to another), needed for
  anything beyond simple even distribution across identical backends.
- **Balancing algorithms**: round-robin (simple, ignores actual backend load), least-connections
  (routes to whichever backend has fewest active connections — better under uneven request
  durations), weighted (proportion traffic by declared backend capacity — used for canary
  releases: send 5% of traffic to the new version).
- **Health checks**: the load balancer needs to actively probe backend health (not just wait for
  a connection failure) to pull a failing instance out of rotation before it accumulates errors —
  the gap between "instance is unhealthy" and "load balancer notices" is exactly the window users
  see errors.

## DNS

- Resolution is hierarchical and heavily cached at every layer (OS, browser, resolver, ISP) — this
  is precisely why DNS changes (a failover, a cutover) aren't instant even with correct
  configuration: clients holding a cached record from before the change keep using it until that
  specific cache's TTL expires. Lowering the TTL *before* a planned change (giving caches time to
  pick up the shorter TTL) is the standard way to make a future cutover propagate faster.
- **GeoDNS / latency-based routing**: returns different IPs depending on the resolver's location,
  routing users to their nearest region/data center — a coarse, DNS-layer form of the same
  "closest to the user" idea a CDN implements at the content layer.
- **DNS-based failover**: health-checked DNS records that stop being returned when a target is
  unhealthy — a real failover mechanism, but bounded by the TTL/caching problem above; it's a
  slower failover path than a load balancer pulling an instance from rotation, not a substitute
  for one.

## Staff-engineer notes

- Least-privilege network segmentation (private subnets by default, security groups scoped to
  exactly the traffic that's actually needed, not "allow all internal") is cheap to design in
  from the start and expensive to retrofit after a security review flags an overly-permissive
  network — treat it as a default posture, not a hardening pass done later.
- DNS TTL and caching behavior is a recurring source of "why isn't my change taking effect"
  confusion during incidents and migrations — check the TTL and cache layers explicitly before
  assuming a DNS-level fix has propagated, especially under incident time pressure.
- Load balancer health check configuration (interval, threshold, timeout) directly trades
  detection speed against false-positive risk (too aggressive pulls healthy-but-briefly-slow
  instances out of rotation, worsening load on the remaining ones) — this is worth tuning
  deliberately against real latency/failure data, not left at a generic default.
