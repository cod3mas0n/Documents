---
title: Kubernetes Ephemeral Containers and kubectl debug Command
published: true
description: Deep Diving Kubernetes Ephemeral Containers and kubectl debug Command
tags: kubernetes
cover_image: 'https://raw.githubusercontent.com/AliMehraji/Documents/refs/heads/main/posts/assets/k8s-debug-ctr-in-pod-shared-ns.webP'
canonical_url: null
id: 2795240
date: '2025-08-23T22:29:01Z'
---

> This post is only a brief overview, with references collected at the end. If you’d like to dive deeper, I recommend starting with [Deep Diving Kubernetes Ephemeral Containers and kubectl debug Command][ix-posts-ephemeral-containers] and then following the related [Task: Copy Files To/From a Distroless Kubernetes Pod][ix-task-copy-files-to-from-distroless-kubernetes-pod].

### Ephemeral containers

[Ephemeral containers][k8s-docs-ephemeral-containers] provide a powerful way to debug applications running in Kubernetes. Unlike regular containers, they are not part of the pod’s original specification but can be injected into a running pod when needed. This makes them especially valuable for interactive troubleshooting, particularly when `kubectl exec` is insufficient—for example, when a container has already crashed or when the original image lacks essential debugging tools.

Modern container practices often favor minimal or `distroless` images to improve security and performance. In many cases, base images are stripped down to the essential. sometimes even using `scratch` with nothing but the application binary. While this approach:

- have a smaller attack vector area.
- have faster-scanning performance.
- Reduced image size.
- have a faster build and CD/CI cycle.
- have fewer dependencies.

It also means that traditional debugging utilities (like a shell or package manager) are unavailable.

Ephemeral containers allow engineers to inject a temporary container image that includes the required debugging tools, without altering the original pod definition. This is particularly useful for inspecting the live state of an application, reproducing elusive issues, and running arbitrary commands inside a pod—giving teams the best of both worlds: lean production images with flexible on-demand debugging capabilities.

### Share Process

While troubleshooting a Pod, I'd typically want to see the processes of all pods containers, as well as I'd be interested in exploring their filesystems. Can an ephemeral container be a little more unraveling?

However, if you explore the filesystem with `ls`, you'll notice that it's a filesystem of the ephemeral container itself (busybox in our case), and not of any other container in the Pod. Well, it makes sense - [containers in a Pod (typically) **share** `net`, `ipc`, and `uts` namespaces][shared-ns-img], they can **share** the `pid` namespace, but they ***never*** share the `mnt` namespace. Also, filesystems of different containers always stay independent. Otherwise, all sorts of collisions would start happening.

Understanding process namespace sharing:

- **The container process no longer has PID 1**. Some containers refuse to start without PID 1 (for example, containers using systemd) or run commands like `kill -HUP 1` to signal the container process. In pods with a shared process namespace, `kill -HUP 1` will signal the pod sandbox (`/pause`).

- **Processes are visible to other containers in the pod**. This includes all information visible in `/proc`, such as passwords that were passed as arguments or environment variables. These are protected only by regular Unix permissions.

- **Container filesystems are visible to other containers**. This makes debugging easier, but it also means that filesystem secrets are protected only by filesystem permissions.Thus, if you know a PID of a process from the container you want to explore, you can find its filesystem at `/proc/<PID>/root` from inside the debugging container.

### Lets get hands dirty

Create pod with a distroless image.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: distroless-nginx
  labels:
    app: distroless-nginx
spec:
  selector:
    matchLabels:
      app: distroless-nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: distroless-nginx
    spec:
      containers:
        - name: distroless-nginx-ctr
          image: cgr.dev/chainguard/nginx
