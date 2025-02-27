---
title: Validating Admission Controller
---

The {{site.kic_product_name}} ships with an Admission Controller for KongPlugin
and KongConsumer resources in the `configuration.konghq.com` API group.

The Admission Controller needs a TLS certificate and key pair which
you need to generate as part of the deployment.

Following guide walks through a setup of how to create the required key-pair
and enable the admission controller.

Please note that this requires {{site.kic_product_name}} >= 0.6 to be
already installed in the cluster.

## Set up with a script

If you are using the stock YAML manifests to install and setup Kong for
Kubernetes, then you can set up the admission webhook using a single command:

```bash
curl -sL https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/main/hack/deploy-admission-controller.sh | bash
```
The output is similar to the following:
```
Generating a 2048 bit RSA private key
.......+++
.......................................................................+++
writing new private key to '/var/folders/h2/chkzcfsn4sl3nn99tk5551tc0000gp/T/tmp.SX3eOgD0/tls.key'
-----
secret/kong-validation-webhook created
deployment.apps/ingress-kong patched
validatingwebhookconfiguration.admissionregistration.k8s.io/kong-validations created
```

This script takes all the following commands and packs them together.
You need `kubectl` and `openssl` installed on your workstation for this to
work.

## Create a certificate for the admission controller

Kubernetes API-server makes an HTTPS call to the Admission Controller to verify
if the custom resource is valid or not. For this to work, Kubernetes API-server
needs to trust the CA certificate that is used to sign Admission Controller's
TLS certificate.

This can be accomplished either using a self-signed certificate or using
Kubernetes CA. Follow one of the steps below and then go to
[Create the secret](#create-the-secret) step below.

Please note the `CN` field of the x509 certificate takes the form
`<validation-service-name>.<ingress-controller-namespace>.svc`, which
in the default case is `kong-validation-webhook.kong.svc`.

### Using self-signed certificate

Use `openssl` to generate a self-signed certificate:

```bash
openssl req -x509 -newkey rsa:2048 -keyout tls.key -out tls.crt -days 365  \
    -nodes -subj "/CN=kong-validation-webhook.kong.svc" \
    -extensions EXT -config <( \
   printf "[dn]\nCN=kong-validation-webhook.kong.svc\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:kong-validation-webhook.kong.svc\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
```

The output is similar to the following:
```
Generating a 2048 bit RSA private key
..........................................................+++
.............+++
writing new private key to 'key.pem'
```

### Using in-built Kubernetes CA

Kubernetes comes with an in-built CA which can be used to provision
a certificate for the Admission Controller.
Please refer to the
[this guide](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)
on how to generate a certificate using the in-built CA.

### Create the secret

Next, create a Kubernetes secret object based on the key and certificate that
was generated in the previous steps.
Here, we assume that the PEM-encoded certificate is stored in a file named
`tls.crt` and private key is stored in `tls.key`.

```bash
kubectl create secret tls kong-validation-webhook -n kong \
    --key tls.key --cert tls.crt
```
The output is similar to the following:
```
secret/kong-validation-webhook created
```

## Update the deployment

Once the secret is created, update the Ingress Controller deployment:

Execute the following command to patch the {{site.kic_product_name}} deployment
to mount the certificate and key pair and also enable the admission controller:

```bash
kubectl patch deploy -n kong ingress-kong \
    -p '{"spec":{"template":{"spec":{"containers":[{"name":"ingress-controller","env":[{"name":"CONTROLLER_ADMISSION_WEBHOOK_LISTEN","value":":8080"}],"volumeMounts":[{"name":"validation-webhook","mountPath":"/admission-webhook"}]}],"volumes":[{"secret":{"secretName":"kong-validation-webhook"},"name":"validation-webhook"}]}}}}'
```
The output is similar to the following:
```
deployment.extensions/ingress-kong patched
```

## Enable the validating admission

If you are using Kubernetes CA to generate the certificate, you don't need
to supply a CA certificate (in the `caBunde` parameter)
as part of the Validation Webhook configuration
as the API-server already trusts the internal CA.

```bash
echo "apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: kong-validations
webhooks:
- name: validations.kong.konghq.com
  objectSelector:
    matchExpressions:
    - key: owner
      operator: NotIn
      values:
      - helm
  failurePolicy: Fail
  sideEffects: None
  admissionReviewVersions: ["v1", "v1beta1"]
  rules:
  - apiGroups:
    - configuration.konghq.com
    apiVersions:
    - '*'
    operations:
    - CREATE
    - UPDATE
    resources:
    - kongconsumers
    - kongplugins
  - apiGroups:
    - ''
    apiVersions:
    - 'v1'
    operations:
    - UPDATE
    resources:
    - secrets
  clientConfig:
    service:
      namespace: kong
      name: kong-validation-webhook
    caBundle: $(cat tls.crt  | base64) " | kubectl apply -f -
```
The output is similar to the following:
```
validatingwebhookconfiguration.admissionregistration.k8s.io/kong-validations configured
```
## Verify if it works

### Verify duplicate KongConsumers

Create a KongConsumer with username as `harry`:

```bash
echo "apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: harry
  annotations:
    kubernetes.io/ingress.class: kong
username: harry" | kubectl apply -f -
```
The output is similar to the following:
```
kongconsumer.configuration.konghq.com/harry created
```

Now, create another KongConsumer with the same username:

```bash
echo "apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: harry2
  annotations:
    kubernetes.io/ingress.class: kong
username: harry" | kubectl apply -f -
```
The output is similar to the following:
```
Error from server: error when creating "STDIN": admission webhook "validations.kong.konghq.com" denied the request: consumer already exists
```

The validation webhook rejected the KongConsumer resource as there already
exists a consumer in Kong with the same username.

### Verify incorrect KongPlugins

Try to create the following KongPlugin resource.
The `foo` config property does not exist in the configuration definition and
hence the Admission Controller returns back an error.
If you remove the `foo: bar` configuration line, the plugin will be
created successfully.

```bash
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-id
config:
  foo: bar
  header_name: my-request-id
plugin: correlation-id
" | kubectl apply -f -
```
The output is similar to the following:
```
Error from server: error when creating "STDIN": admission webhook "validations.kong.konghq.com" denied the request: 400 Bad Request {"fields":{"config":{"foo":"unknown field"}},"name":"schema violation","code":2,"message":"schema violation (config.foo: unknown field)"}
```

### Verify incorrect credential secrets

With 0.7 and above versions of the controller, validations also take place
for incorrect secret types and wrong parameters to the secrets:

```bash
kubectl create secret generic some-credential \
  --from-literal=kongCredType=basic-auth \
  --from-literal=username=foo
```
The output is similar to the following:
```
Error from server: admission webhook "validations.kong.konghq.com" denied the request: missing required field(s): password
```

```bash
kubectl create secret generic some-credential \
  --from-literal=kongCredType=wrong-auth \
  --from-literal=sdfkey=my-sooper-secret-key
```
The output is similar to the following:
```
Error from server: admission webhook "validations.kong.konghq.com" denied the request: invalid credential type: wrong-auth
```
