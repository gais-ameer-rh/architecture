# Prepare the data plane for Ceph

This stage creates separate Nova and Ceph NodeSets from the shared
`dz-storage` compute definitions. In each rack, `compute-0` remains in the
Nova NodeSet and `compute-1` is moved to a Ceph-only NodeSet. The networker
NodeSets are used without modification.

Run the commands from the root of the `architecture` repository after the
[pre-Ceph control plane](control-plane.md) is ready.

```shell
oc project openstack
```

## Customize the shared NodeSet values

Edit the following shared compute values for the host names, addresses,
network configuration, and other environment-specific settings in each rack:

- `examples/dt/dz-storage/edpm/computes/r0/values.yaml`
- `examples/dt/dz-storage/edpm/computes/r1/values.yaml`
- `examples/dt/dz-storage/edpm/computes/r2/values.yaml`

Each file supplies values to both the corresponding Nova overlay and the Ceph
overlay. Do not remove `compute-1` from these files; the `dz-ceph` overlays
perform the role split.

Edit the shared networker values as well:

- `examples/dt/dz-storage/edpm/networkers/r0/values.yaml`
- `examples/dt/dz-storage/edpm/networkers/r1/values.yaml`
- `examples/dt/dz-storage/edpm/networkers/r2/values.yaml`

The initial deployment membership is defined in
`examples/dt/dz-ceph/edpm/deployment/values.yaml`. Its default list includes
all three Nova, Ceph, and networker NodeSets.

## Build and apply the Nova NodeSets

```shell
kustomize build examples/dt/dz-ceph/edpm/computes/r0 > edpm-r0-compute-nodeset.yaml
kustomize build examples/dt/dz-ceph/edpm/computes/r1 > edpm-r1-compute-nodeset.yaml
kustomize build examples/dt/dz-ceph/edpm/computes/r2 > edpm-r2-compute-nodeset.yaml

oc apply -f edpm-r0-compute-nodeset.yaml
oc apply -f edpm-r1-compute-nodeset.yaml
oc apply -f edpm-r2-compute-nodeset.yaml

oc -n openstack wait openstackdataplanenodeset r0-compute-nodes \
  --for condition=SetupReady --timeout=600s
oc -n openstack wait openstackdataplanenodeset r1-compute-nodes \
  --for condition=SetupReady --timeout=600s
oc -n openstack wait openstackdataplanenodeset r2-compute-nodes \
  --for condition=SetupReady --timeout=600s
```

## Build and apply the Ceph-only NodeSets

```shell
kustomize build examples/dt/dz-ceph/edpm/ceph/r0 > edpm-r0-ceph-nodeset.yaml
kustomize build examples/dt/dz-ceph/edpm/ceph/r1 > edpm-r1-ceph-nodeset.yaml
kustomize build examples/dt/dz-ceph/edpm/ceph/r2 > edpm-r2-ceph-nodeset.yaml

oc apply -f edpm-r0-ceph-nodeset.yaml
oc apply -f edpm-r1-ceph-nodeset.yaml
oc apply -f edpm-r2-ceph-nodeset.yaml

oc -n openstack wait openstackdataplanenodeset r0-ceph-nodes \
  --for condition=SetupReady --timeout=600s
oc -n openstack wait openstackdataplanenodeset r1-ceph-nodes \
  --for condition=SetupReady --timeout=600s
oc -n openstack wait openstackdataplanenodeset r2-ceph-nodes \
  --for condition=SetupReady --timeout=600s
```

## Build and apply the networker NodeSets

Because these NodeSets are shared unchanged, build them from `dz-storage`:

```shell
kustomize build examples/dt/dz-storage/edpm/networkers/r0 > edpm-r0-networker-nodeset.yaml
kustomize build examples/dt/dz-storage/edpm/networkers/r1 > edpm-r1-networker-nodeset.yaml
kustomize build examples/dt/dz-storage/edpm/networkers/r2 > edpm-r2-networker-nodeset.yaml

oc apply -f edpm-r0-networker-nodeset.yaml
oc apply -f edpm-r1-networker-nodeset.yaml
oc apply -f edpm-r2-networker-nodeset.yaml

oc -n openstack wait openstackdataplanenodeset r0-networker-nodes \
  --for condition=SetupReady --timeout=600s
oc -n openstack wait openstackdataplanenodeset r1-networker-nodes \
  --for condition=SetupReady --timeout=600s
oc -n openstack wait openstackdataplanenodeset r2-networker-nodes \
  --for condition=SetupReady --timeout=600s
```

## Run the initial EDPM deployment

```shell
kustomize build examples/dt/dz-ceph/edpm/deployment > edpm-deployment.yaml
oc apply -f edpm-deployment.yaml
oc -n openstack wait openstackdataplanedeployment edpm-deployment \
  --for condition=Ready \
  --timeout=180m
```

## Deploy the Ceph clusters

This is the boundary between the initial and post-Ceph stages.

A separate Ceph cluster needs to be deployed in each availability zone.

In a test environment, after the initial EDPM deployment, the ci-framework playbook
[ceph-multiple.yml](https://github.com/openstack-k8s-operators/ci-framework/blob/main/hooks/playbooks/ceph-multiple.yml)
could be used to deploy ceph on each `compute-1` host for three single-node
Ceph clusters.

The following files then need to be upated with the CephX identity and other
Ceph client configuration.

- `examples/dt/dz-ceph/control-plane/ceph-conf-files.yaml`
- `examples/dt/dz-ceph/control-plane/ceph-storage.yaml`
- `examples/dt/dz-ceph/edpm-post-ceph/nova-ceph/r0/values.yaml`
- `examples/dt/dz-ceph/edpm-post-ceph/nova-ceph/r1/values.yaml`
- `examples/dt/dz-ceph/edpm-post-ceph/nova-ceph/r2/values.yaml`

Do not proceed to the post-Ceph stage while these files still contain empty
client data or `CHANGE_TO_CEPH_FSID_*` placeholders.
