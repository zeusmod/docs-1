---
title: Air-Gapped Deployment
description: Learn how to set up an air-gapped Coder deployment.
---

If you need increased security for your Coder deployments, you can set up an
air-gapped deployment.

To do so, you must:

- Pull all Coder deployment resources into your air-gapped environment
- Push the images to your Docker registry,
- Deploy Coder from within your air-gapped environment

> Coder licenses issued as part of the trial program do not support air-gapped
> deployments.

## Dependencies

Before proceeding, please ensure that you've installed the following software
dependencies:

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [helm](https://helm.sh/docs/intro/install/)

In the same network as the Kubernetes cluster that will run Coder, additional
services need to be configured. Links go to suggestions but many other options
can be used.

- [Docker registry](https://hub.docker.com/_/registry)
- [DNS server](https://coredns.io) or
  [HostAliases](https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/)
  patched in
- [Certificate authority](https://github.com/activecm/docker-ca/blob/master/Dockerfile)
  or [self-signed certificates](#self-signed-certificate-for-the-registry)

## Step 1: Pull all Coder resources into your air-gapped environment

Coder is deployed through [helm](https://helm.sh/docs/intro/install/), and the
platform images are hosted in Coder's Docker Hub repo.

1. Pull down the Coder helm charts by running the following in a non-air-gapped
   environment:

   ```console
   helm repo add coder https://helm.coder.com
   helm pull coder/coder
   ```

   These commands will add Coder's helm charts and pull the latest stable
   release into a tarball file whose name uses the following format:
   coder-X.Y.Z.tgz (X.Y.Z is the Coder release number).

1. Pull the images for the Coder platform from the following Docker Hub
   locations:

   [coder-service](https://hub.docker.com/r/coderenvs/coder-service)

   [envbox](https://hub.docker.com/r/coderenvs/envbox)

   [envbuilder](https://hub.docker.com/r/coderenvs/envbuilder)

   [timescale](https://hub.docker.com/r/coderenvs/timescale)

   [dashboard](https://hub.docker.com/r/coderenvs/dashboard)

   You can pull each of these images from their `coderenvs/<img-name>:<version>`
   registry location using the image's name and Coder version:

   ```console
   docker pull coderenvs/coder-service:<version>
   ```

   Additional images may be needed to configure and run workspaces:

   [nginx-ingress-controller](https://quay.io/kubernetes-ingress-controller/nginx-ingress-controller)

   [enterprise-node](https://hub.docker.com/r/codercom/enterprise-node)

   [enterprise-intellij](https://hub.docker.com/r/codercom/enterprise-intellij)

   [ubuntu](https://hub.docker.com/_/ubuntu) as a base image

1. Tag and push all of the images that you've downloaded in the previous step to
   your internal registry; this registry must be accessible from your air-gapped
   environment. For example, to push `coder-service`:

   ```console
   docker tag coderenvs/coder-service:<version> my-registry.com/coderenvs/coder-service:<version>
   docker push my-registry.com/coderenvs/coder-service:<version>
   ```

1. Once all of the resources are in your air-gapped network, run the following
   to deploy Coder to your Kubernetes cluster:

   ```console
   kubectl create namespace coder
   helm --namespace coder install coder /path/to/coder-X.Y.Z.tgz \
   --set cemanager.image=my-registry.com/coderenvs/coder-service:<version> \
   --set envproxy.image=my-registry.com/coderenvs/coder-service:<version> \
   --set envbuilder.image=my-registry.com/coderenvs/envbuilder:<version> \
   --set timescale.image=my-registry.com/coderenvs/timescale:<version> \
   --set dashboard.image=my-registry.com/coderenvs/dashboard:<version> \
   --set envbox.image=my-registry.com/coderenvs/envbox:<version>
   ```

1. Next, follow the [Installation](installation.md) guide beginning with **step
   6** to get the access URL and the temporary admin password, which allows you
   to proceed with setting up and configuring Coder.

## Extensions marketplace

You can configure your deployment to use the internal, built-in extension
marketplace, allowing your developers to utilize whitelisted IDE extensions
within your air-gapped environment. For additional details, see
[Extensions](../admin/environment-management/extensions.md).

## Related infrastructure

If the network already has a certificate authority, domain name service, and
docker registry, the content below is unnecessary. The code snippets are subject
to be outdated since they are all other software packages that Coder does not
control or distribute.

### Self-signed certificate for the registry

This can be used in any environment but it's required for air-gapped
installations.

When creating the registry, create a certificate using a command like:

```bash
export REGISTRY_DOMAINNAME=registry.local
mkdir /certs
openssl req \
    -newkey rsa:4096 -nodes -sha256 -keyout /certs/registry.key \
    -x509 -days 365 -out /certs/registry.crt
```

When it asks for the "Common Name [CN]: " this value must match what you set in
DNS. For the volume mounted at `/var/lib/registry` make sure it can store 10+ GB
for just Coder images.

```bash
docker run -d -p 443:5000 \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
    -v /certs:/certs \
    -v /var/lib/docker/registry:/var/lib/registry \
    registry:2
```

Get the images such as the nginx-ingress-controller and coderenv/\* listed
above.

```bash
docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1
docker tag quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1 $REGISTRY_DOMAIN_NAME/nginx-ingress-controller:0.26.1
docker push $REGISTRY_DOMAIN_NAME/nginx-ingress-controller:0.26.1
```

### Node configurations

For the Kubernetes node to accept the certificate, and different software
packages need to be told to trust that registry.crt file. Put that certificate
in various spots depending on linux distribution and container runtime.

```plaintext
/usr/local/share/ca-certificates/registry.crt
/etc/docker/certs.d/${REGISTRY_DOMAIN_NAME}/ca.crt
/etc/ssl/certs/registry.crt
/etc/pki/tls/registry.crt
```

Containerd resisted the system certificate update so the below command patches
in an untrusted certificate for images coming from the local registry domain.

```bash
update-ca-certificates
cat <<EOT >> /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".registry.configs."$REGISTRY_DOMAIN_NAME".tls]
  insecure_skip_verify = true
EOT
systemctl restart containerd
```

Since each node needs to have this same thing run on it, it needs to be baked
into the image or run as an init script whenever adding a new node to the
cluster. This isn't necessary on master nodes, just those that will be
scheduling the Coder images.

### Helm chart certs secret values

Coder backend validates images and pulls tags using API calls so the system
needs to so a certificate issue will prevent adding images. If a certificate
authority is present in the network, the root certificate may need to be added
this way.

For a self-signed certificate to be passed into the Coder images a secret needs
to be created and then referenced in the chart. The secret is a created from a
directory with a single file in it. The directory name is irrelevant and the
filename becomes the `key` if the following command is used.

```bash
kubectl -n coder create secret generic local-registry-cert --from-file=/certs
```

If the `-out` argument on the OpenSSL command to generate the certificates was
changed or the certificate was moved, adjust the `--from-file=` argument.

To check the secret to make sure it looks right, use this command:

```bash
kubectl -n coder get secret local-registry-cert -o yaml
```

This snippet can be added to the helm command by putting it in a yaml file like
`registry-cert-values.yml` and then adding `-f registry-cert-values.yml` to the
end of the command above.

```yaml
certs:
  secret:
    name: "local-registry-cert"
    key: "registry.crt"
```

### Resolve registry with cluster DNS or hostAliases

Without a customer domain name server, the nodes need to have their host file
set to have the `$REGISTRY_DOMAIN` and static IP address of the local registry.

If the registry is on 10.0.0.2, something like this should be added to the node
configuration script started a few sections back.

```bash
echo "10.0.0.2  $REGISTRY_DOMAIN_NAME" >> /etc/hosts
```

This may not help the containers within the cluster since some Kubernetes DNS
services forward directly out of the cluster.

If the hosts file on the node isn't being heeded by pods, a work-around is to
extract the helm chart from `coder-X.Y.Z.tgz` (retrieved above) and patch the
ce-manager deployment. It goes at the same indent level as `containers:`.

```yaml
hostAliases:
  - hostnames:
      - $REGISTRY_DOMAIN_NAME
    ip: 10.0.0.2
```

### Ingress image adjustment

When making any change to the helm chart, the best approach is to extract the
tgz that comes from the `helm pull` command. The extracted files can be modified
and saved to a git repository for future update differentials.

Once inside the extracted coder helm chart directory, you will find the
`templates/ingress.yaml` file only has one instance of `image:`. Replace the
image reference with the local `registry/name:tag` format.

### Installing from an extracted helm chart

To install from an extracted helm chart, you simply replace the `repo/chart` or
`coder.tgz` path with a period. Run the command below from inside the directory.

```bash
helm install --wait --atomic --debug --namespace coder coder . \
   --set cemanager.image=$REGISTRY_DOMAIN_NAME/coderenvs/coder-service:1.17.2 \
   --set envproxy.image=$REGISTRY_DOMAIN_NAME/coderenvs/coder-service:1.17.2 \
   --set envbox.image=$REGISTRY_DOMAIN_NAME/coderenvs/envbox:1.17.2 \
   --set envbuilder.image=$REGISTRY_DOMAIN_NAME/coderenvs/envbuilder:1.17.2 \
   --set timescale.image=$REGISTRY_DOMAIN_NAME/coderenvs/timescale:1.17.2 \
   --set dashboard.image=$REGISTRY_DOMAIN_NAME/coderenvs/dashboard:1.17.2 \
   -f registry-cert-values.yml
```
