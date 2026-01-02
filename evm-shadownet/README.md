# EVM Shadownet Helm Chart

This document outlines the steps used for creating a Helm chart for an EVM (Ethereum Virtual Machine) node deployment.

## Prerequisites

Before creating the Helm chart, ensure you have the following:

- Kubernetes cluster (v1.19+)
- Helm 3.x installed
- kubectl configured to access your cluster
- Basic understanding of Kubernetes resources (Deployments, Services, ConfigMaps, etc.)
- Docker image for the EVM node

## Steps to Create the Helm Chart

### 1. Initialize the Helm Chart

Create a new Helm chart structure:

```bash
helm create evm-shadownet
cd evm-shadownet
```

This creates the basic structure:
```
evm-shadownet/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── charts/
```

### 2. Configure Chart.yaml

Update the `Chart.yaml` with EVM node specific information:

```yaml
apiVersion: v2
name: evm-shadownet
description: A Helm chart for EVM Shadownet node
type: application
version: 0.1.0
appVersion: "1.0"
keywords:
  - ethereum
  - evm
  - blockchain
  - shadownet
maintainers:
  - name: Your Name
    email: your.email@example.com
```

### 3. Define values.yaml

Configure default values for the EVM node deployment:

```yaml
replicaCount: 1

image:
  repository: ethereum/client-go  # or your EVM node image
  pullPolicy: IfNotPresent
  tag: "stable"

service:
  type: ClusterIP
  ports:
    rpc:
      port: 8545
      targetPort: 8545
      protocol: TCP
    ws:
      port: 8546
      targetPort: 8546
      protocol: TCP
    p2p:
      port: 30303
      targetPort: 30303
      protocol: TCP

resources:
  limits:
    cpu: 2000m
    memory: 4Gi
  requests:
    cpu: 1000m
    memory: 2Gi

persistence:
  enabled: true
  storageClass: ""
  accessMode: ReadWriteOnce
  size: 100Gi
  mountPath: /data

nodeSelector: {}
tolerations: []
affinity: {}

config:
  networkId: "1"
  syncMode: "fast"
  gcMode: "archive"
  rpcEnabled: true
  wsEnabled: true
```

### 4. Create Deployment Template

Create `templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "evm-shadownet.fullname" . }}
  labels:
    {{- include "evm-shadownet.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "evm-shadownet.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "evm-shadownet.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: rpc
          containerPort: {{ .Values.service.ports.rpc.targetPort }}
          protocol: TCP
        - name: ws
          containerPort: {{ .Values.service.ports.ws.targetPort }}
          protocol: TCP
        - name: p2p
          containerPort: {{ .Values.service.ports.p2p.targetPort }}
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: {{ .Values.persistence.mountPath }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
      volumes:
      - name: data
        {{- if .Values.persistence.enabled }}
        persistentVolumeClaim:
          claimName: {{ include "evm-shadownet.fullname" . }}
        {{- else }}
        emptyDir: {}
        {{- end }}
```

### 5. Create Service Template

Create `templates/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "evm-shadownet.fullname" . }}
  labels:
    {{- include "evm-shadownet.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.ports.rpc.port }}
    targetPort: rpc
    protocol: {{ .Values.service.ports.rpc.protocol }}
    name: rpc
  - port: {{ .Values.service.ports.ws.port }}
    targetPort: ws
    protocol: {{ .Values.service.ports.ws.protocol }}
    name: ws
  - port: {{ .Values.service.ports.p2p.port }}
    targetPort: p2p
    protocol: {{ .Values.service.ports.p2p.protocol }}
    name: p2p
  selector:
    {{- include "evm-shadownet.selectorLabels" . | nindent 4 }}
```

### 6. Create PersistentVolumeClaim Template

Create `templates/pvc.yaml`:

```yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "evm-shadownet.fullname" . }}
  labels:
    {{- include "evm-shadownet.labels" . | nindent 4 }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode }}
  {{- if .Values.persistence.storageClass }}
  storageClassName: {{ .Values.persistence.storageClass }}
  {{- end }}
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
{{- end }}
```

