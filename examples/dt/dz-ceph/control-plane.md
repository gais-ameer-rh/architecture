# Deploy the pre-Ceph control plane

The `dz-ceph` topology reuses the `dz-storage` network and topology values. The
initial control-plane overlay disables the storage-backed services until the
three Ceph clusters exist.

Run these commands from the root of the `architecture` repository and use the
`openstack` project:

```shell
oc project openstack
```

## Customize the shared values

Edit the following files for the target environment:

- `examples/dt/dz-storage/control-plane/networking/nncp/values.yaml` contains
  the OpenShift node network and service-network values.
- `examples/dt/dz-storage/control-plane/service-values.yaml` contains the
  remaining control-plane service settings used by the shared base.
- `examples/dt/dz-storage/topology/node-zone-labels.yaml` assigns the OpenShift
  nodes to the three zones.

The external-storage secrets in the `dz-storage` service values do not become
part of `dz-ceph`; its overlays remove those resources.

## Apply the node network configuration

Build and apply the shared NNCP kustomization:

```shell
kustomize build examples/dt/dz-storage/control-plane/networking/nncp > nncp.yaml
oc apply -f nncp.yaml
oc -n openstack wait nncp \
  -l osp/nncm-config-type=standard \
  --for jsonpath='{.status.conditions[0].reason}'=SuccessfullyConfigured \
  --timeout=600s
```

## Apply the remaining networking and topology

Build and apply the shared networking and topology kustomizations:

```shell
kustomize build examples/dt/dz-storage/control-plane/networking > networking.yaml
oc apply -f networking.yaml

kustomize build examples/dt/dz-storage/topology > topology.yaml
oc apply -f topology.yaml
```

See the shared [MetalLB guide](../dz-storage/metallb/) and [topology
guide](../dz-storage/topology/) for the environment preparation associated
with these resources.

## Apply the pre-Ceph control plane

Build the `dz-ceph` pre-Ceph overlay. It uses the shared network and service
values edited above, creates an empty `ceph-conf-files` Secret, and leaves
Cinder volume and backup services, Glance APIs, and Manila disabled.

```shell
kustomize build examples/dt/dz-ceph/control-plane/pre-ceph > control-plane.yaml
oc apply -f control-plane.yaml
oc -n openstack wait openstackcontrolplane controlplane \
  --for condition=Ready \
  --timeout=180m
```

After the control plane is ready, create the BGP configuration as described in
the shared [BGP configuration guide](../dz-storage/bgp-configuration.md).
