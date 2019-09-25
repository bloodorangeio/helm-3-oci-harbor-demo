# Helm 3 + OCI + Harbor Demo

For more info on Helm 3 and registries, please see [the docs](https://v3.helm.sh/docs/topics/registries/).

## Installing Helm 3

The latest beta release of Helm 3 is [v3.0.0-beta.3](https://github.com/helm/helm/releases/tag/v3.0.0-beta.3) (as of 2019-09-24).

Example of installing Helm 3 as `h3` on a Mac:

```
# Install
wget -O h3.tar.gz https://get.helm.sh/helm-v3.0.0-beta.3-darwin-amd64.tar.gz
rm -rf h3/ && mkdir -p h3/
tar xf h3.tar.gz -C h3/ --strip-components 1 
chmod +x h3/helm
mv h3/helm /usr/local/bin/h3
rm -rf h3.tar.gz h3/

# Check version
h3 version
```


## Deploying Harbor with Helm OCI support

*Note: the example below requires a cluster with [cert-manager](https://hub.helm.sh/charts/jetstack/cert-manager) and [nginx-ingress-controller](https://hub.helm.sh/charts/bitnami/nginx-ingress-controller) installed.*


First, add the official Harbor helm repo:

```
h3 repo add harbor https://helm.goharbor.io
```

Next, create the file `helm-oci-support.yaml` with the following contents:
```yaml
core:
  image:
    repository: bloodorangeio/harbor-core
    tag: master
notary:
  enabled: false
externalURL: https://harbor.mysite.io
expose:
  ingress:
    hosts:
      core: harbor.mysite.io
    annotations:
      kubernetes.io/ingress.class: nginx
      kubernetes.io/tls-acme: "true"
      ingress.kubernetes.io/proxy-body-size: "1m"
```

Be sure to replace any instances of `harbor.mysite.io` above with your own domain, which is pointed to the nginx-ingress-controller service load balancer.

(If you're interested in the contents of the `bloodorangeio/harbor-core` image, you can see the [diff](https://github.com/goharbor/harbor/compare/master...bloodorangeio:master).)


Finally, install Harbor chart using Helm 3:

```
h3 upgrade -i harbor harbor/harbor -f ./helm-oci-support.yaml
```

and wait until all Harbor pods have started and the Let's Encrypt cert is ready:
```
watch -n 5 'kubectl get pods && kubectl get certs | grep harbor'
```

Once everything is ready, you are strongly encouraged to change your admin password to something other than "Harbor12345" via the UI.

## Pushing charts to Harbor over OCI

First and foremost, set the required experimental environment variable to enabled OCI support in Helm:
```
export HELM_EXPERIMENTAL_OCI=1
```

Then, login to your Harbor instance using Docker-style auth:

```
h3 registry login harbor.mysite.io -u admin -p Harbor12345
```

Next, download a chart to use (for example, stable/wordpress):

```
h3 repo add stable https://kubernetes-charts.storage.googleapis.com
h3 fetch stable/wordpress --untar
```

Save the chart it to the local OCI cache:

```
h3 chart save wordpress/ harbor.mysite.io/library/wordpress:7.3.7
```

and list all locally-cached charts:
```
h3 chart list
```

Finally, push the chart to Harbor registry remote:
```
h3 chart push harbor.mysite.io/library/wordpress:7.3.7
```

You may also choose to use `--debug` on push to view extra HTTP request info.

## Limitations

Helm charts over OCI have not been fully integrated into the Harbor user experience just yet. For example, you should be able to view your charts in the UI, but you may see a lot of empty fields everywhere.

However, Harbor can still be used as a centralized chart registry to be leveraged across all Helm 3 clients.

For more info, please see [this issue](https://github.com/goharbor/harbor/issues/9260).
