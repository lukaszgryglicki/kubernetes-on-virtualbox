# MetalLB Helm (broken)

This is currently broken because helm chart's config map is interferring with metalLB's config map so the latter cannot read its own configuration.

- Install MetalLB [reference](https://github.com/bitnami/charts/tree/master/bitnami/metallb):
- Add metalLB Helm chart repository: `helm repo add bitnami https://charts.bitnami.com/bitnami`.
- Create `metallb-system` namespace: `k create ns metallb-system`.
- Install chart: `helm install -n metallb-system metallb bitnami/metallb`.
- Hack: Backup helm config map (seems like it conflicts with MetalLB configuration in the same namespace): `k get cm -n metallb-system metallb -o yaml > metallb-helm.yaml`
- Hack: Delete Helm configmap: `k delete cm -n metallb-system metallb`.
- Create MetalLB configuration - specify `master` IP for `test` namespace and `node-0` IP for `prod` namespace, create file `metallb-config.yaml` and apply if `k apply -f metallb-config.yaml`:
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: metallb-config
data:
  config: |
    address-pools:
    - name: prod
      protocol: layer2
      # auto-assign: false
      # avoid-buggy-ips: true
      addresses:
      - 10.13.13.102/32
    - name: test
      protocol: layer2
      addresses:
      - 10.13.13.101/32
```
