---
title: "Installation"
linkTitle: "Installation"
weight: 20
description: >
  Installation and configuration details for Kyverno using `Helm` or `kubectl`.
---

Kyverno can be installed using Helm or deploying from the YAML manifests directly. When using either of these methods, there are no other steps required to get Kyverno up and running. 

{{% alert title="Note" color="info" %}}
Currently, Kyverno runs as a single replica in the `Deployment` resource. Supporting multiple replicas, for increased scale and availability, is being worked on as a [roadmap item](https://github.com/kyverno/kyverno/issues/1214).
{{% /alert %}}

## Install Kyverno using Helm

Add the Kyverno Helm repository.

```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
```

Create a namespace and then install the Kyverno Helm chart.

```sh
# Create a namespace
kubectl create ns <namespace>

# Install the Kyverno Helm chart
helm install kyverno --namespace <namespace> kyverno/kyverno

```

For installing in the `kyverno` namespace:

```sh
kubectl create ns kyverno

helm install kyverno --namespace kyverno kyverno/kyverno
```

Alternatively, use Helm 3.2+ to complete both steps in a single command:

```sh
helm install kyverno --namespace kyverno kyverno/kyverno --create-namespace
```

To install non-stable releases, add the `--devel` switch to Helm

```sh
helm install kyverno --namespace kyverno kyverno/kyverno --create-namespace --devel
```

## Install Kyverno using YAMLs

If you'd rather deploy the manifest directly, simply apply the latest release file. This manifest path will always point to the latest release, including release candidates and other non-stable releases.

```sh
kubectl create -f https://raw.githubusercontent.com/kyverno/kyverno/main/definitions/release/install.yaml
```

## Customize the installation of Kyverno

If you wish to customize the installation of Kyverno to have certificates signed by an internal or trusted CA, or to otherwise learn how the components work together, follow the below guide.

The Kyverno policy engine runs as an admission webhook and requires a CA-signed certificate and key to setup secure TLS communication with the kube-apiserver (the CA can be self-signed). There are two ways to configure secure communications between Kyverno and the kube-apiserver.

### Option 1: Auto-generate a self-signed CA and certificate

Kyverno can automatically generate a new self-signed Certificate Authority (CA) and a CA signed certificate to use for Webhook registration.  

```sh
## Install Kyverno
kubectl create -f https://raw.githubusercontent.com/kyverno/kyverno/main/definitions/release/install.yaml
```

{{% alert title="Note" color="info" %}}
🛈 The above command installs the last released (stable) version of Kyverno. If you want to install a different version, you can edit the `install.yaml` file and update the image tag.
{{% /alert %}}

Also, by default Kyverno is installed in the "kyverno" namespace. To install it in a different namespace, you can edit `install.yaml` and update the namespace.

To check the Kyverno controller status, run the command:

```sh
## Check pod status
kubectl get pods -n <namespace>
```

If the Kyverno controller is not running, you can check its status and logs for errors:

```sh
kubectl describe pod <kyverno-pod-name> -n <namespace>
```

```sh
kubectl logs <kyverno-pod-name> -n <namespace>
```

### Option 2: Use your own CA-signed certificate

You can install your own CA-signed certificate, or generate a self-signed CA and use it to sign a certificate. Once you have a CA and X.509 certificate-key pair, you can install these as Kubernetes secrets in your cluster. If Kyverno finds these secrets, it uses them. Otherwise it will request the kube-controller-manager to generate a certificate (see Option 1 above).

#### 2.1. Generate a self-signed CA and signed certificate-key pair

{{% alert title="Note" color="warning" %}}
Using a separate self-signed root CA is difficult to manage and not recommended for production use.
{{% /alert %}}

If you already have a CA and a signed certificate, you can directly proceed to Step 2.

Here are the commands to create a self-signed root CA, and generate a signed certificate and key using OpenSSL (you can customize the certificate attributes for your deployment):

1. Create a self-signed CA

```bash
openssl genrsa -out rootCA.key 4096
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt  -subj "/C=US/ST=test/L=test /O=test /OU=PIB/CN=*.kyverno.svc/emailAddress=test@test.com"
```

2. Create a keypair

```bash
openssl genrsa -out webhook.key 4096
openssl req -new -key webhook.key -out webhook.csr  -subj "/C=US/ST=test /L=test /O=test /OU=PIB/CN=kyverno-svc.kyverno.svc/emailAddress=test@test.com"
```

3. Create a **`webhook.ext` file** with the Subject Alternate Names (SAN) to use. This is required with Kubernetes 1.19+ and Go 1.15+.

```
subjectAltName = DNS:kyverno-svc,DNS:kyverno-svc.kyverno,DNS:kyverno-svc.kyverno.svc
```

4. Sign the keypair with the CA passing in the extension

```bash
openssl x509 -req -in webhook.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out webhook.crt -days 1024 -sha256 -extfile webhook.ext
```

5. Verify the contents of the certificate

```bash
 openssl x509 -in webhook.crt -text -noout
```

The certificate must contain the SAN information in the _X509v3 extensions_ section:

```
X509v3 extensions:
    X509v3 Subject Alternative Name:
        DNS:kyverno-svc, DNS:kyverno-svc.kyverno, DNS:kyverno-svc.kyverno.svc
```

#### 2.2. Configure secrets for the CA and TLS certificate-key pair

You can now use the following files to create secrets:

- `rootCA.crt`
- `webhooks.crt`
- `webhooks.key`

To create the required secrets, use the following commands (do not change the secret names):

```bash
kubectl create ns <namespace>
kubectl create secret tls kyverno-svc.kyverno.svc.kyverno-tls-pair --cert=webhook.crt --key=webhook.key -n <namespace>
kubectl annotate secret kyverno-svc.kyverno.svc.kyverno-tls-pair self-signed-cert=true -n <namespace>
kubectl create secret generic kyverno-svc.kyverno.svc.kyverno-tls-ca --from-file=rootCA.crt -n <namespace>
```

{{% alert title="Note" color="info" %}}
The annotation on the TLS pair secret is used by Kyverno to identify the use of self-signed certificates and checks for the required root CA secret.
{{% /alert %}}

Secret | Data | Content
------------ | ------------- | -------------
`kyverno-svc.kyverno.svc.kyverno-tls-pair` | rootCA.crt | root CA used to sign the certificate
`kyverno-svc.kyverno.svc.kyverno-tls-ca` | tls.key & tls.crt  | key and signed certificate

Kyverno uses secrets created above to setup TLS communication with the kube-apiserver and specify the CA bundle to be used to validate the webhook server's certificate in the admission webhook configurations.

#### 2.3. Install Kyverno

You can now install Kyverno by downloading and updating `install.yaml`, or using the command below (assumes that the namespace is **kyverno**):

```sh
kubectl create -f https://raw.githubusercontent.com/kyverno/kyverno/main/definitions/release/install.yaml
```

## Configuring Kyverno

### Permissions

Kyverno, in `foreground` mode, leverages admission webhooks to manage incoming api-requests, and `background` mode applies the policies on existing resources. It uses ServiceAccount `kyverno-service-account`, which is bound to multiple ClusterRoles, which defines the default resources and operations that are permitted.

ClusterRoles used by kyverno:

- `kyverno:webhook`
- `kyverno:userinfo`
- `kyverno:customresources`
- `kyverno:policycontroller`
- `kyverno:generatecontroller`

The `generate` rule creates a new resource, and to allow Kyverno to create resources the Kyverno ClusterRole needs permissions to create/update/delete. This can be done by adding the resource to the ClusterRole `kyverno:generatecontroller` used by Kyverno or by creating a new ClusterRole and a ClusterRoleBinding to Kyverno's default ServiceAccount.

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: kyverno:generatecontroller
rules:
- apiGroups:
  - "*"
  resources:
  - namespaces
  - networkpolicies
  - secrets
  - configmaps
  - resourcequotas
  - limitranges
  - ResourceA # new Resource to be generated
  - ResourceB
  verbs:
  - create # generate new resources
  - get # check the contents of exiting resources
  - update # update existing resource, if required configuration defined in policy is not present
  - delete # clean-up, if the generate trigger resource is deleted
```

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: kyverno-admin-generate
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kyverno:generatecontroller # clusterRole defined above, to manage generated resources
subjects:
- kind: ServiceAccount
  name: kyverno-service-account # default kyverno serviceAccount
  namespace: kyverno
```

### Version

To install a specific version, download `install.yaml` and then change the image tag.

e.g., change image tag from `latest` to the specific tag `v1.0.0`.
>>>
    spec:
      containers:
        - name: kyverno
          # image: nirmata/kyverno:latest
          image: nirmata/kyverno:v1.0.0

To install in a specific namespace replace the namespace "kyverno" with your namespace.

Example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
```

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kyverno
  name: kyverno-svc
  namespace: <namespace>
```

and in other places (ServiceAccount, ClusterRoles, ClusterRoleBindings, ConfigMaps, Service, Deployment) where namespace is mentioned.

To run kyverno:

```sh
kubectl create -f ./install.yaml
```

To check the Kyverno controller status, run the command:

```sh
kubectl get pods -n <namespace>
```

If the Kyverno controller is not running, you can check its status and logs for errors:

```sh
kubectl describe pod <kyverno-pod-name> -n <namespace>
```

```sh
kubectl logs <kyverno-pod-name> -n <namespace>
```

Here is a script that generates a self-signed CA, a TLS certificate-key pair, and the corresponding kubernetes secrets: [helper script](https://github.com/kyverno/kyverno/blob/main/scripts/generate-self-signed-cert-and-k8secrets.sh)

### Flags

1. `excludeGroupRole` : excludeGroupRole role expected string with Comma separated group role. It will exclude all the group role from the user request. Default we are using `system:serviceaccounts:kube-system,system:nodes,system:kube-scheduler`.
2. `excludeUsername` : excludeUsername expected string with Comma separated kubernetes username. In generate request if user enable `Synchronize` in generate policy then only kyverno can update/delete generated resource but admin can exclude specific username who have access of delete/update generated resource.
3. `filterK8Resources`: k8s resource in format [kind,namespace,name] where policy is not evaluated by the admission webhook. For example --filterKind "[Deployment, kyverno, kyverno]" --filterKind "[Deployment, kyverno, kyverno],[Events, *, *].

### PolicyViolation access

During Kyverno installation, it creates a ClusterRole `kyverno:policyviolations` which has the `list,get,watch` operations on resource `policyviolations`. To grant access to a namespace admin, configure the following YAML file then apply to the cluster.

- Replace `metadata.namespace` with namespace of the admin
- Configure `subjects` field to bind admin's role to the ClusterRole `policyviolation`

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: policyviolation
  # change namespace below to create rolebinding for the namespace admin
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: policyviolation
subjects:
# configure below to access policy violation for the namespace admin
- kind: ServiceAccount
  name: default
  namespace: default
# - apiGroup: rbac.authorization.k8s.io
#   kind: User
#   name:
# - apiGroup: rbac.authorization.k8s.io
#   kind: Group
#   name:
```

### Resource Filters

The admission webhook checks if a policy is applicable on all admission requests. The Kubernetes kinds that are not be processed can be filtered by adding a `ConfigMap` in namespace `kyverno` and specifying the resources to be filtered under `data.resourceFilters`. The default name of this `ConfigMap` is `init-config` but can be changed by modifying the value of the environment variable `INIT_CONFIG` in the kyverno deployment spec. `data.resourceFilters` must be a sequence of one or more `[<Kind>,<Namespace>,<Name>]` entries with `*` as wildcard. Thus, an item `[Node,*,*]` means that admissions of `Node` in any namespace and with any name will be ignored.

By default we have specified Nodes, Events, APIService, and SubjectAccessReview as the kinds to be skipped in the default configuration in `install.yaml`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: init-config
  namespace: kyverno
data:
  # resource types to be skipped by kyverno policy engine
  resourceFilters: "[Event,*,*][*,kube-system,*][*,kube-public,*][*,kube-node-lease,*][Node,*,*][APIService,*,*][TokenReview,*,*][SubjectAccessReview,*,*][*,kyverno,*]"
```

To modify the `ConfigMap`, either directly edit the `ConfigMap` `init-config` in the default configuration inside `install.yaml` and redeploy it or modify the `ConfigMap` using `kubectl`.  Changes to the `ConfigMap` through `kubectl` will automatically be picked up at runtime.
