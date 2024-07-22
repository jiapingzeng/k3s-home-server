## Kubernetes Home Media Server

---

### What is this?

A home media server deployed on [k3s](https://k3s.io/) using [Lima-vm](https://lima-vm.io/). Tested on a Mac mini (M2 base model).

At the current project state, config and media files are stored locally on the host machine at `~/config` and `~/media` then mounted to the VM under `/mnt`. The directories are then used as `local` type PVs and requested by the corresponding pods through PVCs. The pods writes media files to the `~/media` folder which is then picked up by [Jellyfin](https://jellyfin.org/) installed directly on the host machine.

Includes the following containers:
- prowlarr
- sonarr
- radarr
- qbittorrent
- flaresolverr
- nginx

I do plan on adding `home-assistant` and `pi-hole` to this cluster in the near future. Stay tuned!

### Why Kubernetes

Definitely overkill and the same result can be achieved via direct installs or `docker-compose` with a lot less effort. However, it is a fun project with Kubernetes and works well once everything is set up.

Running Kubernetes or any containers on macOS is not ideal due to needing an extra layer of Linux VM, but allows running additional services such as macOS update server, AltServer (for iOS sideloading) simutaneously. I do plan on testing out [Asahi Linux](https://asahilinux.org/) in the future for better performance, if time allows.

### Installation steps:

1. Clone this repo
```
$ git clone https://github.com/jiapingzeng/k3s-home-server.git
$ cd k3s-home-server
```
2. Install [Lima](https://lima-vm.io/). Skip to step 4 if installing k3s directly on the host machine.
```
$ brew install lima
```
3. Create an Ubuntu VM with k3s installed. The configuration that I am using is based on the k3s VM configuration provided by Lima, with mounts in `~/media` and `~/config` and port forwarding so that the services can be accessed on LAN. Modify as needed.
```
$ limactl start k3s.yaml
```
4. Install [`kubectl`](https://kubernetes.io/docs/tasks/tools/) if not installed already, configure either `$KUBECONFIG` or `~/.kube/config`, then create the PVs:
```
$ brew install kubectl
$ export KUBECONFIG="{{.Dir}}/copied-from-guest/kubeconfig.yaml"
$ kubectl apply -f volume/
```
5. Deploy the applications and corresponding services:
```
$ kubectl apply -f apps/
```
6. Verify that all pods are in `Running` status and that the services are created:
```
$ kubectl get pod -n default
$ kubectl get svc -n default
```
Note that the services for the below apps are using the following `NodePort`, so that they can be configured via browser at `http://127.0.0.1:<NodePort>`, or accessed on another device on the same LAN at `http://<SERVER_IP>:<NODEPORT>`. 
```
sonarr: 31002
radarr: 31003
prowlarr: 31004
nginx -> qbittorrent: 31005
```
Optionally, Bonjour (mDNS service included macOS, `<NAME>.local`) or [avahi](https://wiki.archlinux.org/title/Avahi) can be used to access the server on your LAN via mDNS.

7. Once everything is working in browser, set `sonarr` and `radarr` as apps in `prowlarr` settings, then set `qbittorrent` as the download client in `sonarr` and `radarr` settings. Note that when configuring server URLs, the local address can be used which will be resolved by Kubernete's `CoreDNS`. For example, the server address of `prowlarr` is simply
```
http://prowlarr.default.svc.cluster.local:9696
```
This ensures that we do not need to update the config when the `ClusterIP` of the services change, if a redeploy or reboot is needed.

8. Configure VPN, if desired, either directly on the host machine or on `qBittorrent`.

9. Since the media files are saved to the host, I installed [Jellyfin](https://jellyfin.org/) directly on the host to stream to other devices on LAN, reading from `~/media/tv` and `~/media/movies`. This allows for easier configuration for using [hardware acceleration](https://jellyfin.org/docs/general/administration/hardware-acceleration/) when transcoding.