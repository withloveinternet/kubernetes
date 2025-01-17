<!-- BEGIN MUNGE: UNVERSIONED_WARNING -->


<!-- END MUNGE: UNVERSIONED_WARNING -->

# Nginx https service

This example creates a basic nginx https service useful in verifying proof of concept, keys, secrets, and end-to-end https service creation in kubernetes.
It uses an [nginx server block](http://wiki.nginx.org/ServerBlockExample) to serve the index page over both http and https.

### Generate certificates

First generate a self signed rsa key and certificate that the server can use for TLS.

```sh
$ make keys secret KEY=/tmp/nginx.key CERT=/tmp/nginx.crt SECRET=/tmp/secret.json
```

### Create a https nginx application running in a kubernetes cluster

You need a [running kubernetes cluster](../../docs/getting-started-guides/) for this to work.

```
$ kubectl create -f /tmp/secret.json
secrets/nginxsecret

$ kubectl create -f examples/https-nginx/nginx-app.yaml
services/nginxsvc
replicationcontrollers/my-nginx

$ kubectl get svc nginxsvc -o json
...
                    {
                        "name": "http",
                        "protocol": "TCP",
                        "port": 80,
                        "targetPort": 80,
                        "nodePort": 30849
                    },
                    {
                        "name": "https",
                        "protocol": "TCP",
                        "port": 443,
                        "targetPort": 443,
                        "nodePort": 30744
                    }
...

$ kubectl get nodes -o json | grep ExternalIP -A 2
...
                        "type": "ExternalIP",
                        "address": "104.197.63.17"
                    }
--
                        "type": "ExternalIP",
                        "address": "104.154.89.170"
                    }
...

$ curl https://nodeip:30744 -k
...
<title>Welcome to nginx!</title>
...
```

For more information on how to run this in a kubernetes cluster, please see the [user-guide](../../docs/user-guide/connecting-applications.md).


<!-- BEGIN MUNGE: IS_VERSIONED -->
<!-- TAG IS_VERSIONED -->
<!-- END MUNGE: IS_VERSIONED -->


<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/examples/https-nginx/README.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->
