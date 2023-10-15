# cert-manager webhook for Bunny DNS

## Development

### Running the test suite

All DNS providers **must** run the DNS01 provider conformance testing suite,
else they will have undetermined behaviour when used with cert-manager.

**It is essential that you configure and run the test suite when creating a
DNS01 webhook.**

An example Go test file has been provided in [main_test.go](https://gitlab.com/digilol/cert-manager-webhook-bunny/-/blob/master/main_test.go).

You can run the test suite with:

```bash
$ TEST_ZONE_NAME=example.com. make test
```

The example file has a number of areas you must fill in and replace with your
own options in order for tests to pass.

## Webhook
Chart is based on: https://github.com/vadimkim/cert-manager-webhook-hetzner/

### Using public helm chart
```bash
helm repo add cert-manager-webhook-bunny  https://schmitzis.github.io/cert-manager-webhook-bunny /
# Replace the groupName value with your desired domain
helm install --namespace cert-manager cert-manager-webhook-bunny cert-manager-webhook-bunny/cert-manager-webhook-bunny --set groupName=acme.bunny.net
```

### From local checkout

```bash
helm install --namespace cert-manager cert-manager-webhook-bunny deploy/cert-manager-webhook-bunny
```
**Note**: The kubernetes resources used to install the Webhook should be deployed within the same namespace as the cert-manager.

To uninstall the webhook run
```bash
helm uninstall --namespace cert-manager cert-manager-webhook-bunny
```

## Issuer

Create a `ClusterIssuer` or `Issuer` resource as following:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
  namespace: cert-manager
spec:
  acme:
    # Staging: https://acme-staging-v02.api.letsencrypt.org/directory
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: mail@example.com # REPLACE THIS WITH YOUR EMAIL!!!
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - dns01:
          webhook:
            # This group needs to be configured when installing the helm package, otherwise the webhook won't have permission to create an ACME challenge for this API group.
            groupName: acme.bunny.net
            solverName: bunny
            config:
              apiSecretRef:
                Key: accessKey
                Name: bunny-credentials
```

### Credentials
In order to access the Bunny API, the webhook needs an API token.

If you choose another name for the secret than `bunny-secret`, you must install the chart with a modified `secretName` value. Policies ensure that no other secrets can be read by the webhook. Also modify the value of `secretName` in the `[Cluster]Issuer`.

The secret for the example above will look like this:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bunny-credentials
  namespace: cert-manager
type: Opaque
data:
  accessKey: yourbase64encodedaccesskey
```

### Create a certificate

Finally you can create certificates, for example:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: acme-ingress
  namespace: cert-manager
spec:
  commonName: example.com
  dnsNames:
    - example.com
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  secretName: example-cert
```
