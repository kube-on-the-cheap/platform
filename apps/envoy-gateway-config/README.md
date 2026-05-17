# Envoy Gateway Config

Cluster-specific configuration for the Envoy Gateway data plane: `GatewayClass`, `Gateway`, `EnvoyProxy`, and `ClientTrafficPolicy` resources. The Helm-installed controller lives in `apps/envoy-gateway/`; this overlay only configures the resources it manages.

## Architecture

### freeloader (OCI / OKE)

The `external` Gateway terminates HTTP/80 and HTTPS/443 with a wildcard cert for `*.cloud.blacksd.tech` (provisioned by cert-manager via DNS-01 on DigitalOcean, synced into `envoy-gateway-system` by an `ExternalSecret`).

The data-plane proxies sit behind an OCI Network Load Balancer (`oci.oraclecloud.com/load-balancer-type: nlb`), configured to:

- forward PROXY protocol v2 (`oci-network-load-balancer.oraclecloud.com/is-ppv2-enabled: "true"`),
- manage ingress NSG rules from the Service annotations (`oci.oraclecloud.com/security-rule-management-mode: NSG`),
- attach the LB to the cluster NSG `ocid1.networksecuritygroup.oc1.eu-frankfurt-1.aaaaaaaafrdfn54r6aeb3hqpsl7ob2ggffqazvnum3ort73h5ytf4h7zg6fa`.

Envoy is deployed as 2 replicas with a soft pod anti-affinity on `kubernetes.io/hostname`, so they spread across the two worker nodes when scheduling permits. The `ClientTrafficPolicy` enables PROXY protocol consumption so requests reach the upstream with the real client IP rather than the LB's VIP.

### understairs (on-prem / Talos)

See the `overlays/understairs/` files. Uses a Cilium-managed LoadBalancer rather than a cloud NLB; PROXY protocol and the OCI-specific annotations don't apply.

## Files

| File (`overlays/freeloader/`) | Resource | Purpose |
|---|---|---|
| `gatewayclass.yaml` | `GatewayClass/envoy-gateway` | Binds the class to the controller and references the `EnvoyProxy` parameters |
| `envoyproxy.nlb.yaml` | `EnvoyProxy/oci-loadbalancer` | Active config: NLB + PPv2 + 2 replicas |
| `envoyproxy.alb.yaml` | `EnvoyProxy/oci-loadbalancer` | Inactive: prior ALB-based config kept for reference. Not in `kustomization.yaml`. |
| `gateway.yaml` | `Gateway/external` | The L7 entry point: ports 80 (HTTP) and 443 (HTTPS, terminate with `wildcard-cert`) |
| `client-traffic-policy.yaml` | `ClientTrafficPolicy/preserve-client-ip` | Enables PROXY protocol consumption on the Gateway |

## History

### HTTP/3 / QUIC experiment (May 2026)

Attempted to enable HTTP/3 to reduce login-page latency on Zitadel (~20 small Next.js chunks per page load, ~120ms TTFB each, ~2.4s total chunk waterfall). Reverted after running into three independent OCI constraints that, combined, make HTTP/3 unworkable through this NLB:

1. **OCI CCM rejects NSG-managed UDP listeners.** When Envoy Gateway adds UDP/443 to the Service for QUIC, the OCI cloud-controller-manager fails with:

   ```
   Error syncing load balancer: failed to ensure load balancer:
     invalid service: Security rule management mode can only be 'None' for UDP protocol
   ```

   OCI's CCM only allows `security-rule-management-mode: None` for Services with UDP listeners. The NLB is intentionally `NSG` mode so ingress rules stay in version control with the rest of the infrastructure; switching to `None` would mean managing NSG rules out-of-band.

2. **NSG ingress rules don't allow UDP/443.** Even if (1) is solved, the worker-node NSG only permits TCP/80 and TCP/443. UDP/443 must be opened explicitly in `cloud-infra` for QUIC packets to reach the nodes.

3. **PROXY protocol v2 corrupts QUIC datagrams.** OCI's `is-ppv2-enabled` flag is global across listeners. PPv2 framing prepended to the first QUIC packet would corrupt the QUIC header, Envoy would fail to parse, and the browser would silently fall back to HTTP/2.

Browsers handle the failure gracefully (`Alt-Svc` advertisement is a hint, not a contract — clients fall back to HTTP/2 transparently), so a half-broken config produces no visible breakage, just no actual HTTP/3 traffic. The Alt-Svc header is therefore misleading and the small per-connection cost of probing QUIC is wasted.

The replica bump (1 → 2) stayed in. The latency win we were chasing is better pursued via a CDN in front of `cloud.blacksd.tech` for static asset fan-out, which would also bypass all three OCI constraints.

References:

- Envoy Gateway HTTP/3: <https://gateway.envoyproxy.io/docs/tasks/traffic/http3>
- OCI CCM source for the UDP/NSG check: search `oci-cloud-controller-manager` for `Security rule management mode can only be 'None' for UDP protocol`
