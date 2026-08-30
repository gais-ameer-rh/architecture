# Distributed Zones with BGP and Ceph

This deployment topology is for testing only. It reuses the [dz-storage](../dz-storage)
network, networker, and VM definitions, but replaces the external storage arrays with
three independent single-node Ceph clusters.

Each rack assigns `r*-compute-0` to Nova and `r*-compute-1` to Ceph. The Ceph
nodes retain their existing VM names so the topology can directly reuse the
`dz-storage` infrastructure definitions. A production deployment requires
additional Ceph nodes for redundancy and normally has more Nova computes per
availability zone.

The automation in `automation/vars/dz-ceph.yaml` defines these phases:

1. Deploy the shared `dz-storage` networking and a pre-Ceph control plane
   with its storage-backed services disabled.
2. Create separate Nova, Ceph, and networker data-plane NodeSets.
3. Prepare the Ceph-only nodes with one Ceph cluster per rack.
4. Reapply the control plane with the generated Ceph configuration and run
   the post-Ceph Nova compute node deployments.

## Shared configuration

This topology intentionally references the `dz-storage` networking, topology,
networker, and VM definitions instead of copying them. Customize the values in
`examples/dt/dz-storage` when directed by the deployment guides, but build the
Nova and Ceph NodeSets from the `dz-ceph` overlays. Those overlays divide each
rack's two compute VMs by role: `compute-0` runs Nova and `compute-1` runs Ceph.

The shared environment preparation is also documented by `dz-storage`:

- [Configure the tester-node taints](../dz-storage/configure-taints.md)
- [Disable reverse-path filtering](../dz-storage/disable-rp-filters.md)
- [Install the OpenStack K8S operators and their dependencies](../../common/)
- [Apply metallb customization required to run a speaker pod on the OCP tester node](metallb/)
- [Define zones and topologies](../dz-storage/topology/)
- [Create the BGPConfiguration](../dz-storage/bgp-configuration.md)

## Deployment stages

Run the stages in this order:

1. [Configure networking and deploy the pre-Ceph control plane](control-plane.md).
2. [Create the Nova, Ceph, and networker NodeSets and run the initial EDPM
   deployment](data-plane.md).
3. Deploy Ceph in each Availability Zone
4. [Update the control plane for Ceph and finish the Nova
   deployments](post-ceph.md).
