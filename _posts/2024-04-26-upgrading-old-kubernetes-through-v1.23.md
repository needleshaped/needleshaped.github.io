# Upgrading old Kubernetes through v1.23

- Goal: upgrade Kubernetes from `1.22` up to `1.24` (in this document)
- Problem: repository was changed, leaving a gap in upgrade process. As a result, `1.23` had to be downloaded as binaries
  - New repository starts with `1.24.x` (see official [log post](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/#where-can-i-get-packages-for-kubernetes-versions-prior-to-v1-24-0))
- Given: OS **Debian** 10/11, bare metal, manual setup, already migrated from dockershim to `containerd`

This text assumes you know how to upgrade, focuses on v1.23 peculiarities and troubleshooting, all errors I've got myself.
Props to https://flex-solution.com/page/blog/install-k8s-lower-than-1_24, which shows, how to add a v1.23 node.

---

Reminder on general approach:

1. Upgrade `kubeadm`, run `upgrade plan` and `upgrade apply` - on 1st master
2. Upgrade `kubeadm`, run `upgrade node` - on all other masters
3. Upgrade `kubelet`/`kubectl` and restart Kubelet with new version - on all nodes

Thus, specific upgrade plan is to repeat above 3 times:

1. 1.22.0 > 1.23.7, using downloaded binaries
2. 1.23.7 > 1.24.17, using new repository

## 1.23

### Upgrading kubeadm

```sh
# Getting v1.23 binaries, includes everything
wget https://dl.k8s.io/v1.23.7/kubernetes-server-linux-amd64.tar.gz
tar -xvf ./kubernetes-server-linux-amd64.tar.gz

# using those
./kubernetes/server/bin/kubeadm version
```

#### Troubleshooting

```sh
apt update # on old repo
  # Repository 'https://apt.kubernetes.io kubernetes-xenial Release' does not have a Release file.
  # N: Updating from such a repository cant be done securely, and is therefore disabled by default.
    # https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/, new repository
    # is there one for 1.23?
    #echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.23/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt update # on new repo, attempt with v1.23
# Err:5 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.23/deb  InRelease
#   403  Forbidden [IP: 18.173.205.8 443]
# Reading package lists... Done
# E: Failed to fetch https://pkgs.k8s.io/core:/stable:/v1.23/deb/InRelease  403  Forbidden [IP: 18.173.205.8 443]
# E: The repository 'https://pkgs.k8s.io/core:/stable:/v1.23/deb  InRelease' is not signed.
  # https://forums.developer.nvidia.com/t/installing-nvidia-tao-toolkit-api/285780/12
  # No, there is NO repository for v1.23, new repository lists version 1.24 onwards

  # read
  #  - https://kubernetes.io/blog/2023/08/31/legacy-package-repository-deprecation/#what-releases-are-available-in-the-new-community-owned-package-repositories
  # - https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/#where-can-i-get-packages-for-kubernetes-versions-prior-to-v1-24-0
    # https://www.reddit.com/r/kubernetes/comments/1b685wy/is_it_still_possible_to_install_kubernetes_121/
    # https://flex-solution.com/page/blog/install-k8s-lower-than-1_24

  # have to download binaries
```

### Upgrading masters/nodes

```sh
./kubernetes/server/bin/kubeadm upgrade plan
./kubernetes/server/bin/kubeadm upgrade apply v1.23.7 # <<--- on 1st master
./kubernetes/server/bin/kubeadm upgrade node # <---- on all other control plane nodes
```

#### Troubleshooting

```sh
kubeadm upgrade node
#[ERROR ImagePull]: failed to pull image registry.k8s.io/coredns/coredns:v1.8.6: output: time="2024-05-26T13:24:40+02:00" level=fatal
#msg="validate service connection: CRI v1 image API is not implemented for endpoint \"unix:///run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.ImageService"
  # https://github.com/containerd/containerd/discussions/8706
    #For Kubernetes v1.24, you need containerd (Server) v1.6.4+ or 1.7.0+: https://github.com/containerd/containerd/blob/v1.7.2/RELEASES.md#kubernetes-support
    apt update && apt-get install containerd.io # install//upgrade

kubeadm upgrade node
#[ERROR ImagePull]: failed to pull image registry.k8s.io/coredns:v1.8.6: output: time="2024-05-25T23:07:36+02:00" level=fatal msg="pulling image: rpc error: code = NotFound
# desc = failed to pull and unpack image \"registry.k8s.io/coredns:v1.8.6\": failed to resolve reference \"registry.k8s.io/coredns:v1.8.6\": registry.k8s.io/coredns:v1.8.6: not found", error: exit status 1
  #https://github.com/kubernetes/kubeadm/issues/2761
  # revert the imageRepository registry.k8s.io back to k8s.gcr.io
  kubectl -n kube-system edit cm kubeadm-config
  # or make local tag to trick
  ctr image pull docker.io/coredns/coredns:1.8.6
  ctr image tag docker.io/coredns/coredns:1.8.6 registry.k8s.io/coredns/coredns:v1.8.6
```

### Upgrading Kubelet, kubelet restart

```sh
# reconfigure tu use  v1.23 kubelet from binaries
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
sed -i 's|/usr/bin/kubelet|/root/kubernetes/server/bin/kubelet|' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload

# restart
systemctl restart kubelet
```

## 1.24

### Upgrading kubeadm

```sh
# new repository
sudo rm /etc/apt/sources.list.d/kubernetes.list
mkdir -vp /etc/apt/keyrings/ # prior Debian 12
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.26/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg # all keys for all versions are same
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.24/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# upgrade
apt-get install -y --allow-change-held-packages kubeadm=1.24.17-1.1
```

### Upgrading masters/nodes

```sh
kubeadm upgrade plan
kubeadm upgrade apply v1.24.17 # <<--- on 1st master
kubeadm upgrade node # <---- on all other control plane nodes
```

### Upgrading Kubelet, kubelet restart

```sh
# During upgrade to 1.24, revert back from 1.23 binaries
sed -i 's|/root/kubernetes/server/bin/kubelet|/usr/bin/kubelet|' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload

# upgrade
apt-get install -y --allow-change-held-packages kubelet=1.24.17-1.1 kubectl=1.24.17-1.1
apt-mark hold kubelet kubeadm kubectl
systemctl restart kubelet

# cleanup
rm -rvf kubernetes-server-linux-amd64.tar.gz /root/kubernetes
```

#### Troubleshooting

```sh
# kubelet[350]: Error: failed to parse kubelet flag: unknown flag: --network-plugin
  cat /var/lib/kubelet/kubeadm-flags.env
  # KUBELET_KUBEADM_ARGS="--cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --rotate-certificates --rotate-server-certificates --container-runtime-endpoint=unix:///run/containerd/containerd.sock --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice --container-runtime=remote
    sed -i 's|--network-plugin=cni||' /var/lib/kubelet/kubeadm-flags.env
```
