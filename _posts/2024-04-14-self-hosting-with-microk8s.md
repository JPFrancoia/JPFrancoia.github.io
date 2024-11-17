---
layout: posts
title:  "Self-hosting with MicroK8s"
image: microk8s.png

excerpt: "A few insights about how I host my own services with MicroK8s"

---

# Self-hosting with MicroK8s

I have been self-hosting various services for a while now, and I thought it
would be useful to share some insights about how I do it. I reached my current
setup over the course of a few years, so it's possible that some details
slipped my mind.

## Why self-host?

I don't want to rant too much about freedom, privacy, and all that, there
are plenty of resources out there that do a great job at explaining why
self-hosting helps on these particular topics. I have more practical reasons
for doing this: 1) I like doing this kind of stuff. It brings me joy to set
up and maintain my own infrastructure. 2) I learn a lot by doing this. I
am convinced that tinkering with my own servers at home (and software
in general) has made me a better software engineer. 3) It's cheaper. I
host many services and if I were to host all this in the cloud, my monthly
bill would probably be in the hundreds. It's obviously a trade-off: I lose
the disaster recovery capacity of the cloud, the uptime, the compliance,
the elasticity, etc. But I don't need all that for the services I host. 4)
I'm not impacted if an online service closes down. E.g: I was a heavy user
of Google Reader a few years ago until Google decided to shut it down.

## What do I host?

