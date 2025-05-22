# Installation

1. Install k0rdent >= 0.3.0
2. Wait for `Management` object readiness

```bash
kubectl wait --for=condition=Ready=True management/kcm --timeout=300s
```

3. Install `OCI` chart

```bash
helm install oci-pp oci://ghcr.io/mirgdenis/oci-ccm/charts/oci-ccm:0.1.0 -n kcm-system
```

4. Update Management object to enable OCI provider

```bash
kubectl patch mgmt kcm \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/providers/-",
      "value": {
        "name": "cluster-api-provider-oci",
        "template": "cluster-api-provider-oci-0-2-1",
      }
    }
  ]'
```

5. Wait for `Management` object readiness

```bash
kubectl wait --for=condition=Ready=True management/kcm --timeout=300s
```

> Note: Steps 6 and 7 may be temporary, please check actual state of k0rdent and dynamic RBAC for pluggable providers

6. Change kcm-manager-role cluster role by adding `ociclusteridentities` to element with other cluster identities (e.g. `awsclusterstaticidentities`) and adding `ociclusters` and `ocimanagedclusters` to element with other clusters (e.g. `awsclusters`)

```bash
kubectl edit clusterrole kcm-manager-role
```

7. Rollout restart kcm-controller-manager

```bash
kubect rollout restart -n kcm-system deploy kcm-controller-manager
```

# Cluster deployment

## Prerequisites
envsubst (https://github.com/a8m/envsubst) installed

## Deployment
1. Create file with env variables using following example and source it

```
export NAMESPACE=kcm-system
export CLUSTER_NAME_SUFFIX=test
export OCI_COMPARTMENT=<COMPARTMENT_OCID>
export OCI_KEY="    -----BEGIN PRIVATE KEY-----
    <OCI_KEY_CONTENT>
    -----END PRIVATE KEY-----"
export OCI_FINGERPRINT=<KEY_FINGERPRINT>
export OCI_REGION=<OCI_REGION>
export OCI_TENANCY=<OCI_TENANCY_OCID>
export OCI_USER=<OCI_USER_OCID>
export OCI_IMAGE_ID=<OCI_IMAGE_OCID>
export OCI_PUB_SSH=<SSH_PUB_KEY>
```

2. Render cluster and cred files

```
cat dev/oci-credentials.yaml | envsubst -no-unset > cred.yaml
cat dev/oci-clusterdeployment.yaml | envsubst -no-unset > cluster.yaml
```

3. Apply creds.yaml and check if credentials are valid

```
kubectl apply -f cred.yaml
kubectl get cred -n kcm-system
```

4. Apply cluster.yaml

```
kubectl apply -f cluster.yaml
```

5. Check capoci-controller-manager logs, and status of cld, ocicluster, ocimachines objects

```
kubectl logs -n n kcm-system deploy/capoci-controller-manager
kubectl get cld,ocicluster,ocimachines -n kcm-system
```

6. If cluster is ready, then you can extract kubeconfig and check it

```
kubectl get secret oci-test-kubeconfig -n kcm-system -o yaml | yq .data.value | base64 -d > kubeconfig
kubectl --kubeconfig ./kubeconfig get po -A
```
