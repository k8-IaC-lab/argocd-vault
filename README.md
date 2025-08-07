# ArgoCD App of Apps: HA Vault Deployment with MetalLB

This repository is designed to be synced via [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) and follows the **App of Apps** pattern. Each application leverages ArgoCD Helm templating.

The end result is a **basic yet highly available instance of [Vault](https://www.vaultproject.io/)**, fronted by [MetalLB](https://metallb.universe.tf/) for load balancing, **with the Vault-ArgoCD Plugin enabled** (deployed in the `argocd` namespace).

> **Note:** Before the cluster can enter a healthy state, a few resources must be manually created. These resources are not included here to avoid exposing internal details of your cluster. Please add them to your own copy of this repository as needed.

---

## Prerequisites

- A Kubernetes cluster (RKE2 or similar)

---

## Required Manual Resources

> **Replace** all instances of `xxx.xxx.x.xxx` with appropriate reserved IPs **within your network**.

### 1. MetalLB IP Address Pool

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: internal
  namespace: metallb-system
spec:
  addresses:
    - xxx.xxx.x.xxx-xxx.xxx.x.xxx
```

### 2. NGINX Ingress Helm Chart Config

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    controller:
      config:
        use-forwarded-headers: "true"
        enable-real-ip: "true"
      publishService:
        enabled: true
      service:
        enabled: true
        type: LoadBalancer
        external:
          enabled: true
        externalTrafficPolicy: Local
        annotations:
          metallb.universe.tf/loadBalancerIPs: xxx.xxx.x.xxx
```

Once these resources are created, **Kubernetes LoadBalancer services will be assigned IPs** within your specified range.

---

## Vault Initialization (Using Raft)

1. **Set VAULT_ADDR**

   ```sh
   export VAULT_ADDR="http://xxx.xxx.x.xxx:8200"
   ```

   _Replace with the IP assigned to Vault’s service. Confirm via:_
   ```sh
   kubectl get svc -n vault
   ```

2. **Install Vault CLI (Linux example)**

   [Follow official instructions](https://developer.hashicorp.com/vault/install)

3. **Initialize Vault**

   ```sh
   vault operator init
   ```

4. **Unseal All Vault Pods**

   For each Vault pod (there should be 3 when HA is enabled):

   ```sh
   kubectl exec -it -n vault vault-0 -- vault operator unseal
   ```

   When prompted, enter one of the 5 unseal keys. Repeat, providing a different key each time (per pod) until `Sealed` is reported as `false`.  
   Repeat this on `vault-1` and `vault-2`.

   Example sequence:

   ```sh
   # For vault-0
   kubectl exec -it -n vault vault-0 -- vault operator unseal
   # Enter key 1
   kubectl exec -it -n vault vault-0 -- vault operator unseal
   # Enter key 2
   # ...

   # Repeat for vault-1 and vault-2
   ```

5. **Verify Vault Pods**

   All Vault pods should now be `1/1 READY` and running.

   ```sh
   kubectl get pods -n vault
   ```

---

## Vault Configuration

6. **Login to Vault**

   ```sh
   vault login
   ```
   _Use the generated Root Token when prompted._

7. **Enable Key/Value Secret Engine**

   ```sh
   vault secrets enable -path=kv -version=2 kv
   ```

8. **Enable AppRole Authentication**

   ```sh
   vault auth enable approle
   ```

9. **Create an AppRole**

   ```sh
   vault write auth/approle/role/<roleName> \
     token_type=batch \
     secret_id_ttl=10m \
     token_ttl=20m \
     token_max_ttl=30m \
     secret_id_num_uses=40
   ```

   _Replace `<roleName>` with your desired name, e.g., `argocd`._

10. **Get the AppRole Role ID**

    ```sh
    vault read auth/approle/role/<roleName>/role-id
    ```

    _Note the `role_id` output; this is needed later._

11. **Generate a Secret ID**

    ```sh
    vault write -f auth/approle/role/argocd/secret-id
    ```

    _Note the `secret_id` output as well._

---

## Create ArgoCD Vault Plugin Secret

Combine all the above information **(base64-encode all values)** and apply as a Kubernetes Secret in the `argocd` namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-vault-plugin-credentials
  namespace: argocd
type: Opaque
data:
  AVP_AUTH_TYPE: <base64:approle>
  AVP_ROLE_ID: <base64:role_id>
  AVP_SECRET_ID: <base64:secret_id>
  AVP_TYPE: <base64:vault>
  VAULT_ADDR: <base64:http://xxx.xxx.x.xxx:8200>
```

---

## Verification

If all steps completed successfully, you should now have a secret in the `argocd` namespace that allows the ArgoCD Vault Plugin to pull secrets from Vault!

---

## Notes

- Encountered a bug where vault pods would not start due to missing pvc. Had to patch the openebs storage class to be the default with
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

---

## References

- [Vault Documentation](https://www.vaultproject.io/docs/)
- [ArgoCD Vault Plugin](https://argocd-vault-plugin.readthedocs.io/)
- [Medium Article about ArgoCD Vault](https://itnext.io/argocd-secret-management-with-argocd-vault-plugin-539f104aff05)

---
