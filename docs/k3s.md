---
layout: post
title:  "Raspberry Pi k3s cluster"
date:   2021-02-07 16:14:21 +0300
categories: k3s
---

# Setting up a private Docker registry used by a Raspberry Pi k3s cluster

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {:no_toc .text-delta }
1. TOC
{:toc}
</details>


We are going to setup a private Docker registry and generate a self-signed certificate that will be used to authenticate to the registry.

## Certificate for accessing the registry using TLS

We will create a folder in the `/home/pi/` directory of our Raspberry Pi and will generate our keys and certificates.
This is just a testing environment and a prototype.

```bash
mkdir -p certs && \
openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout certs/pi_reg_domain.key \
  -x509 -days 3650 -out certs/pi_reg_domain.crt \
  -subj "/C=FI/ST=Espoo/L=Espoo/O=Example_Company/OU=Example_Department/CN=pi-reg" \
  -addext "subjectAltName = DNS:pi-reg" && \
cp certs/pi_reg_domain.crt certs/pi_reg_ca.crt
```

> **NOTE**: Added `subjectAltName` field when creating the certificate to fix the `k3s/containerd` error when trying to pull the images from the registry.
Apparently, the certificate validation fails in `golang` if `subjectAltName` is not provided. 
Although in the given post we will set `k3s` to use Docker instead of `containerd` as an OCI runtime, in case you would like to do otherwise, this is a handy note.

Eventually, they will be used by Docker for registry authentication when operating with the registry.
```bash
sudo mkdir -p /etc/docker/certs.d/pi-reg:444 && \
sudo cp certs/pi_reg_domain.crt /etc/docker/certs.d/pi-reg:444/client.cert && \
sudo cp certs/pi_reg_domain.crt /etc/docker/certs.d/pi-reg:444/ca.crt && \
sudo cp certs/pi_reg_domain.key /etc/docker/certs.d/pi-reg:444/client.key
```

Because we are having a cluster consisting of several Pis, we will have to copy the authentication certificate and the key to the agent nodes.
```bash
IP_WORKING_NODE=<ip-of-the-working-node> && \
IP_WORKING_NODE=192.168.0.152 && \
ssh -t pi@$IP_WORKING_NODE 'sh -c "mkdir -p /home/pi/reg_keys"' && \
scp certs/pi_reg_domain.crt pi@$IP_WORKING_NODE:~/reg_keys/client.cert && \
scp certs/pi_reg_domain.crt pi@$IP_WORKING_NODE:~/reg_keys/ca.crt && \
scp certs/pi_reg_domain.key pi@$IP_WORKING_NODE:~/reg_keys/client.key && \
ssh -t pi@$IP_WORKING_NODE \
'sudo sh -c "mkdir -p /etc/docker/certs.d/pi-reg:444 && mv /home/pi/reg_keys/* /etc/docker/certs.d/pi-reg:444/"' && \
ssh -t pi@$IP_WORKING_NODE 'sudo sh -c "mkdir -p /usr/local/share/ca-certificates && \
    cp /etc/docker/certs.d/pi-reg:444/ca.crt /usr/local/share/ca-certificates/pi_reg_ca.crt && \
    update-ca-certificates"' && \
ssh -t pi@$IP_WORKING_NODE 'sudo sh -c "rm -r /home/pi/reg_keys && echo 192.168.0.179 pi-reg >> /etc/hosts"'
ssh -t pi@$IP_WORKING_NODE 'sudo sh -c "rm -r /home/pi/reg_keys && echo <node-ip-address> pi-reg >> /etc/hosts"'
```

Reload the Docker daemon to refresh the latest configurations and restart the Docker service.
```bash
sudo systemctl daemon-reload && \
sudo systemctl restart docker.service
```

Perform the same operation (daemon reloading and restarting Docker service) on all the agent nodes.

Since on the local development environment there is no DNS configured, edit the `/etc/hosts` file and add an entry for `pi-reg` domain to point to the host machine of the private docker registry (`127.0.0.1` - in the case the registry runs on current machine, `<ip-of-the host>` - otherwise).

## Private registry setup

Create a local directory for persisting the custom images:
```bash
sudo mkdir -p /var/local/docker/registry
```

Create the file `registry_config.yaml` with the following reference content:
```yaml
version: 0.1
log:
  fields:
    service: registry
  level: info
storage:
  cache:
    blobdescriptor: ""
  filesystem:
    rootdirectory: /var/lib/registry
    maxthreads: 100
  delete:
    enabled: true
http:
  addr: 5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

Deploy a local registry within a docker container:
```bash
sudo docker run -d \
  --restart=always \
  --name registry \
  -v `pwd`/certs:/certs \
  -v `pwd`/registry_config.yaml:/etc/docker/registry/config.yaml \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/pi_reg_domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/pi_reg_domain.key \
  -v /var/local/docker/registry:/var/lib/registry \
  -p 444:443 \
  registry:2
