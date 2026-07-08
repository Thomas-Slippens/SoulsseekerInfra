# Gateway API Migration Plan

## Background

ingress-nginx was announced as retiring in November 2025
([blog post](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)).
The Kubernetes project is moving to **Gateway API** as the official successor.
ingress-nginx will continue receiving security patches until at least end of 2026,
so this migration is planned but not urgent.

## What Changes

| Current | Replacement |
|---|---|
| `kind: Ingress` | `kind: HTTPRoute` |
| `ingressClassName: nginx` | `parentRefs` pointing at a `Gateway` |
| ingress-nginx Helm chart | nginx Gateway Fabric (or AGIC) Helm chart |
| Per-ingress TLS annotations | TLS configured once on the `Gateway` |
| `nginx.ingress.kubernetes.io/*` annotations | Native HTTPRoute filters or separate policy resources |

## Files Affected

### SoulsseekerInfra repo

| File | Change |
|---|---|
| `platform/bootstrap/ingress-nginx-application.yaml` | Replace ingress-nginx chart with nginx Gateway Fabric chart |
| `platform/argocd/argocd-ingress.yaml` | Replace `kind: Ingress` with `kind: HTTPRoute` |
| `helm/combat-analysis/templates/ingress.yaml` | Replace with `httproute.yaml` |
| `helm/feedback/templates/ingress.yaml` | Replace with `httproute.yaml` |
| `helm/combat-analysis/values.yaml` | Update ingress values shape |
| `helm/feedback/values.yaml` | Update ingress values shape |

### KubernernetesWorkshop repo

| File | Change |
|---|---|
| `helm/templates/ingress.yaml` | Replace with `httproute.yaml` |
| `helm/values.yaml` | Update ingress values shape |

## New Resources to Create

```
platform/gateway/
├── gatewayclass.yaml       # defines which controller handles Gateway resources
├── gateway.yaml            # main listener — port 443, TLS, owned by platform team
└── workshop-gateway.yaml   # separate gateway for workshop-* namespaces (optional)
```

## Known Blockers to Resolve Before Migrating

### 1. Rate limiting
Currently using `nginx.ingress.kubernetes.io/limit-rpm` annotations on combat-analysis
and feedback ingresses. Gateway API has no standard rate limiting spec yet.
Options:
- Use nginx Gateway Fabric policy extensions (vendor-specific)
- Add a dedicated rate limiting sidecar (e.g. Envoy, Traefik middleware)
- Switch to AGIC which has its own WAF/rate limit policy model via Azure

### 2. Path rewriting with regex
`/analysis(/|$)(.*)` → `/$2` rewrite is simple in ingress-nginx but requires
`URLRewrite` filters in HTTPRoute which are more verbose. Needs testing.

### 3. Workshop participant template
Participants fork the workshop repo. Template must be updated before any workshop
that runs post-migration, otherwise `kind: Ingress` in their fork won't work.

### 4. cert-manager integration
cert-manager supports Gateway API via the `cert-manager.io/gateway` annotation
but requires cert-manager v1.14+. Verify version before migrating.
```sh
kubectl get deployment cert-manager -n cert-manager -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## Migration Steps

1. **Install Gateway API CRDs** (cluster-scoped, one-time)
   ```sh
   kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
   ```

2. **Install nginx Gateway Fabric** as the new GatewayClass controller
   (replaces ingress-nginx Helm chart in `platform/bootstrap/ingress-nginx-application.yaml`)

3. **Create `platform/gateway/gatewayclass.yaml`**
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: GatewayClass
   metadata:
	 name: nginx
   spec:
	 controllerName: gateway.nginx.org/nginx-gateway-controller
   ```

4. **Create `platform/gateway/gateway.yaml`**
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: Gateway
   metadata:
	 name: main-gateway
	 namespace: gateway
	 annotations:
	   cert-manager.io/cluster-issuer: letsencrypt-prod
   spec:
	 gatewayClassName: nginx
	 listeners:
	   - name: https
		 port: 443
		 protocol: HTTPS
		 hostname: "*.soulsseeker.com"
		 tls:
		   mode: Terminate
		   certificateRefs:
			 - name: main-tls
   ```

5. **Replace `helm/combat-analysis/templates/ingress.yaml`** with `httproute.yaml`
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
	 name: combat-analysis
   spec:
	 parentRefs:
	   - name: main-gateway
		 namespace: gateway
	 hostnames:
	   - kubernetes.soulsseeker.com
	 rules:
	   - matches:
		   - path:
			   type: PathPrefix
			   value: /analysis
		 filters:
		   - type: URLRewrite
			 urlRewrite:
			   path:
				 type: ReplacePrefixMatch
				 replacePrefixMatch: /
		 backendRefs:
		   - name: combat-analysis
			 port: 80
   ```

6. **Repeat step 5** for feedback and argocd ingresses

7. **Update workshop template repo** — replace `helm/templates/ingress.yaml`
   with `httproute.yaml` pointing at a workshop-scoped Gateway

8. **Resolve rate limiting** — decide on approach (see blockers above)
   and implement before removing ingress-nginx

9. **Run `ingress2gateway` tool** as a sanity check
   ```sh
   # converts existing Ingress objects to HTTPRoute equivalents
   ingress2gateway print
   ```

10. **Test in a staging namespace** before cutting over production traffic

11. **Remove ingress-nginx Application** from `platform/bootstrap/` once
	all HTTPRoutes are verified healthy

## Useful References

- [Gateway API docs](https://gateway-api.sigs.k8s.io/)
- [nginx Gateway Fabric](https://github.com/nginxinc/nginx-gateway-fabric)
- [ingress2gateway migration tool](https://github.com/kubernetes-sigs/ingress2gateway)
- [cert-manager Gateway API support](https://cert-manager.io/docs/usage/gateway/)
- [ingress-nginx retirement announcement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
