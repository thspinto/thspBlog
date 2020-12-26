---
title: "RPi K3s"
date: 2020-11-20T08:40:49-03:00
lastmod: 2020-12-26T010:40:00-03:00
draft: false
screenshot: /k3s.jpg
tags: [ansible, rbp, rbi, raspberry, k3s]
---

![K3s logo](/k3s.png)

# Back with the RPis

Last time I was playing with the Raspberry Pi I used ansible to configure docker and some services on it. I still wasn't happy with the solution: my RPi 2 was really slow to run the Ansible configuration and the setup couldn't really recover from any errors without manual intervention.

Now I've upgraded my hardware and certainly over complicated the software for a small home server, but now I got metrics =).

# Hardware

Since I use Kubernetes on a daily basis I decided to upgrade my home setup to a k3s cluster. I've started with 3 nodes:

- RPi 2 model B
- RPi 3
- RPi 4 4GB ram

I added "so much" hardware because my objective is to actively monitor the cluster using Prometheus and Grafana.

The oldest node `RPi 2 model B` got decommissioned in the process. It wasn't consistent and had issues joining the cluster. He'll probably turn in to a weather station. I replaced it with RPi 4 to run the cluster with HA etcd. In the latest configuration all 3 nodes are master nodes and compose an cluster for K3s to use.

# OS bootstrapping

This first problem I didn't want to deal manually was configuring the ssh on the RPis. The previous process I had one playbook to configure the ssh and user on Raspberian, and another to configured everything else. This time, though, the node will boot with password login disabled and with my public keys automatically added to the host. To achieve this I used [Raspberian Firstboot](https://github.com/nmcclain/raspberian-firstboot) with a [custom script](https://github.com/thspinto/rpi-cluster/tree/main/os_bootstrap) to setup the ssh configuration and hostname.

All I have to do now I burn the Rapberian Firstboot image to the SD card and put the `firstboot.sh` and `bootstrap.env` in the `boot` dir. After the node boots up we can proceed directly to the Ansible configuration.

# Ansible

Ansible is responsible for the initial setup of the nodes. The playbook optimizes the RPi to run headless, configures k3s and my local kubectl context with the new cluster. Since all k3s resources are managed by ArgoCD ansible also installs it in the cluster.

![Meme of spiderman pointing to another spiderman](/spiderman-meme.jpg)

After the initial ArgoCD installation Argo manages itself.

# ArgoCD

Why ArgoCD for such a small setup?

![Meme - Strong dog: "computers then: I sent people to the moon with just 4kb ram. Crying normal dog: "computers now: help me please 4Gb ram is not enough for me.](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcR9es8OrAdRShum4eHs9WViIt72A05MZvQ30g&usqp=CAU)

On the previous setup I had to expose the ssh service on the internet to play with the cluster outside the lan. I also had to run Ansible every time I modified the services. Because my RPi was slow it really boring to wait several minutes for simple modifications. Now with GitOps on ArgoCD I don't need direct access to the nodes to configure the services.

There was a risk that I would exhaust the cluster resources with only the maintenance services, but there are still around 60% free resources after adding:

- ArgoCD
- Prometheus
- Grafana
- External DNS
- Cert-Manager
- Kong

## ArgoCD project and application bootstrapping

To bootstrap the argo configuration I used the [app of apps](https://argoproj.github.io/argo-cd/operator-manual/cluster-bootstrapping/) pattern. This is Ansible's responsibility: it applies to k3s [3 initial configurations](https://github.com/thspinto/rpi-cluster/tree/main/argocd/bootstrap). Service applications and infrastructure applications are separated to give less permissions to non infrastructure service configuration. The other configuration is to bootstrap argo projects to create new ones directly from argo in case we need it.

## Secrets

My first approach to keep secrets versioned was using [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets). Since I don't care about backups yet and rebuild the cluster frequently for testing, I had to recreate the sealed secrets every time I destroyed the cluster because I lost the encryption keys. I found it a lot easier to just keep an encrypted `secrets.yaml` file in the repository. In this workflow, every time I spin up a new cluster, I just have o decrypt the file and apply it to the new cluster.
Although it is encrypted I don't really recommend keeping the file in a public github repository. I'm doing this to not loose the secrets util I workout a better flow for personal projects.
