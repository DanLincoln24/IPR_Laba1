## GitOps с Argo CD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: messager-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DanLincoln24/IPR_Laba1.git
    targetRevision: HEAD
    path: k8s/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: messager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
````

## Проверки

- `kubectl kustomize k8s/overlays/dev` – без ошибок
- `kubectl kustomize k8s/overlays/prod` – без ошибок
- Все поды в `Running` (кроме миграций)
- Фронтенд доступен через port-forward
- S3 PVC в статусе `Bound`
- Argo CD Application создан