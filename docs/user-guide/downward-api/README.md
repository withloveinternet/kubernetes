<!-- BEGIN MUNGE: UNVERSIONED_WARNING -->


<!-- END MUNGE: UNVERSIONED_WARNING -->

# Downward API example

Following this example, you will create a pod with a containers that consumes the pod's name and
namespace using the [downward API](../downward-api.md).

## Step Zero: Prerequisites

This example assumes you have a Kubernetes cluster installed and running, and that you have
installed the `kubectl` command line tool somewhere in your path. Please see the [getting
started](../../../docs/getting-started-guides/) for installation instructions for your platform.

## Step One: Create the pod

Containers consume the downward API using environment variables.  The downward API allows
containers to be injected with the name and namespace of the pod the container is in.

Use the [`examples/downward-api/dapi-pod.yaml`](dapi-pod.yaml) file to create a Pod with a container that consumes the
downward API.

```console
$ kubectl create -f docs/user-guide/downward-api/dapi-pod.yaml
```

### Examine the logs

This pod runs the `env` command in a container that consumes the downward API.  You can grep
through the pod logs to see that the pod was injected with the correct values:

```console
$ kubectl logs dapi-test-pod | grep POD_
2015-04-30T20:22:18.568024817Z POD_NAME=dapi-test-pod
2015-04-30T20:22:18.568087688Z POD_NAMESPACE=default
```


<!-- BEGIN MUNGE: IS_VERSIONED -->
<!-- TAG IS_VERSIONED -->
<!-- END MUNGE: IS_VERSIONED -->


<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/docs/user-guide/downward-api/README.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->
