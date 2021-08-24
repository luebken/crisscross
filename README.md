# Crisscross

Crisscross is a special [Crossplane](https://crossplane.io/) provider that
allows you to write "Providers as a Function" (PaaF).

## How It Works

Traditional Crossplane providers utilize
[crossplane-runtime](https://github.com/crossplane/crossplane-runtime) to defer
much of the logic for interacting with Kubernetes objects to the generic managed
resource reconciler it implements. You can find more documentation on this model
[here](https://crossplane.io/docs/v0.14/contributing/provider_development_guide.html),
or check out one of the many providers in the Crossplane community. This is a
powerful model that has allowed many community members to create new providers
in a few hours. However, though much of the code is boilerplate from a template
such as [provider-template](https://github.com/crossplane/provider-template),
there is still quite a lot of machinery required to add support for a simple
API. This can be especially challenging if you are not familiar with Kubernetes
controllers.

Crisscross shifts more of the burden that currently resides on provider authors
to a single system. Instead of provider authors implementing many managed
reconcilers for their different API types, they deploy a single service and a
corresponding `Registration` object in their cluster for each API type. When the
`Registration` is created, Crisscross spins up a new controller watching for the
referenced API type that will call out to the service at the supplied endpoint.

For example, a `Registration` for a `Bucket` on GCP could look as follows:

```yaml
apiVersion: crisscross.crossplane.io/v1alpha1
kind: Registration
metadata:
  name: bucket-paaf
spec:
  typeRef:
    apiVersion: storage.gcp.crossplane.io/v1alpha3
    kind: Bucket
  endpoint: http://172.18.0.2:32062
```

The `spec.endpoint` field can point to any service, whether it is a traditional
`Pod`, a [Knative](https://knative.dev/) `Service`, or a public API on the
internet. The only requirement for this service is that it serves the following
methods:

- `/observe`
- `/create`
- `/update`
- `/delete`

Crisscross will send the managed resource to these endpoints, and will take
appropriate action in the cluster based on the response. For an example of how
simple a "PaaF" can be, take a look at [nop-paaf](/examples/nop-paaf), which
just reports that a resource exists and all other operations are no-ops.

## Getting started

Install the provider: 
```
kubectl crossplane install provider hasheddan/crisscross
```
Check the provider:
```
kubectl get providers
NAME         INSTALLED   HEALTHY   PACKAGE                AGE
crisscross   True        True      hasheddan/crisscross   41s
````
Find the crisscross pod: 
```
kubectl get pods -n crossplane-system
NAME                                       READY   STATUS    RESTARTS   AGE
crisscross-d6b6774f9279-7796f645f6-pnvnl   1/1     Running   0          118s
```
Inspect the logs:
```
kubectl logs -f crisscross-d6b6774f9279-7796f645f6-pnvnl  -n crossplane-system
```
For more logs install the debug ControllerConfig: https://crossplane.io/docs/v1.3/reference/troubleshoot.html#provider-logs


Install an example managed resource which we are going to register a crisscross
registration for:
```
# Create a (hopefully) unique S3 bucket name
BUCKET_NAME=test-bucket-$RANDOM

# Install an S3 bucket
cat <<EOF | kubectl apply -f -
apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: $BUCKET_NAME
spec:
  forProvider:
    acl: public-read-write
    locationConstraint: us-east-1
EOF
```

Build and install a service deployment:
```
cd examples/nop-paaf
docker build . -t luebken/nop-paaf
docker push luebken/nop-paaf
kubectl apply -f deploy.yaml
kubectl expose deployment nop-paaf --type=NodePort --name=nop-paaf
```

Check the service:
```
kubectl get pods
kubectl get svc #grab cluster-api
```

RBAC for criss-cross
```
kubectl apply -f cluster-role.yaml
kubectl apply -f cluster-rolebinding.yaml #get the right service account via kubectl get sa -n crossplane-system
```

Create the registration: 
```
kubectl apply -f examples/registration.yaml #get the endpoint from the exposed deplyoment clusterip
```

## License

Crisscross is under the Apache 2.0 license.
