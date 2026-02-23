# TLS Cert
## which ingress uses which secret:
```bash
kubectl get ingress -n nr-zions -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.tls[*].secretName}{"\n"}{end}'
```
## Get the date for expiring
```bash
kubectl get secret https-certs -n nr-zions -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```
## check on cluster level
```bash
for row in $(kubectl get ingress -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"|"}{.metadata.name}{"|"}{range .spec.tls[*]}{.secretName}{" "}{end}{"\n"}{end}'); do
  ns=$(echo $row | cut -d'|' -f1)
  ing=$(echo $row | cut -d'|' -f2)
  for sec in $(echo $row | cut -d'|' -f3); do
    if kubectl get secret "$sec" -n "$ns" >/dev/null 2>&1; then
      echo "=============================="
      echo "Namespace : $ns"
      echo "Ingress   : $ing"
      echo "Secret    : $sec"
      kubectl get secret "$sec" -n "$ns" -o jsonpath='{.data.tls\.crt}' | base64 -d | \
        openssl x509 -noout -subject -issuer -dates
    else
      echo "⚠️  Secret $sec not found in $ns"
    fi
  done
done | awk '!seen[$0]++'
```

## check on cluster level
```bash
for sec in $(kubectl get secret -A --field-selector type=kubernetes.io/tls -o jsonpath='{range .items[*]}{.metadata.namespace}{"|"}{.metadata.name}{"\n"}{end}'); do
  ns=$(echo $sec | cut -d'|' -f1)
  name=$(echo $sec | cut -d'|' -f2)
  end=$(kubectl get secret "$name" -n "$ns" -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -enddate | cut -d= -f2)
  if [ $(date -d "$end" +%s) -lt $(date -d "+30 days" +%s) ]; then
    echo "⚠️  $ns/$name expires on $end"
  fi
done
```
## Check Certificate wise
```bash
for secret in $(kubectl get secrets -n nr-entain-qa --field-selector type=kubernetes.io/tls -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); do
  echo "==== $secret ===="
  kubectl get secret $secret -n nr-entain-qa -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
  echo
done
```
