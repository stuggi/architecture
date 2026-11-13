# HCI with MetalLB-backed public endpoints

The [hci](../hci/README.md) VA with the public endpoints of **all deployed API
services** (barbican, cinder, glance, horizon, keystone, neutron, nova, its cell1 noVNC proxy, and placement)
served by MetalLB `LoadBalancer` services instead of OpenShift Routes.

- All services share the VIP `192.168.122.85` of the `ctlplane` MetalLB pool
  (`allow-shared-ip`), so no networking change is needed. Every service has its own
  front port, so they do not collide on the shared VIP.
- Clients reach them by FQDN, `<service>.public.openstack.lab:<port>`, set through
  `override.service.*endpointURL`: keystone `:13000`, nova `:13774`, neutron `:13696`,
  glance `:13292`, cinder `:13776`, barbican `:13311`, placement `:13778`, nova
  noVNC proxy (cell1) `:13080`, horizon on 443.
  Those names must resolve to the VIP for every client, in-cluster pods and EDPM
  nodes included, and are **not** registered by the operators. CI Framework
  registers them in its dnsmasq from the overlay.
- The placement public endpoint uses a bring-your-own certificate
  (`placement-public-byo-cert`, CA in `byo-ca-bundle`), every other service stays
  operator-issued, so both paths run. The secrets must exist before the control plane is
  applied; CI Framework's `hooks/playbooks/public_endpoints_byo_cert.yml` creates them.
- Not converted: the services `hci` disables (designate, heat, ironic, manila, octavia,
  swift, telemetry, watcher), which have no public endpoint here.
- Only the `control-plane` and `edpm-post-ceph` stages differ from hci: the post-ceph stage
  re-applies the control plane, so it carries the same overlay (`overlay/`), otherwise
  that apply would remove it. The automation file is
  [hci-public-lb.yaml](../../../automation/vars/hci-public-lb.yaml).

Only services whose operator sets the public vhost `ServerName` from
`endpointURL` can be converted; extend the patch in
`overlay/kustomization.yaml` as other operators gain that.

## Enabling and reverting

The feature is selected by the scenario: run CI Framework with
`cifmw_architecture_scenario: hci-public-lb` instead of `hci`. Switching back to
`hci` restores Routes; nothing else changes.

The `control-plane` stage runs two CI Framework hooks beforehand
(`hooks/playbooks/public_endpoints_byo_cert.yml` and `public_endpoints_dns.yml`),
so a CI Framework revision that contains them is required (`Depends-On` while
unmerged). The DNS hook writes to the `hypervisors` inventory group; its records
are removed by running it with `cifmw_public_endpoints_dns_state=absent`.
