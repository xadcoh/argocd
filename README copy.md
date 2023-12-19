# kyverno-poc

- We decided not to use ApplicationSet as it wasn't tested by us
- Check helm deep dependency download and configuration
- We can try leverage helm for repo bootstrap

## Manual process (track)

1. Create `charts` folder to store chart versions
    ```bash
    mkdir -p charts
    ```
    - Create a version subfolder and put an umbrella `Chart.yaml` definition there (e.g. referencing `v3.0.0`)
    ```bash
    mkdir -p charts/v3.0.0
    cat <<EOT >> charts/v3.0.0/Chart.yaml
    apiVersion: v2
    appVersion: latest
    name: kyverno-poc
    type: application
    version: 1
    dependencies:
    - name: kyverno
      repository: https://kyverno.github.io/kyverno/
      version: v3.0.0
    EOT
    ```
    - [Optional] Download dependencies and commit them as well
    ```bash
    helm dependency build charts/v3.0.0/
    ```
    - [Optional] Template using env values:
    ```bash
    mkdir -p charts/v3.0.0/environments/cit1
    cat <<EOT >> charts/v3.0.0/environments/cit1/values.yaml
    ---
    kyverno:
      fullnameOverride: test-test-test
    EOT
    helm template charts/v3.0.0/ -f charts/v3.0.0/cit1/values.yaml
    ```

2. Create `argocd` folder to store argocd project and application manifests
    ```bash
    mkdir -p argocd
    ```
    1. Create project manifest `project.yaml`:
    ```bash
    cat <<EOT >> argocd/project.yaml
    ---
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
      name: kyverno-poc
      namespace: argocd
      labels:
        app: kyverno-poc
      # Finalizer that ensures that project is not deleted until it is not referenced by any application
      finalizers:
      - resources-finalizer.argocd.argoproj.io
    spec:
      roles:
      - name: "project-admin"
        policies:
          - "p, proj:kyverno-poc:project-admin, applications, update, kyverno-poc/*, allow"
          - "p, proj:kyverno-poc:project-admin, applications, sync, kyverno-poc/*, allow"
          - "p, proj:kyverno-poc:project-admin, applications, get, kyverno-poc/*, allow"
          - "p, proj:kyverno-poc:project-admin, applications, override, kyverno-poc/*, allow"
          - "p, proj:kyverno-poc:project-admin, repositories, get, kyverno-poc/*, allow"
        groups:
        - res_paloalto_prodops
      clusterResourceWhitelist:
      - group: '*'
        kind: '*'
      namespaceResourceWhitelist:
      - group: '*'
        kind: '*'
      destinations:
      - namespace: "*-kyverno-poc"
        server: "*"
      sourceRepos:
      - "https://gitlab.evolution.com/prod-ops/application-infra/kyverno-poc.git"
    EOT
    ```

    2. Create environment specific application manifest, e.g. `cit1.yaml` with requirements:
    ```bash
    mkdir -p argocd/environments/cit1
    cat <<EOT >> argocd/environments/cit1/application.yaml
    ---
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: cit1-kyverno-poc
      namespace: argocd
    spec:
      project: kyverno-poc
      source:
        repoURL: https://gitlab.evolution.com/prod-ops/application-infra/kyverno-poc.git
        targetRevision: master
        path: charts/v3.0.0
        helm:
          releaseName: cit1-kyverno-poc
          valueFiles:
            - ./cit1/values.yaml
      destination:
        server: "<cit1-api>"
        namespace: cit1-kyverno-poc
      syncPolicy:
        automated:
          prune: false
          selfHeal: false
        retry:
          backoff:
            duration: 2m
            factor: 2
            maxDuration: 10m
          limit: 2
        syncOptions:
        - CreateNamespace=true
    EOT
    ```