### 7. Create ConfigMap Template (Optional)

Create `templates/configmap.yaml` for EVM node configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "evm-shadownet.fullname" . }}
  labels:
    {{- include "evm-shadownet.labels" . | nindent 4 }}
data:
  config.toml: |
    [Eth]
    NetworkId = {{ .Values.config.networkId }}
    SyncMode = "{{ .Values.config.syncMode }}"
    GCMode = "{{ .Values.config.gcMode }}"
```

### 8. Update Helper Templates

Update `templates/_helpers.tpl` with necessary template functions:

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "evm-shadownet.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "evm-shadownet.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "evm-shadownet.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "evm-shadownet.labels" -}}
helm.sh/chart: {{ include "evm-shadownet.chart" . }}
{{ include "evm-shadownet.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "evm-shadownet.selectorLabels" -}}
app.kubernetes.io/name: {{ include "evm-shadownet.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

## Deployment Steps

### 1. Validate the Chart

```bash
helm lint evm-shadownet
```

### 2. Dry Run Installation

Test the chart without actually deploying:

```bash
helm install evm-shadownet ./evm-shadownet --dry-run --debug
```

### 3. Install the Chart

Deploy to your Kubernetes cluster:

```bash
helm install evm-shadownet ./evm-shadownet --namespace blockchain --create-namespace
```

### 4. Verify the Deployment

Check the deployment status:

```bash
kubectl get pods -n blockchain
kubectl get svc -n blockchain
kubectl logs -f <pod-name> -n blockchain
```

### 5. Upgrade the Chart

Update the chart with new values:

```bash
helm upgrade evm-shadownet ./evm-shadownet --namespace blockchain
```

### 6. Uninstall the Chart

Remove the deployment:

```bash
helm uninstall evm-shadownet --namespace blockchain
```

## Customization

### Override Values

Create a custom `values.yaml` file:

```yaml
# custom-values.yaml
replicaCount: 3

resources:
  limits:
    cpu: 4000m
    memory: 8Gi
  requests:
    cpu: 2000m
    memory: 4Gi

persistence:
  size: 500Gi
```

Install with custom values:

```bash
helm install evm-shadownet ./evm-shadownet -f custom-values.yaml
```

### Command-line Overrides

```bash
helm install evm-shadownet ./evm-shadownet \
  --set replicaCount=2 \
  --set persistence.size=200Gi \
  --set image.tag=latest
```

## Testing

### 1. Test RPC Endpoint

Port-forward to access the RPC endpoint:

```bash
kubectl port-forward svc/evm-shadownet 8545:8545 -n blockchain
```

Test with curl:

```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://localhost:8545
```

### 2. Check Sync Status

```bash
kubectl exec -it <pod-name> -n blockchain -- geth attach /data/geth.ipc
> eth.syncing
```

## Best Practices

1. **Resource Management**: Always set resource requests and limits
2. **Persistence**: Use PersistentVolumes for blockchain data
3. **Monitoring**: Add Prometheus metrics endpoints
4. **Security**: Use NetworkPolicies to restrict traffic
5. **High Availability**: Consider running multiple replicas with proper configuration
6. **Backup**: Implement regular backup strategies for blockchain data
7. **Updates**: Use rolling updates for zero-downtime deployments

## Troubleshooting

### Pod not starting

```bash
kubectl describe pod <pod-name> -n blockchain
kubectl logs <pod-name> -n blockchain
```

### PVC issues

```bash
kubectl get pvc -n blockchain
kubectl describe pvc <pvc-name> -n blockchain
```

### Service connectivity

```bash
kubectl get svc -n blockchain
kubectl describe svc evm-shadownet -n blockchain
```

## Additional Resources

- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [EVM Node Documentation](https://geth.ethereum.org/docs/)
- [Best Practices for Helm Charts](https://helm.sh/docs/chart_best_practices/)

## Support

For issues or questions, please open an issue in the repository.
