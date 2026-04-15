# Agent Tutorial

## Create the secrets to make the agent work

First, export the GitHub token:

```sh
export GITHUB_TOKEN=<PAT>
```

```sh
kubectl create ns autopilot-system

kubectl create secret -n autopilot-system generic ghcr-secret \
  --from-literal=token=$GITHUB_TOKEN

kubectl create secret docker-registry docker-registry-secret \
  --namespace autopilot-system \
  --docker-server=ghcr.io \
  --docker-username=EdmondDantes21 \
  --docker-password=$GITHUB_TOKEN \
  --docker-email=redivreto@gmail.com

kubectl create secret -n autopilot-system generic gcloud-creds \
  --from-file=key.json=/home/redi/serviceaccount.json
```

## Create the Agent `compositionDefinition`

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: agent
  namespace: autopilot-system
spec:
  chart:
    repo: agent
    url: https://charts.krateo.io/
    version: "0.0.8"
    credentials:
      username: "EdmondDantes21"
      passwordRef:
        key: token
        name: ghcr-secret
        namespace: autopilot-system
EOF
```

Wait for the `CompositionDefinition` to be ready:

```sh
kubectl wait compositiondefinition agent -n autopilot-system --for condition=Ready=True --timeout=300s
```

Now the core-provider creates the CRD with kind `Agent` and `apiVersion`, allowing us to personalize the agent we will use.

## Create the Agent

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v0-0-8
kind: Agent
metadata:
  name: primeagent
  namespace: autopilot-system
spec:
  description: "Prime Number Agent. Answers questions about prime numbers."
  instruction: "You are a helpful assistant that answers questions about prime numbers."

  model:
    provider: "GeminiVertexAI"
    name: "gemini-2.5-flash"

    googleCloudProjectID: "autopilot-dev-krateo-io"
    googleCloudLocation: "global"
    googleCloudSecretRef:
      name: gcloud-creds
      namespace: autopilot-system
      key: key.json
  
  imagePullSecrets:
    - name: docker-registry-secret
EOF
```

## Add a Tool Using the MCP Protocol

Create the MCP server `CompositionDefinition`:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: mcp-server
  namespace: autopilot-system
spec:
  chart:
    repo: mcpserver
    url: https://charts.krateo.io/
    version: "0.0.6"
    credentials:
      username: "EdmondDantes21"
      passwordRef:
        key: token
        name: ghcr-secret
        namespace: autopilot-system
EOF
```

Wait for the `CompositionDefinition` to be up and running: 

```sh
kubectl wait compositiondefinition mcp-server -n autopilot-system --for condition=Ready=True --timeout=300s
```

Now create the MCP server:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v0-0-6
kind: Mcpserver
metadata:
  name: math
  namespace: autopilot-system
spec:
  deployment: 
    image:
      repository: redi21/math_mcp
      pullPolicy: IfNotPresent
      tag: "1.0.1"
EOF
``` 

## Update the Agent

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v0-0-8
kind: Agent
metadata:
  name: primeagent
  namespace: autopilot-system
spec:
  description: "Prime Number Agent. Answers questions about prime numbers."
  instruction: "You are a helpful assistant that answers questions about prime numbers."

  model:
    provider: "GeminiVertexAI"
    name: "gemini-2.5-flash"

    googleCloudProjectID: "autopilot-dev-krateo-io"
    googleCloudLocation: "global"
    googleCloudSecretRef:
      name: gcloud-creds
      namespace: autopilot-system
      key: key.json
    
  tools:
    - name: "math"
      namespace: "autopilot-system"
      toolFilter: ["check_prime"]
  
  imagePullSecrets:
    - name: docker-registry-secret
EOF
```

Delete the pod to speed up the connection to the newly created MCP server.