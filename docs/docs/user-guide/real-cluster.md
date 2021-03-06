# Installing Virtlet on a real cluster

For Virtlet to work, the following prerequisites have to be fulfilled
on the nodes which will run them:

1. Node names must be resolvable via DNS configured on the nodes
1. SELinux must be disabled on the nodes (apparmor is currently supported)

Virtlet deployment consists of preparing the nodes and then deploying
the Virtlet DaemonSet.

# Installing CRI Proxy

Virtlet requires [CRI Proxy](https://github.com/Mirantis/criproxy)
package to be able to run as DaemonSet on the nodes and support
runnings system pods like `kube-proxy` there. To install CRI Proxy,
please follow the steps from its
[documentation](https://github.com/Mirantis/criproxy/blob/master/README.md).
Repeat it on each node that's going to run Virtlet.

# Deploying Virtlet DaemonSet

## Applying apparmor profiles
Follow [this guide](apparmor) if you deploy
the Virtlet DaemonSet on an apparmor-enabled environment.
##

First, you need to apply `extraRuntime=virtlet` label to each node that will run Virtlet DaemonSet (replace `XXXXXX` with the node name):
```
kubectl label node XXXXXX extraRuntime=virtlet
```

Then you need to install image translations configmap. You can use the default one:
```
curl https://raw.githubusercontent.com/Mirantis/virtlet/master/deploy/images.yaml >images.yaml
kubectl create configmap -n kube-system virtlet-image-translations --from-file images.yaml
```

After that, you need to get `virtletctl` command line tool (replace `N.N.N` in the command below accordingly):
```
curl -SL -o virtletctl https://github.com/Mirantis/virtlet/releases/download/vN.N.N/virtletctl
```
In case if you're using Mac OS X, you need to use this command instead:
```
curl -SL -o virtletctl https://github.com/Mirantis/virtlet/releases/download/vN.N.N/virtletctl.darwin
```
You can also use `virtletctl` from Virtlet image, see below.

Then you can deploy Virtlet:
```
./virtletctl gen | kubectl apply -f -
```

If you want to use the latest image, you can use `virtletctl` from that image:
```
docker run --rm mirantis/virtlet:latest virtletctl gen --tag latest | kubectl apply -f -
```
You can also use other image tag instead of `latest`, just replace it in both places
in the above command.

By default it has KVM enabled, but you can configure Virtlet to
disable it.  In order to do so, create a configmap named
`virtlet-config` in `kube-system` prior to creating Virtlet DaemonSet
that contains key-value pair `disable_kvm=y`:
```
kubectl create configmap -n kube-system virtlet-config --from-literal=disable_kvm=y
```

After completing this step, you can look at the list of pods to see
when Virtlet DaemonSet is ready:
```
kubectl get pods --all-namespaces -o wide -w
```

# Testing the installation

## Checking basic pod startup

To test your Virtlet installation, start a sample VM:
```
kubectl create -f https://raw.githubusercontent.com/Mirantis/virtlet/master/examples/cirros-vm.yaml
kubectl get pods --all-namespaces -o wide -w
```

And then connect to console:
```
$ kubectl attach -it cirros-vm
If you don't see a command prompt, try pressing enter.
```

Press enter and you will see:

```
login as 'cirros' user. default password: 'gosubsgo'. use 'sudo' for root.
cirros-vm login: cirros
Password:
$
```

Escape character is ^]

## Verifying ssh access to a VM pod

You can also ssh into the VM using
[virtletctl tool](../../reference/virtletctl/) (available as part
of each Virtlet release on GitHub starting from Virtlet 1.0).

```
virtletctl ssh cirros@cirros-vm -- -i examples/vmkey
```

## Verifying accessing services from a VM pod

After connecting to the VM using one of the above methods you can check access
from the VM to cluster services. To check DNS resolution of cluster services,
use the following command:

```
nslookup kubernetes.default.svc.cluster.local
```

The following command may be used to check service connectivity (note that
it'll give you an authentication error):

```
curl -k https://kubernetes.default.svc.cluster.local
```

You can also verify Internet access from the VM:

```
curl -k https://google.com
ping -c 1 8.8.8.8
```

If you have Kubernetes Dashboard installed (it's present in
kubeadm-dind-cluster installations for example), you can check
dashboard access using this command:

```
curl http://kubernetes-dashboard.kube-system.svc.cluster.local
```

This should display some html from the dashboard's main page.

# Removing Virtlet

In order to remove Virtlet, first you need to delete all the VM pods.

You can remove Virtlet DaemonSet with the following command:
```
kubectl delete daemonset -R -n kube-system virtlet
```

After doing this, remove CRI proxy from each node by reverting the
changes in Kubelet flags, e.g. by removing
`/etc/systemd/system/kubelet.service.d/20-virtlet.conf` in case of
kubeadm scenario described above. After this you need to restart
kubelet and remove the CRI Proxy binary (`/usr/local/bin/criproxy`)
and its node configuration file (`/etc/criproxy/node.conf`).

# Customizing Virtlet per-node configuration

It's possible to specify per-node configuration options for Virtlet.
See [this document](../../reference/config/) for more information.