```

After the pod is created, try to get shell via `kubectl exec` (try with `sh`, `ash` , ...):

```bash
k exec -it -n default distroless-nginx-cbbdbd698-49fjr -- bash
```

It wont be able to give you typical TTY shell.

```bash
error: Internal error occurred: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "c336401c1da1996fc382a2c8aefd82552a60b6de52da1d12aec0d3b5f9b3b65a" : OCI runtime exec failed: exec failed: unable to start container process: exec: "bash": executable file not found in $PATH
```

### `kubectl debug`

```bash
POD_NAME=$(kubectl get pods -l app=distroless-nginx -o jsonpath='{.items[0].metadata.name}')
kubectl debug -it -c debugger --image=busybox ${POD_NAME}
```

When you inject an `ephemeralContainer` into a pod, it behaves like any other container, unless you explicitly grant it additional privileges or enable process and filesystem sharing with the target container.

So, unless your workloads run with `shareProcessNamespace` enabled by default (which is unlikely a good idea since it reduces the isolation between containers), we still need a better solution.

```bash
POD_NAME=$(kubectl get pods -l app=distroless-nginx -o jsonpath='{.items[0].metadata.name}')
kubectl debug -it -c debugger --target=distroless-nginx-ctr --image=busybox ${POD_NAME}
```

```bash
Targeting container "distroless-nginx-ctr". If you don't see processes from this container it may be because the container runtime doesn't support this feature.
--profile=legacy is deprecated and will be removed in the future. It is recommended to explicitly specify a profile, for example "--profile=general".
If you don't see a command prompt, try pressing enter.

/ # ps aux
PID   USER     TIME  COMMAND
    1 65532     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf -e /dev/stderr -g daemon off;
    7 65532     0:00 nginx: worker process
    8 65532     0:00 nginx: worker process
    9 65532     0:00 nginx: worker process
   10 65532     0:00 nginx: worker process
   17 root      0:00 sh
   22 root      0:00 ps aux
/ #
```

In this case the [debugger container can access to other processes][after-ns-shared]. what does the `--target` do ?

```bash
k debug --help

--target='':
  When using an ephemeral container, target processes in this container name.

