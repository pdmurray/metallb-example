# Example to set up MetalLB on local cluster with Nginx demo

## Intro

In my homelab, I have been using TrueNAS running on a VM in ESXi with my pool disks attached through a passthrough PCI-Express HBA.

I also run an Ubuntu VM with Docker hosting several applications, including Pi-Hole.  I have configured my router to use Pi-Hole as the DNS server for all devices connected to my LAN environment.

I recently added another VM to experiment with Kubernetes in this environment.  This guide explains the steps I took to configure it to:
- Use MetalLB as a LoadBalancer that assigns pods a real IP in my LAN
- Use ExternalDNS to assigns a DNS name registered in Pi-Hole in my local domain for any K8S Service, 
- Leverage TrueNAS SCALE for Persistent Volumes over my ZFS pool.

## Guide

### Pre-requisites

- TrueNAS SCALE with Pool created
- Kubernetes cluster (I use K3S running on Ubuntu)
- Docker host with macvlan network configured:

``` 
  # Reserving a single address on the LAN:
  
  docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --ip-range=192.168.1.2/32 \
  --gateway=192.168.1.1 \
  -o parent=ens160 macnet32
```

### Install Pi-Hole

`$ cd pihole-docker`

`$ docker-compose up`

### Install metalLB

Prereqs: 
- Configure network in your router, e.g. 192.168.1.231-192.168.1.250 
- Configure same network in values.yaml

Install:

`$ helm upgrade --install --create-namespace --values values.yaml --namespace=metallb-system metallb metallb/metallb`

### Install External-DNS

References:
- https://github.com/tinyzimmer/external-dns/blob/pihole-support/docs/tutorials/pihole.md#externaldns-manifest

Notes:

Using custom-built docker image `gcr.io/harness-test-338118/external-dns:latest` from fork: https://github.com/tinyzimmer/external-dns.git

Prereqs:
- configure `--pihole-server=` in template args section
- create namespace: `$ kubectl create namespace externaldns`
- create password secret: `$ kubectl -n externaldns create secret generic pihole-password     --from-literal EXTERNAL_DNS_PIHOLE_PASSWORD=secretpassword`

Install: 

`$ kubectl apply -f externaldns.yml`

### Install democratic-csi

References: 
- https://jonathangazeley.com/2021/01/05/using-truenas-to-provide-persistent-storage-for-kubernetes/
- https://github.com/democratic-csi/democratic-csi
- https://github.com/democratic-csi/democratic-csi/issues/101

Prereqs:
- Truenas: enable nfs w/ nfsv4 support, create datasets, create API key
- configure host, paths, api key in template

Install:

`$ helm upgrade --install --create-namespace --values freenas-nfs.yaml --namespace democratic-csi zfs-nfs democratic-csi/democratic-csi`

Test: 

`$ kubectl apply -f test-claim-nfs.yml`

### Test MetalLB, democratic-csi & External-DNS (Pihole) with Nginx

With the following example, nginx should request an external IP with MetalLB, register DNS with pihole, and mount the html directory to your TrueNAS ZFS PV.

`$ kubectl apply -f nginx.yaml`

`$ kubectl cp index.html my-nginx-67f95948d-k8pp2:/usr/share/nginx/html -n nginx`