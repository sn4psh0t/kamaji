# Kamaji Development

## Prerequisites

Make sure you have these tools installed:

- [Go 1.18+](https://golang.org/dl/)
- [Operator SDK 1.7.2+](https://github.com/operator-framework/operator-sdk), or [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)
- [KinD](https://github.com/kubernetes-sigs/kind)
- [golangci-lint](https://github.com/golangci/golangci-lint)
- OpenSSL

## Setup a Kubernetes Cluster

A lightweight Kubernetes within your laptop can be very handy for Kubernetes-native development like Kamaji.

### By `kind`

```shell
# # Install kind cli by brew in Mac, or your preferred way
$ brew install kind

# run the kind make target
$ make -C deploy/kind
```

## Fork, build, and deploy Kamaji

The `fork-clone-contribute-pr` flow is common for contributing to OSS projects like Kubernetes and Kamaji.

Let's assume you've forked it into your GitHub namespace, say `myuser`, and then you can clone it with Git protocol.
Do remember to change the `myuser` to yours.

```shell
$ git clone git@github.com:myuser/kamaji.git && cd kamaji
```

It's a good practice to add the upstream as the remote too so we can easily fetch and merge the upstream to our fork:

```shell
$ git remote add upstream https://github.com/clastix/kamaji.git
$ git remote -vv
origin	git@github.com:myuser/kamaji.git (fetch)
origin	git@github.com:myuser/kamaji.git (push)
upstream	https://github.com/clastix/kamaji.git (fetch)
upstream	https://github.com/clastix/kamaji.git (push)
```

Pull all tags

```
$ git fetch --all && git pull upstream
```

Build and deploy:

```shell
# Download the project dependencies
$ go mod download

# Build the kamaji image
$ make docker-build

# Retrieve the built image version
$ export KAMAJI_IMAGE_VESION=`docker images --format '{{.Tag}}' clastix/kamaji`

# If Kind, load the image into cluster by
$ kind load docker-image --name kind-kamaji clastix/kamaji:${KAMAJI_IMAGE_VESION}

# deploy all the required manifests
# Note: 1) please retry if you saw errors; 2) if you want to clean it up first, run: make undeploy
$ make deploy

# Make sure the controller is running
$ kubectl get pod -n kamaji-system
NAME                                          READY   STATUS    RESTARTS   AGE
kamaji-controller-manager-5c6b8445cf-566dc   1/1     Running   0          23s

# Check the logs if needed
$ kubectl -n kamaji-system logs --all-containers -l control-plane=controller-manager
```

As of now, a complete Kamaji environment has been set up in `kind`-powered cluster, and the `kamaji-controller-manager` is running as a deployment serving as:

- The reconcilers for CRDs and;
- A series of webhooks

## Setup the development environment

During development, we prefer that the code is running within our IDE locally, instead of running as the normal Pod(s) within the Kubernetes cluster.

To achieve that, there are some necessary steps we need to walk through, which have been made as a `make` target names `dev-setup` within our `Makefile`.

So the answer is:

```shell
# If you haven't installed or run `make deploy` before, do it first
# Note: please retry if you saw errors
# $ make deploy

# To retrieve your laptop's IP and execute `make dev-setup` to setup dev env
# For example: LAPTOP_HOST_IP=192.168.10.101 make dev-setup
$ LAPTOP_HOST_IP="<YOUR_LAPTOP_IP>" make dev-setup
```

This is a very common setup for typical Kubernetes Operator development so we'd better walk them through with more details here.

1. Scaling down the deployed Pod(s) to 0

We need to scale the existing replicas of `kamaji-controller-manager` to 0 to avoid reconciliation competition between the Pod(s) and the code running outside of the cluster, in our preferred IDE for example.

```shell
$ kubectl -n kamaji-system scale deployment kamaji-controller-manager --replicas=0
deployment.apps/kamaji-controller-manager scaled
```

2. Preparing TLS certificate for the webhooks

Running webhooks requires TLS, we can prepare the TLS key pair in our development env to handle HTTPS requests.

```shell
# Prepare a simple OpenSSL config file
# Do remember to export LAPTOP_HOST_IP before running this command
$ cat > _tls.cnf <<EOF
[ req ]
default_bits       = 4096
distinguished_name = req_distinguished_name
req_extensions     = req_ext
[ req_distinguished_name ]
countryName                = SG
stateOrProvinceName        = SG
localityName               = SG
organizationName           = KAMAJI
commonName                 = KAMAJI
[ req_ext ]
subjectAltName = @alt_names
[alt_names]
IP.1   = ${LAPTOP_HOST_IP}
EOF

# Create this dir to mimic the Pod mount point
$ mkdir -p /tmp/k8s-webhook-server/serving-certs

# Generate the TLS cert/key under /tmp/k8s-webhook-server/serving-certs
$ openssl req -newkey rsa:4096 -days 3650 -nodes -x509 \
  -subj "/C=SG/ST=SG/L=SG/O=KAMAJI/CN=KAMAJI" \
  -extensions req_ext \
  -config _tls.cnf \
  -keyout /tmp/k8s-webhook-server/serving-certs/tls.key \
  -out /tmp/k8s-webhook-server/serving-certs/tls.crt

# Clean it up
$ rm -f _tls.cnf
```

3. Patching the Webhooks

By default, the webhooks will be registered with the services, which will route to the Pods, inside the cluster.

We need to _delegate_ the controllers' and webbooks' services to the code running in our IDE by patching the `MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration`.
We do also have to avoid `cert-manager` to inject certficates from `kamaji-serving-cert` into the webhooks. 

```shell
# Export your laptop's IP with the 9443 port exposed by controllers/webhooks' services
$ export WEBHOOK_URL="https://${LAPTOP_HOST_IP}:9443"

# Export the cert we just generated as the CA bundle for webhook TLS
$ export CA_BUNDLE=`openssl base64 -in /tmp/k8s-webhook-server/serving-certs/tls.crt | tr -d '\n'`

# Patch the MutatingWebhookConfiguration webhook
$ kubectl patch MutatingWebhookConfiguration kamaji-mutating-webhook-configuration \
    --type='json' -p="[\
      {'op': 'replace', 'path': '/webhooks/1/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/mutate-kamaji-clastix-io-v1alpha1-tenantcontrolplane\",'caBundle':\"${CA_BUNDLE}\"}},\
      {'op': 'replace', 'path': '/metadata/annotations/cert-manager.io~1inject-ca-from', 'value':\"\"}\
    ]"

# Verify it if you want
$ kubectl get MutatingWebhookConfiguration kamaji-mutating-webhook-configuration -o yaml

# Patch the ValidatingWebhookConfiguration webhooks
# Note: there is a list of validating webhook endpoints, not just one
$ kubectl patch ValidatingWebhookConfiguration kamaji-validating-webhook-configuration \
    --type='json' -p="[\
       {'op': 'replace', 'path': '/webhooks/2/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/validate-kamaji-clastix-io-v1alpha1-tenantcontrolplane\", 'caBundle': \"${CA_BUNDLE}\"}},\
	   {'op': 'replace', 'path': '/metadata/annotations/cert-manager.io~1inject-ca-from', 'value':\"\"}\
    ]"

# Verify it if you want
$ kubectl get ValidatingWebhookConfiguration kamaji-validating-webhook-configuration -o yaml

# Patch the crd
$ kubectl patch crd tenantcontrolplanes.kamaji.clastix.io \
    --type='json' -p="[\
        {'op': 'replace', 'path': '/spec/conversion/webhook/clientConfig', 'value':{'url': \"$${WEBHOOK_URL}\", 'caBundle': \"$${CA_BUNDLE}\"}},\
        {'op': 'replace', 'path': '/metadata/annotations/cert-manager.io~1inject-ca-from', 'value':\"\"}\
    ]";
```

## Run Kamaji outside the cluster

Now we can run Kamaji controllers with webhooks outside of the Kubernetes cluster:

```shell
$ export NAMESPACE=kamaji-system && export TMPDIR=/tmp/
$ go run .
```

We should see output and logs in the `make run` console.

Now it's time to work through our familiar inner loop for development in our preferred IDE. For example, if you're using [Visual Studio Code](https://code.visualstudio.com), this `launch.json` file can be a good start.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}",
            "args": [
                "manager", 
                "--health-probe-bind-address=:8081", 
                "--leader-elect",
                "--metrics-bind-address=:8080",
                "--datastore=default", 
                "--pod-namespace=kamaji-system",
                "--serviceaccount-name=kamaji-controller-manager",
                "--zap-encoder=console",
                "--zap-log-level=debug"
            ],
            "env": {
                "NAMESPACE": "kamaji-system",
                "TMPDIR": "/tmp/"
            }
        }
    ]
}
```