```

***Note***:

The `--target` parameter must be supported by the Container Runtime. When not supported, the Ephemeral Container may not be started, or it may be started with an isolated process namespace so that ps does not reveal processes in other containers.

Now, let’s say we want to modify the distroless-nginx-ctr container’s /etc/nginx/nginx.conf. Immediately we run into a problem: we don’t even have permission to change directories with cd, nor can we list the root directory of the target container.

```bash
~ # ls /proc/1/root
ls: /proc/1/root: Permission denied
~ # cd /proc/1/root
sh: cd: can't cd to /proc/1/root: Permission denied
~ #
```

This highlights a key limitation—process sharing alone isn’t enough. To properly inspect and interact with the target container’s filesystem, we also need elevated privileges. That’s where the [`--profile`][debugging-profiles] flag comes in, as it can grant the necessary access for debugging and modification.

```bash
POD_NAME=$(kubectl get pods -l app=distroless-nginx -o jsonpath='{.items[0].metadata.name}')
kubectl debug -it -c debugger --target=distroless-nginx-ctr --profile=sysadmin --image=busybox ${POD_NAME}
```

***Note***:

- kubectl debug automatically generates a container name if you don't choose one using the `--container` flag.
- The `-i` flag causes kubectl debug to attach to the new container by default. You can prevent this by specifying -`-attach=false`. If your session becomes disconnected you can reattach using `kubectl attach`.
- The `--share-processes` allows the containers in this Pod to see processes from the other containers in the Pod. For more information about how this works, see [Share Process Namespace between Containers in a Pod][share-process-namespace]
- **Don't forget** to clean up the debugging Pod when you're finished with it.
- Install a Kubernetes policy engine (OPA/Gatekeeper) to prevent [exposing your cluster][medium-share-proc-exposed]:
  - Block privileged debug containers
  - Require `automountServiceAccountToken`: false
  - Implement just-in-time access for debugging
  - Use managed debug tools like Teleport instead of raw kubectl debug
- Debug Session Monitoring Now we alert on:
  - Added Debug Session Monitoring
  - Any ephemeral container creation
  - Service account token access from unexpected pods
  - Debug sessions lasting >5 minutes

### `kubectl debug node`

```bash
k debug node/debian-11 -it --image-pull-policy='IfNotPresent' --image=ubuntu --profile=sysadmin
```

[When creating a debugging session on a node, keep in mind that][shell-debug-on-node]:

- `kubectl debug` automatically generates the name of the new Pod based on the name of the Node.
- The root filesystem of the Node will be mounted at `/host`.
- The container runs in the host IPC, Network, and PID namespaces, although the pod isn't privileged, so reading some process information may fail, and `chroot /host` may fail.
- If you need a privileged pod, create it manually or use the `--profile=sysadmin` flag.

To debug node like an SSH session, just invoke `chroot /host`.

```bash
Creating debugging pod node-debugger-debian-11-wrjhs with container debugger on node debian-11.
If you don't see a command prompt, try pressing enter.
/ # chroot /host
```

Don't forget to clean up the debugging Pod when you're finished with it:

```bash
kubectl delete pod node-debugger-debian-11-wrjhs
```

## Resources

- [Docker Containers vs. Kubernetes Pods - Taking a Deeper Look][ix-posts-containers-vs-pods]
- [Multi-container pods and container communication][mirantis-kubernetes-pod-vs-container]

- [Ephemeral Containers][ephemeral-ctrs-feature-request]
  - [Kubernetes Ephemeral Containers and kubectl debug Command][ix-posts-ephemeral-containers]
  - [FR: New kubectl command kubectl debug][FR: New kubectl command]
  - [Using Kubernetes Ephemeral Containers for Troubleshooting][ephemeral-ctr-tshoot]
  - [Ephemeral Containers][k8s-docs-ephemeral-containers]
  - [Introducing Ephemeral Containers][google-blog-ephemeral-containers]
  - [The future of Kubernetes workload debugging][medium-ephemeral-containers]
- [Share Process Namespace between Containers in a Pod][share-process-namespace]
- [kubectl Debug Exposed Kubernetes Cluster][medium-share-proc-exposed]
- [Debug node][kubectl-debug-node]
- [Debugging via a shell on the node][shell-debug-on-node]

- Tasks
  - [Debugging with an ephemeral debug container][k8s-docs-debug-pod]
  - [Copy Files To/From a Running Kubernetes Pod: a Distroless Image Case][ix-task-copy-files-to-from-distroless-kubernetes-pod]
  - [Run a Sidecar Container in the Namespace of Another Container][ix-task-docker-container-share-namespaces]

[ix-posts-ephemeral-containers]: https://iximiuz.com/en/posts/kubernetes-ephemeral-containers/
[ephemeral-ctr-tshoot]: https://www.vcluster.com/blog/using-kubernetes-ephemeral-containers-for-troubleshooting
[k8s-docs-ephemeral-containers]: https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/
[google-blog-ephemeral-containers]: https://opensource.googleblog.com/2022/01/Introducing%20Ephemeral%20Containers.html
[medium-ephemeral-containers]: https://medium.com/01001101/ephemeral-containers-the-future-of-kubernetes-workload-debugging-c5b7ded3019f

[ix-posts-containers-vs-pods]: https://labs.iximiuz.com/tutorials/containers-vs-pods
[mirantis-kubernetes-pod-vs-container]: https://www.mirantis.com/blog/kubernetes-pod-vs-container-multi-container-pods-and-container-communication/

[share-process-namespace]: https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/
[medium-share-proc-exposed]: https://medium.com/@rudra910203/the-kubectl-debug-command-that-exposed-our-entire-kubernetes-cluster-3a125ed6e539

[k8s-docs-debug-pod]: https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container
[ix-task-copy-files-to-from-distroless-kubernetes-pod]: https://labs.iximiuz.com/challenges/copy-files-to-from-distroless-kubernetes-pod
[ix-task-docker-container-share-namespaces]:https://labs.iximiuz.com/challenges/docker-container-share-namespaces

[ephemeral-ctrs-feature-request]: https://github.com/kubernetes/enhancements/issues/277
[FR: New kubectl command]: https://github.com/kubernetes/kubernetes/issues/45922

[shared-ns-img]: https://raw.githubusercontent.com/AliMehraji/Documents/refs/heads/main/posts/assets/k8s-debug-ctr-in-pod-shared-ns.webP
[after-ns-shared]: https://raw.githubusercontent.com/AliMehraji/Documents/refs/heads/main/posts/assets/ctr-in-pod-shared-pid-ns.WebP

[debugging-profiles]: https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#debugging-profiles

[kubectl-debug-node]: https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/
[shell-debug-on-node]: https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#node-shell-session
