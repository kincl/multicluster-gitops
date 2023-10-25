# Cluster Configuration

This directory holds all of the cluster configuration pushed by ACM to the managed clusters

## Repository Filesystem Layout

| Directory | Purpose |
| ---- | ------- |
| applications/ | Where the ApplicationSets that ACM uses on the hub to configure the managed clusters |
| clusters/ | Using Kustomize we build off of the configs in global/ and customize them for the specific cluster |
| global/ | Where all of the generic Kubernetes deployment YAML is |

## Getting Started

### Hub Cluster (groundbreaker-acm)

- install Red Hat OpenShift GitOps Operator
- bootstrap hub cluster ArgoCD instance and link ArgoCD to ACM [(upstream doc)](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/applications/managing-applications#prerequisites-argo) 
using the manifests in `groundbreaker-acm/argocd`

```
$ oc apply -k groundbreaker-acm/argocd
```

| File | Purpose |
| ---- | ------- |
| argocd.yaml | Changes to the OpenShift GitOps cluster install of Argo CD |
| gitopscluster.yaml, placement.yaml | Sets up the link between ACM and openshift-gitops |
| managedclustersetbinding.yaml | Adds the ACM ClusterSet `global` and `default` groupings to ArgoCD |

Once the links are made, ACM will populate the clusters based on the placement rule in the Argo CD instance in the openshift-gitops namespace.

### Configure ArgoCD for GitHub Repository

Since we are using a private repo on GitHub we have to configure ArgoCD to use a token for authentication.

```
$ ssh-keygen -m pem -f groundbreaker-demo-deploy-key
```
```
$ cat <<EOF | oc create -f -
apiVersion: v1
kind: Secret
metadata:
  name: groundbreaker-demo-repo
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/kincl/multicluster-gitops.git
  insecure: "true"
  sshPrivateKey: |
$(cat groundbreaker-demo-deploy-key | sed 's/^/    /')
EOF
```

Add the public key to the list of [deploy keys for the repository.](https://github.com/rh-nspdev/groundbreaker-demo/settings/keys)

#### Debugging ArgoCD

:warning: This section is only required if you are debugging things in the ArgoCD UI and need read/write access to ArgoCD directly! :warning:

By default, users have no permissions in the cluster-wide ArgoCD deployed into the `openshift-gitops` namespace. We will add our user to a cluster-admins group to debug ArgoCD directly.

```
$ oc adm groups new cluster-admins
$ oc adm groups add-users cluster-admins <your username>
```

We are using [sync phases](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) extensively with ArgoCD to place resources in a particular order.

### Deploy Cluster Secrets Management

We will use an ACM Governance Policy to deploy secrets to nodes. We want to do this because we need
to "push" secrets to the managed clusters from the hub cluster but without storing them in Git.

First create the acm-policies project

```
oc new-project acm-policies
```

Add the first policy for managing the Google IDP shared secret for all clusters

```
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: placement-secret-management
  namespace: acm-policies
spec:
  clusterConditions:
  - status: "True"
    type: ManagedClusterConditionAvailable
  clusterSelector:
    matchExpressions:
    - key: vendor
      operator: In
      values:
      - OpenShift
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-secret-management
  namespace: acm-policies
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placement-secret-management
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: secret-management
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: secret-management
  namespace: acm-policies
spec:
  disabled: false
  remediationAction: enforce
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: secrets
      spec:
        object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: v1
            kind: Secret
            metadata:
              name: google-client-secret
              namespace: openshift-config
            data:
              clientSecret: (base64-encoded client secret from google)
        remediationAction: enforce
        severity: low
```

Add the second policy for managing the route53 credentials for certificate management
on bare metal clusters.

```
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: placement-cloud-secret
  namespace: acm-policies
spec:
  clusterConditions:
  - status: "True"
    type: ManagedClusterConditionAvailable
  clusterSelector:
    matchExpressions:
    - key: cloud
      operator: In
      values:
      - BareMetal
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-cloud-secret
  namespace: acm-policies
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placement-cloud-secret
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: cloud-secret
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: cloud-secret
  namespace: acm-policies
spec:
  disabled: false
  remediationAction: enforce
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: cloud-secret
      spec:
        object-templates:
            - complianceType: musthave
              objectDefinition:
                kind: Secret
                apiVersion: v1
                metadata:
                  name: cloud-credentials
                  namespace: cert-manager
                data:
                  aws_access_key_id: REDACTED
                  aws_secret_access_key: REDACTED
                  credentials: REDACTED
                type: Opaque
```

### Deploy Cluster Certificate Management

Now we can deploy the ApplicationSet that manages the cert-manager operator as well as the resources that manage updating the default ingress and API server certificates.

Note: Managed clusters may go offline as a result of changing the API certificates, [ACM has documentation on how to fix it](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html-single/troubleshooting/index#identifying-clusters-offline-after-certificate-change)

```
$ oc apply -f groundbreaker-project/applications/cluster-certificates.yaml
```

We can also deploy our base configuration

```
$ oc apply -f groundbreaker-project/applications/base-config.yaml
```
