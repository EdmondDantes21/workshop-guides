# Method 2 - Composable Operations

Install the CompositionDefinition for the Blueprint:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: github-scaffolding
  namespace: krateo-system
spec:
  chart:
    repo: github-scaffolding-with-composition-page
    url: https://marketplace.krateo.io
    version: 1.2.5
EOF
```

Now the Krateo core-provider creates a new CRD in the cluster, enabling the creation of resources of kind `GithubScaffoldingWithCompositionPage` and `apiVersion` `composition.krateo.io/v1-2-5`.

Create a compositon of the blueprint as follows:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v1-2-5
kind: GithubScaffoldingWithCompositionPage
metadata:
  name: my-fireworks-app
  namespace: krateo-system 
spec:
  argocd:
    namespace: krateo-system
    application:
      project: default
      source:
        path: chart/
      destination:
        server: https://kubernetes.default.svc
        namespace: fireworks-app
      syncEnabled: false
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
  app:
    service:
      type: NodePort
      port: 30086
  git:
    unsupportedCapabilities: true
    insecure: true
    fromRepo:
      scmUrl: https://github.com
      org: krateoplatformops-blueprints
      name: github-scaffolding-with-composition-page
      branch: main
      path: skeleton
      credentials:
        authMethod: generic
        secretRef:
          namespace: krateo-system
          name: github-repo-creds
          key: token
    toRepo:
      scmUrl: https://github.com
      org: leovice-org
      name: fireworks-app
      branch: main
      path: /
      credentials:
        authMethod: generic
        secretRef:
          namespace: krateo-system
          name: github-repo-creds
          key: token
      private: false
      initialize: true
      deletionPolicy: Delete
      verbose: false
      configurationRef:
        name: repo-config
        namespace: demo-system
EOF
```

The git-provider will create a new repo named `fireworks-app` in the `leovice-org` organization available at `https://github.com/leovice-org/fireworks-app`. 

The generated repo contains the application and the chart, as well as a GitHub action that will publish the image of the application used by the deployment in the chart. 

Argo CD will reconcyle any and all changes in this repo to the cluster.

Now Wait for the GitHub action to publish the image in the packages section.

In the frontend, edit the field `spec.argocd.application.syncEnabled` to `true`.

Argo CD now sees the changes in the GitHub repo and a deployment is created in the cluster in the `fireworks-app` namespace.

Access the fireworksapp at `localhost:30086`.


# Method 3 - Composable Portal

Create a `compositionDefinition` for the `portal-blueprint-page` blueprint:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: portal-blueprint-page
  namespace: krateo-system
spec:
  chart:
    repo: portal-blueprint-page
    url: https://marketplace.krateo.io
    version: 1.0.6
EOF
```

Wait for the `compositiondefinition` to be up:

```sh
kubectl wait compositiondefinition portal-blueprint-page -n krateo-system --for condition=Ready=True
```

Install a composition of the blueprint pointing to the `github-scaffoldind-with-composition-page` blueprint:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v1-0-6
kind: PortalBlueprintPage
metadata:
  name: github-scaffolding-with-composition-page
  namespace: demo-system
spec:
  blueprint:
    repo: github-scaffolding-with-composition-page
    url: https://marketplace.krateo.io
    version: 1.2.5 # this is the Blueprint version
    hasPage: true
  form:
    alphabeticalOrder: false
  panel:
    title: GitHub Scaffolding with Composition Page
    icon:
      name: fa-cubes
EOF
```

Open the blueprints section

Create a new composition, specifying the following fields:

- `name`: `my-fireworks-app-2`
- `namespace`: `krateo-system`
- `git.toRepo.org`: `leovice-org`
- `git.toRepo.name`: `fireworks-app-2`
- `git.fromRepo.path`: `skeleton`

Optionally, port-forward the new service `fireworks-app-2` to access this application too.