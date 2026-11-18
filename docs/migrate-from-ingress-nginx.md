# Migrate from ingress-nginx

Use this guide to switch an existing upstream `ingress-nginx` Helm release to the OSShield chart. Upgrade with the same release name and namespace so Helm can update the existing controller and Service. Keep the IngressClass name and controller value the same so your existing Ingress resources continue to select the controller. Plan a maintenance window and try the upgrade in staging first.

These steps assume Helm owns the existing release. If you installed ingress-nginx with raw manifests, plan a separate cutover with its own IngressClass, Service, and traffic switch. If Flux or Argo CD manages the release, make the chart source and values changes in Git and let your reconciler apply them in place.

## 1. Record the current release

Check your cluster context and find the release. Substitute its name and namespace below.

```sh
kubectl config current-context
helm list --all-namespaces

RELEASE=ingress-nginx
NAMESPACE=ingress-nginx

helm history "$RELEASE" --namespace "$NAMESPACE"
helm get values "$RELEASE" --namespace "$NAMESPACE" -o yaml > ingress-nginx-current.yaml
helm get manifest "$RELEASE" --namespace "$NAMESPACE" > ingress-nginx-current-manifest.yaml
kubectl get services --namespace "$NAMESPACE" -o wide
kubectl get ingressclass
kubectl get ingress --all-namespaces
```

Record the last working Helm revision and the controller Service's external address or hostname. Store the exported files securely because they can contain sensitive values. If you use multiple ingress controllers, identify the Ingresses served by this release before proceeding.

## 2. Download the OSShield chart and review your values

```sh
helm repo add osshield https://helm.osshield.eu
helm repo update osshield
helm search repo osshield/ingress-nginx --versions
```

Select a chart version from the search results. Pin it for both the preview and the upgrade. The chart version and the controller image tag are separate.

```sh
CHART=osshield/ingress-nginx
CHART_VERSION='<chart version from search>'

helm show chart "$CHART" --version "$CHART_VERSION"
helm show values "$CHART" --version "$CHART_VERSION" > ingress-nginx-osshield-defaults.yaml
cp ingress-nginx-current.yaml ingress-nginx-migration.yaml
```

If the Helm repository is unavailable, download and extract the `ingress-nginx-chart` artifact from a successful **Build and publish images and chart** workflow run. Set `CHART` to the extracted `.tgz` path and omit `--version` in the commands that use it. [Build workflow](../.github/workflows/build-all.yaml)

Review `ingress-nginx-migration.yaml` against the new defaults and the [chart changelog](../charts/ingress-nginx/changelog). In particular:

- Remove any upstream `controller.image` overrides for `repository`, `registry`, `image`, `tag`, `digest`, and `digestChroot`. The packaged defaults select `cr.osshield.eu/nginx/controller` and pin its digest. The controller image already includes the NGINX proxy.
- If `controller.image.chroot` is enabled, resolve that requirement before upgrading. The OSShield build workflow publishes the standard controller image. Set `chroot: false` only if the standard image meets your deployment requirements.
- Preserve the IngressClass and controller settings, release naming overrides, controller workload kind, and Service settings. Pay particular attention to load balancer annotations, static IP settings, ports, and `externalTrafficPolicy`.
- Preserve custom settings for controller ConfigMaps, TCP/UDP services, TLS, and admission webhooks. Rebuild any third-party NGINX dynamic modules against the fork's patched headers before using them. [NGINX module compatibility](../README.md#nginx-source-patches-and-abi)
- Ensure cluster nodes can pull from `cr.osshield.eu`. If your registry access requires credentials, reference a pull secret in the release namespace using the chart's `imagePullSecrets` value.

## 3. Preview and upgrade the existing release

```sh
helm upgrade "$RELEASE" "$CHART" --namespace "$NAMESPACE" \
  --version "$CHART_VERSION" --reset-values -f ingress-nginx-migration.yaml \
  --dry-run=server --hide-secret
```

Check the rendered controller image and compare the controller workload, Service, IngressClass, and selectors with the saved manifest. Resolve any unintended name, selector, or load balancer changes before applying. `--reset-values` starts from the OSShield chart defaults and applies your reviewed overrides. [Helm upgrade reference](https://helm.sh/docs/helm/helm_upgrade/)

```sh
helm upgrade "$RELEASE" "$CHART" --namespace "$NAMESPACE" \
  --version "$CHART_VERSION" --reset-values -f ingress-nginx-migration.yaml \
  --wait --timeout 10m
```

## 4. Verify traffic and roll back if needed

```sh
helm status "$RELEASE" --namespace "$NAMESPACE"
kubectl get pods --namespace "$NAMESPACE" \
  -l "app.kubernetes.io/instance=$RELEASE,app.kubernetes.io/component=controller" \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,IMAGE:.spec.containers[*].image'
kubectl get services --namespace "$NAMESPACE" -o wide
```

Confirm that the controller pods are ready and use `cr.osshield.eu/nginx/controller`, and that the Service still has the expected address. Send requests to representative Ingress hosts from a client outside the cluster. Check routing and TLS, plus any authentication, rewrites, or TCP/UDP services you use. If admission webhooks are enabled, confirm that a server-side dry run of a known-good Ingress manifest succeeds.

If the rollout fails or traffic regresses, roll back to the revision recorded in step 1 and repeat the traffic checks. For GitOps, revert the chart source and values in Git so the reconciler restores the previous configuration.

```sh
helm history "$RELEASE" --namespace "$NAMESPACE"
PREVIOUS_REVISION='<last working revision recorded before the upgrade>'
helm rollback "$RELEASE" "$PREVIOUS_REVISION" --namespace "$NAMESPACE" \
  --wait --timeout 10m
```

Rollback requires the previous controller image to remain pullable. [Helm rollback reference](https://helm.sh/docs/helm/helm_rollback/)
