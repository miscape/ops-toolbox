La [versione indicata attualmente]() è v1.8.2 e l’installazione Helm applica anche le CRD Gateway API/Envoy Gateway.

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.8.2 \
  -n envoy-gateway-system \
  --create-namespace
```

Attendi che il controller sia pronto:
```bash
kubectl wait --timeout=5m \
  -n envoy-gateway-system \
  deployment/envoy-gateway \
  --for=condition=Available
```

Verifiche:
```bash
kubectl get pods -n envoy-gateway-system -o wide
kubectl get svc -n envoy-gateway-system
kubectl get crd | grep -E 'gateway|envoy'
kubectl api-resources | grep gateway.networking.k8s.io
```

**STEP SUCCESSIVI:** `GatewayClass`, un `Gateway` e poi migrare un singolo servizio pilota da Ingress a `HTTPRoute`