- [Tiny Tiny RSS](https://tt-rss.org/): I use it to read RSS feeds. This
is a very critical software to me, that's how I keep up with the news. This
is what replaced Google Reader for me.
- [Ghostfolio](https://ghostfol.io/en/start): I use it to track my
  investments and my pension (I donated to the project as it's a critical tool for me).
- [Home Assistant](https://www.home-assistant.io/): I use it to automate
  various things in my home. I have a few smart plugs, a smart thermostat,
  smart light bulbs, smart valves, and a few other things. I also use it
  to monitor the temperature and humidity in my home.
- [Zulip](https://zulip.com/): I use it to communicate with friends when we
    work on a project together. It's basically the equivalent of Slack, but
    cheaper and open-source. I'm in control of all the data (messages and
    files). The same features with Slack would cost me around £7 per user per
    month (that's a minimum of £14 per month to talk to one friend).
- Various home-made services: backends and frontends for various projects
  I'm working on.

The list is not exhaustive, but it gives you an idea of what I host. The
services above also imply deploying "supporting services" like databases,
caches, MQTT brokers, etc.


## The hardware

My Kubernetes cluster is composed of 2 nodes:
- My home-made NAS (well, it's not just a NAS anymore...). See [this
post](https://jpfrancoia.github.io/2021/01/17/home-server.html)
- A second hand Dell Optiplex 3050 Micro. It's small, silent, doesn't consume a
lot of power, and it's surprisingly powerful for the price. They can be found
on eBay for ~£100. My model packs an i5 CPU, 16GB of RAM, and a 512GB SSD

## The setup

Both the NAS and the Dell run Archlinux. I installed microk8s on both of them
(via snap). The NAS hosts the control plane (and is also a worker node). The
Dell is simply a worker node.

<p align="center">
  <img width="700" src="{{ site.baseurl }}/images/microk8s/components-of-kubernetes.svg">
</p>

Building this cluster was surprisingly easy. After installing microk8s, I just
needed to run a few commands on the nodes. The full documentation is
[here](https://microk8s.io/docs/clustering), but in a nutshell:

On the NAS (master node):

```bash
microk8s add-node
```

This command outputs some instructions that then need to be run on the **worker
nodes**.

This was fast, the setup took less than 10 minutes and I had a cluster running
two nodes:

```bash
microk8s kubectl get no

NAME            STATUS   ROLES    AGE    VERSION
djipey-server   Ready    <none>   486d   v1.28.9
djipey-minipc   Ready    <none>   240d   v1.28.9
```

In this configuration my cluster was easy to setup, but it's also not
*highly available*. If the NAS goes down, the cluster goes down too, as
the NAS hosts the control plane. I can live with that though.


## Deploying services

To deploy services, I use [Kustomize](https://kustomize.io/). The idea
behind this tool is to separate the base configuration of a service from its
environment-specific configuration. This is done by creating a `base` folder
and an `overlays` folder. The `base` folder contains the base configuration of
the service. The `overlays` folder contains the environment specific tweaks.

This is an example of the folder structure for the deployment of one of my
backends:

```bash
├── backend
│   ├── base
│   │   ├── api-deploy.yaml
│   │   ├── api-svc.yaml
│   │   ├── application.env
│   │   ├── db-deploy.yaml
│   │   ├── db-svc.yaml
│   │   ├── defaultbackend-deploy.yaml
│   │   ├── defaultbackend-svc.yaml
│   │   ├── kustomization.yaml
│   │   └── secrets.env
│   └── overlays
│       ├── dev
│       │   ├── application.env
│       │   ├── db-deploy-patch.yaml
│       │   ├── db-svc-patch.yaml
│       │   ├── kustomization.yaml
│       │   └── secrets.env
│       └── prod
│           ├── application.env
│           ├── db-svc-patch.yaml
│           ├── kustomization.yaml
│           └── secrets.env
```

Without getting into too much details, the `base` folder defines the basic
components of my application: database, api, service, etc. The `overlays`
then define the environment-specific tweaks. For example, the `dev` overlay
might set the number of replicas to 1 for the api component, while the `prod`
overlay might set the number of replicas to 3. In general, the subfolders
of the `overlays` folder are named after different namespaces. i.e:
the `dev` overlay will deploy the service in the `dev` namespace (the
`kustomization.yaml` file in the `dev` folder will set the namespace to
`dev`).

When I do not need the concept of environments for a service, I use a different
structure. For example, for my home assistant deployment, I use the following:

```bash
├── base
│   ├── ha-deploy.yaml
│   ├── ha-svc.yaml
│   ├── kustomization.yaml
│   ├── mqtt-deploy.yaml
│   └── mqtt-svc.yaml
└── overlays
    └── homeassistant
        └── kustomization.yaml
```


### The dirty truth about a self-hosted setup

For a self-hosted setup, you need to be prepared to deal with a few things:
- volumes, i.e: where you store your persistent data
- additional "stuff" plugged into a specific machine

These factors might force you to run a service on a specific node, which can
quickly render the multi-nodes setup useless. If critical services absolutely
need to be deployed on a specific node, they can become a bottleneck if
this node doesn't have sufficient resources.

In my case for example, I run several database services. These services
require persistent storage. Since this data is somewhat important, I want
to make sure it lives on the NAS, since the NAS has a RAID setup. This means
that I need to run my database services on the NAS. All my database pods are
then deployed on the master node (the NAS). This is not ideal, but I haven't
noticed any performance issues so far. Some solutions to this problem exist,
like [Ceph](https://ceph.io), but I haven't had the time to set it up yet.

I also have a [Con Bee II](https://phoscon.de/en/conbee2) Zigbee USB stick
plugged into the NAS. I use it to control my Zigbee devices through Home
assistant. Since this stick is only plugged into the NAS, I need to make sure
that the Home Assistant pod is deployed on the NAS. This is easily accomplished
with a `nodeName` field in the pod spec:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: homeassistant
  name: homeassistant
spec:
  replicas: 1
  selector:
    matchLabels:
      app: homeassistant
  template:
    metadata:
      labels:
        app: homeassistant
    spec:
      nodeName: djipey-server
      restartPolicy: Always
      volumes:
      - name: homeassistant-volume
        hostPath:
          path: /home/djipey/home_assistant
      - name: ttyacm
        hostPath:
          path: /dev/serial/by-id/usb-dresden_elektronik_ingenieurtechnik_GmbH_ConBee_II_DE2462071-if00
      containers:
      - name: homeassistant
        image: ghcr.io/home-assistant/home-assistant:stable
        securityContext:
          privileged: true
        imagePullPolicy: Always
        ports:
        - containerPort: 8123
        volumeMounts:
        - mountPath: /config
          name: homeassistant-volume
        - mountPath: /dev/serial/by-id/usb-dresden_elektronik_ingenieurtechnik_GmbH_ConBee_II_DE2462071-if00
          name: ttyacm
        env:
        - name: TZ
          value: Europe/London
```


## Docker registry

With microk8s, it's easy to deploy services from public Docker images, you
just need to fill the `image` field in the deployment's specs. But what if
you want to build and deploy your own images?

The quickest way to do it, if you can ssh into a node:

```bash
docker build -t my_image .
docker tag my_image ${NAME}:latest
docker save my_image > myimage.tar
microk8s ctr image import myimage.tar
rm myimage.tar
```

But this is really painful:
- you need to ssh into a node
- you need to build the image on the nodes
- you need to save the image to a tarball
- you need to import the image with `ctr`
- you need to clean up

There is a better way. Microk8s comes with a [built-in registry](https://microk8s.io/docs/registry-built-in).
Once again, microk8s' documentation made it very easy to enable the registry,
the steps were straightforward. After a few commands I had a Docker registry
running on the master node, on port 32000.

### Security of the Docker registry

The documentation mentions this:

> Note that this is an insecure registry and you may need to take extra
steps to limit access to it.

It's indeed an insecure registry. There is no form of authentication, and
anyone who can access the node can push and pull images from the registry.
However, I haven't exposed the registry to the internet, so any potential
attacker would need to be on my local network to access the node. This isn't
very high risk, and for good measure, I configured the firewall of the NAS to
only allow connections on port 32000 from a couple of (reserved) IP addresses.


## Routing Internet traffic

I need to expose some services to the internet. For example, I want to access
Tiny Tiny RSS from my phone when I'm not at home (same for Zulip). To achieve
this, I follow this process:

- I reserve a domain name (I use AWS Route 53 for this)
- I create an A record that points to my home's public IP address
- I configure my router to forward the traffic on the ports I need to the NAS
    (it's basically just the https port, 443)
- I configure the ingress controller (that's a Kubernetes service) to route
the traffic to the correct services


There is no need to reserve several domain names for a home setup. I bought
 one, and I use a mixture of url paths and subdomains to route the traffic
 to the correct service. The route 53 setup looks like this (Terraform code):


```hcl
resource "aws_route53_zone" "base_domain_name" {
  name = var.base_domain_name
}

resource "aws_route53domains_registered_domain" "domain" {
  domain_name = var.base_domain_name
  auto_renew  = false

  # Choose Enable/True (to lock the domain) or Disable/False (to unlock the domain).
  transfer_lock = true

  dynamic "name_server" {
    for_each = aws_route53_zone.base_domain_name.name_servers

    content {
      name = name_server.value
    }
  }
}

resource "aws_route53_record" "www" {
  allow_overwrite = true
  zone_id         = aws_route53_zone.base_domain_name.zone_id
  name            = var.base_domain_name  # E.g. example.com
  type            = "A"
  ttl             = 300

  records = [var.k8s_cluster_ip]
}

resource "aws_route53_record" "zulip_www" {
  allow_overwrite = true
  zone_id         = aws_route53_zone.base_domain_name.zone_id
  name            = var.zulip_domain_name  # E.g. zulip.example.com
  type            = "A"
  ttl             = 300

  records = [var.k8s_cluster_ip]
}
```

And the ingress configuration for Zulip looks like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: 'true'
    cert-manager.io/cluster-issuer: letsencrypt-cluster-issuer
spec:
  ingressClassName: nginx
  rules:
  - host: zulip.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: zulip
            port:
              number: 80
  tls:
  - hosts:
    - zulip.example.com
    secretName: letsencrypt-cluster-issuer
```

In the case of Zulip, I use a subdomain to route the traffic to the
service. If you look closely at the ingress configuration, you'll see that
the ingress controller routes the traffic to the `zulip` service on port 80.
The ingress controller is responsible for SSL termination, and it uses
a certificate managed by a cert-manager cluster issuer.


### Managing certificates with cert-manager

To enable https traffic between the clients and the
services when the services are exposed to the internet, I use
[cert-manager](https://cert-manager.io). They have an excellent tutorial
[here](https://cert-manager.io/docs/tutorials/acme/nginx-ingress/). Deploying
Cert-manager creates a few Kubernetes resources (certificates, secrets,
issuers, etc). Cert-manager then takes care of renewing the certificates
when they are about to expire. Cert-manager is so easy to use that I don't
need to think about it anymore.

There is one catch to using cert-manager for a home setup though: you'll
likely buy one unique domain name, but you'll want to create ingress
resources in different namespaces. This isn't possible with the `Issuer`
resource recommended in their tutorial. I ended up creating one unique
`ClusterIssuer`, which handles the certificate for my unique domain name. I
can then create ingress resources in any namespace I want.

