# Dapr app API token for workshop namespaces

The `app-api-token` Secret protects participant `/internal/*` endpoints from direct requests. Dapr sidecars add the token as the `dapr-api-token` header when they forward service-invocation requests to an application.

The token must not be committed to this public repository. Create it once in the `default` namespace and let Reflector copy it to every `workshop-*` namespace.

```sh
TOKEN=$(openssl rand -hex 32)
kubectl create secret generic app-api-token --namespace default --from-literal=token="$TOKEN"
kubectl annotate secret app-api-token --namespace default reflector.v1.k8s.emberstack.com/reflection-allowed="true" reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces="workshop-.*" reflector.v1.k8s.emberstack.com/reflection-auto-enabled="true" reflector.v1.k8s.emberstack.com/reflection-auto-namespaces="workshop-.*"
```

Verify replication after a workshop namespace exists:

```sh
kubectl get secret app-api-token --namespace workshop-thomas
```

The workshop Helm chart expects the replicated Secret to be named `app-api-token` and to contain the `token` key. Rotate the token by updating the source Secret, then restart participant Deployments so Dapr sidecars load the new value.
