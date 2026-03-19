# k3s Cluster Build Automation

This Ansible project builds an HA Kubernetes cluster with `k3s`, Cilium, `kube-vip`, `cert-manager`, `external-dns`, and more.

The foundation is based very heavily on [Timothy Stewart's k3s-ansible automation](https://github.com/timothystewart6/k3s-ansible), but with a more opinionated and limited set of CNI options. His work was based on a few other community playbooks noted in the [Thanks](#thanks) section, all of which you should check out.

> [!WARNING]
> If you are interested in this repo, **please** start by checking out [the upstream project](https://github.com/timothystewart6/k3s-ansible) instead, as its CNI options are far more comprehensive and probably meet your needs better than my fork.
>
> Also note this project is very much in flux at the moment and several manifests are not committed yet. But it's close!


## Architecture Overview

### Components

| Component | Notes |
| --------- | ----- |
| [**k3s**](https://k3s.io/) | Kubernetes itself in a more compact and easier to install/manage distribution. |
| [**Cilium**](https://cilium.io/) | CNI/Networking plugin for Kubernetes. Uses eBPF for maximal performance, and offers better scaling characteristics. A bit more on the advanced side than typical plugins for small k3s installations. |
| [**kube-vip**](https://kube-vip.io/) | Only used to provide HA for control plane endpoint. All its service LB management functionality is disabled in favor of Cilium-managed LBs and the native Gateway API. |
| [**external-dns**](https://kubernetes-sigs.github.io/external-dns/) | Automatic DNS record management. Currently configured to target a private/internal PowerDNS instance through its API, but has potential to work for other use cases. |
| [**cert-manager**](https://cert-manager.io/) | Automatic issuance and rotation of HTTPS certificates for exposed endpoints. Currently bootstrapped using a private intermediate CA, but may be adopted to optionally work with ACME later as well. |
| [**ArgoCD**](https://argo-cd.readthedocs.io/en/stable/) | Allows declarative management of apps; monitors a git repo for apps and manifests, and applies them to the cluster automatically. |


### Networking

The cluster exposes a control plane VIP from whichever server is elected leader. In the event a different leader is elected, that VIP moves and issues a GARP to keep cluster-external downtime to a minimum for control plane API. This is managed by kube-vip.

All external-facing apps are issued a Cilium-managed VIP assigned from a Cilium IPAM pool (instead of MetalLB for instance).

Ingress to those services is handled by Cilium's Gateway API implementation (instead of Traefik). Egress is subject to Cilium policies.

Service VIPs are advertised via BGP to the upstream router. In a multi-node cluster, this naturally results in ECMP load balancing across available nodes.

I do not run these service VIPs on the same subnet that my general LAN uses since I have some L3 forwarding/filtering rules between subnets and wifi networks, and I wanted to maintain L3 separation and control there. Obviously I am insane so YMMV.

As a result, this project currently requires an upstream BGP-enabled router. Whether that's [FRR](https://frrouting.org/), BIRD, etc. makes no real difference, but I've been using FRR at home for a long time. It's great.


## Usability Goals

* Maintain application manifests in git. Commit an app with manifests and overrides, magically get a fully deployed app with the below stuff.

* Applications get a routed VIP automatically.
  * No static routes for the VIP subnet, as the subnet could be subdivided to different k3s clusters with minimal fuss.

* Applications get automatic DNS entries pointing to that VIP.
  * TTLs on these entries should be very low to reduce user-facing downtime in the event an application has to be re-deployed on a different cluster.

* Applications get automatic HTTPS certificates from an intermediate CA created explicitly for this cluster's use.
  * All automatic certs should be trusted, as the root CA is already trusted by all my machines.

* Pod egress traffic leaving the cluster is not considered trusted.
  * Traffic destinations outside of the cluster must be allow-listed. This applies both to internal private traffic and internet-destined traffic.


## Requirements

### Control System Requirements

TODO


### Server/Agent Requirements

TODO


## Getting Started

### Preparation

TODO


### Create Cluster

Start provisioning of the cluster using the following command:

```bash
ansible-playbook site.yml -i inventory/my-cluster/hosts.ini
```

After deployment control plane will be accessible via virtual ip-address which is defined in inventory/group_vars/all.yml as `apiserver_endpoint`.


### Remove k3s cluster

```bash
ansible-playbook reset.yml -i inventory/my-cluster/hosts.ini
```

>You should also reboot these nodes due to the VIP not being destroyed


### Kube Config

To copy your `kube config` locally so that you can access your **Kubernetes** cluster run:

```bash
scp root@master:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

Edit the file to change `server: https://127.0.0.1:6443` to match your master IP: `server: https://192.168.1.222:6443`


### Testing your cluster

TODO


### Role Variables

See [CONFIG.md](CONFIG.md). Not up to date yet, but it is the upstream documentation.


### Testing

TODO


### Pre-commit Hooks

This repo uses `pre-commit` and `pre-commit-hooks` to lint and fix common style and syntax errors.  Be sure to install python packages and then run `pre-commit install`.  For more information, see [pre-commit](https://pre-commit.com/)


## Contributions

At this stage I am not seeking contributions yet. A lot of things are in flux so I'm still reconciling parts of the logic with the vision in my head.

I'm more than happy to discuss and consider/share ideas in an issue or discussions topic, but of course there is no guarantee any suggestions or requests will be implemented here.


## Thanks

This repo builds heavily on the hard work from many others in the open source community. Thank you to those who published these repos and everyone who has contributed to them. This fork will always remain licensed under the Apache Public License to match the upstream repo.

- [timothystewart6/k3s-ansible](https://github.com/timothystewart6/k3s-ansible)
- [k3s-io/k3s-ansible](https://github.com/k3s-io/k3s-ansible)
- [geerlingguy/turing-pi-cluster](https://github.com/geerlingguy/turing-pi-cluster)
- [212850a/k3s-ansible](https://github.com/212850a/k3s-ansible)
