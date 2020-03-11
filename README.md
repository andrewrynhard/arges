# Arges

## Prerequisites

Arges is built on top of [Cluster API](https://github.com/kubernetes-sigs/cluster-api).
Because of this, an existing Kuberntes cluster is required.
The cluster does not have to be based on Talos.
If you would like to use Talos as the OS of choice for the Arges control plane, you can find a number of ways to deploy Talos in the [documentation][1].

In addition to a Kubernetes cluster, the `kubectl` and `kustomize` binaries are also required.

Finally, we need to host a `kubeconfig` somewhere that machines can download from.

## Quickstart

This will deploy a number of components:

- Cluster API
- Talos Cluster API bootstrap provider
- Metal Cluster API infrastructure provider
- Metal controller manager
- and the Metal metadata server

### Deploying Arges

To get started with Arges, we need to know a few bits of information beforehand.
Determining this information is dependent on how you choose to expose the Arges `metal-ipxe`, `metal-metadata-server`, and `metal-tftp` services.
However you choose to expose these servers, we simply need to know the endpoint for each of these services.

Once you determine these, we will need to edit a few files in `example/arges`.
Let's start with `discovery_kubeconfig_patch.yaml`.

Using the endpoint for the discovery kubeconfig, run:

```bash
cat <<EOF >>example/arges/discovery_kubeconfig_patch.yaml
- op: add
  path: /spec/template/spec/containers/1/args/-
  value: --discovery-kubeconfig=$ENDPOINT
EOF
```

Now, with the `metal-metadata-server` endpoint:

```bash
cat <<EOF >>example/arges/default_environment_patch.yaml
- op: add
  path: /spec/kernel/args/-
  value: talos.config=$ENDPOINT/configdata?uuid=
EOF
```

With our patches in place, we can now deploy Arges.
To do so, run the following:

```bash
kustomize build ./example/arges | kubectl apply -f -
```

You should see the following components running as pods in the `arges-system` namespace:

```bash
$ kubectl get pods -n arges-system
NAME                                        READY   STATUS    RESTARTS   AGE
cabpt-controller-manager-7fdd65469c-pfkc9   2/2     Running   0          11s
capi-controller-manager-76798c8d9d-t8k4t    1/1     Running   0          11s
capm-controller-manager-6768c57dc5-7m2vm    2/2     Running   0          11s
metal-controller-manager-c65c7b478-srv49    2/2     Running   0          10s
metal-metadata-server-6b754c8d84-v9sb4      1/1     Running   0          10s
```

At this point, there are numerous ways to expose the necessary services (`metal-ipxe`, `metal-metadata-server`, and `metal-tftp`) so that we can begin to create clusters.
It is assumed some familiarity with Kubernetes and leave this as an exercise to the reader.
Please, expose these services now, ensuring that the endpoints align with the previous steps.

> Note: It is common to use an ingress controller to expose services, however, BGP may also be used.

#### Register the Machines

Once we have the `metal-ipxe`, `metal-metadata-server`, and `metal-tftp` services exposed, we will use them to boot a lightweight environment that will register the machine with Arges, making it available to provision as a Kubernetes node.

To do this, you must ensure that your network is setup such that the machine can and will PXE boot using the `metal-tftp` and `metal-ipxe` services.
The way in which this can be done varies greatly, so we leave it as an exercise to the reader.

> Important: _It is crucial to get this step right, or machnines will not use the Arges services_.

Once you have the services exposed, and can ensure that the machines can and will boot using these services and is able to download the `kubeconfig`, we are ready to register machines with Arges.

To register machines, simply power them on.
The machines should do a number of things:

- boot using the deployed TFTP, and iPXE services in Arges
- download the discovery `kubeconfig`
- create a custom `server` resource in the Kubernetes cluster hosting Arges

### Creating a Cluster

Once we have all the machines registered that we would like to be available as resources for creating clusters, we can start creating clusters.

#### Generate the Talos Configuration Files

We will start by creating the configs required by Talos:

```bash
osctl config generate example-0 https://192.168.1.101:6443
```

> Note: Be sure to update the port and IP address that you will use for the cluster's API endpoint.

You should see four files in the directory:

- `init.yaml`
- `controlplane.yaml`
- `join.yaml`
- `talosconfig`

At this point, you can make edits the Talos configs.

In this example, we will designate `controlplane-0` as the `init` node.
Copy the contents of `init.yaml` into the `spec.data` field of `./talosconfigs/controlplane-0.yaml`.

This same thing should be done for `controlplane.yaml` and `join.yaml`.

> For more on the concept of an `init` node, see the Talos [documentation][1].

#### Create the Cluster

Now that we have a PXE boot environment, and the Talos config files, we are ready to create our cluster.
To do this, run the following from within the `example/cluster` directory:

```bash
kubectl apply -k .
```

[1]: https://www.talos.dev "Talos"