```

We will export our registry on the port `444`, because the default load balancer will use the `443` port for exporting the `https` endpoints.

Test whether the registry is accessible and the we can push/pull images to it.
Considering that the image `flask-rest` already exists in our docker registry:
```bash
# Verify the catalog of our private registry and assert that it is empty
curl --cacert ~/certs/pi_reg_domain.crt https://pi-reg:444/v2/_catalog
docker tag flask-rest pi-reg:444/flask-rest
docker push pi-reg:444/flask-rest
# Verify the catalog of our private registry and assert the newly pushed image
curl --cacert ~/certs/pi_reg_domain.crt https://pi-reg:444/v2/_catalog
```

Test that docker is able to pull and run images from the private repository
```bash
# Delete the image from your docker registry
docker rmi flask-rest && \
# The might be the tagged image still present in the docker registry
docker rmi pi-reg:444/flask-rest && \
# List all the available images and assert `flask-rest` is not in the list
docker images

# Spawn a container from the `flask-rest` that is pulled from the private repository
docker run -p 5000:5000 pi-reg:444/flask-rest
```

If docker is not able to access the private registry because of the proxy, then edit `/etc/systemd/system/docker.service.d/proxy.conf` file and add the domain name of our registry to `NO_PROXY` section:
```bash
NO_PROXY=localhost,pi-reg,127.0.0.1,::1
```

This may happen if you want to deploy the registry within an enterprise LAN behind a proxy. The entire `[Service]` section of the `/etc/systemd/system/docker.service.d/proxy.conf` file should look something like:
```bash
[Service]
Environment="HTTP_PROXY=<your-corporate-proxy-url>"
Environment="HTTPS_PROXY=<your-corporate-proxy-url>"
Environment="NO_PROXY=localhost,pi-reg,127.0.0.1,::1"
```

Additionally, edit the `/etc/environment` and add `pi-reg` to `no_proxy` section, too.

Debugging commands:
```bash
$ sudo journalctl -fu docker.service
$ sudo nano /etc/systemd/system/docker.service.d/proxy.conf
$ sudo systemctl show --property=Environment docker
```

## Private Registry. K3S usage

We need `k3s` OCI runtime to be able to access our private registries. For this, we have to edit `/etc/rancher/k3s/registries.yaml` on **every agent and master node**.

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://pi-reg:444"
configs:
  "pi-reg:444":
    tls:
      cert_file: /etc/docker/certs.d/pi-reg:444/client.cert
      key_file: /etc/docker/certs.d/pi-reg:444/client.key
      ca_file: /etc/docker/certs.d/pi-reg:444/ca.crt
```

Register locally the self-signed certificate authority:
```bash
sudo mkdir -p /usr/local/share/ca-certificates && \
    sudo cp /etc/docker/certs.d/pi-reg:444/ca.crt /usr/local/share/ca-certificates/pi_reg_ca.crt && \
    sudo update-ca-certificates
```

(Deprecated) We will disable the default `containerd` OCI runtime that is used by `k3s` and will use `docker` instead.
For this, append the string `--docker \` at the end of the:
- `/etc/systemd/system/k3s.service` - for the master node;
- `/etc/systemd/system/k3s-agent.service` - for the agent nodes.
See [k3s documentation](https://rancher.com/docs/k3s/latest/en/installation/install-options/server-config/) for more information.

And install docker on Raspberry Pi (3):
```bash
mkdir docker_installation && \
    cd docker_installation && \
    curl -fsSL https://get.docker.com -o get-docker.sh && \
    sudo sh get-docker.sh
```

Restart `docker` and `k3s` on the master node:
```bash
sudo systemctl daemon-reload && \
sudo systemctl restart docker && \
sudo systemctl restart k3s.service
```

Restart `docker` and `k3s` on the agent node:
```bash
sudo systemctl daemon-reload && \
sudo systemctl restart docker && \
sudo systemctl restart k3s-agent.service
```

Test that `docker` can pull the image from the private docker repository:
```bash
sudo docker pull pi-reg:444/med/flask-rest
```

> NOTE: If the certificate is creating without passing the `subjectAltName`, then `k3s`'s `containerd` will complain with:

```bash
FATA[2021-02-01T22:03:25.974609779+02:00] pulling image: rpc error: code = Unknown desc = failed to pull and unpack image "openmpi-reg:444/med/flask-rest:latest": failed to resolve reference "pi-reg:444/med/flask-rest:latest": failed to do request: Head "https://pi-reg:444/v2/flask-rest/manifests/latest": x509: certificate relies on legacy Common Name field, use SANs or temporarily enable Common Name matching with GODEBUG=x509ignoreCN=0
```

>NOTE: When `containerd` is used, a success pull message should be displayed:

```bash
$ sudo k3s crictl pull pi-reg:444/flask-rest
Image is up to date for sha256:f7527e63e7585640724c3961c684bd0e83ad5dbed37e5e1c6d4d6a3bea5c9964
```

If the image is pulled successfully, you can start deploying k3s services and use `pi-reg:444/flask-rest` image from the private docker registry.

(Optional) Adding the current user to the docker group:
```bash 
sudo usermod -aG docker pi
```
