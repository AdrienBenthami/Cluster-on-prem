# cert-manager — tests

Manifests **non déployés en permanence**. Utilisés ponctuellement pour valider une chaîne d'émission ACME (issuer, solver DNS-01, API DNS, propagation, signature Let's Encrypt) sans toucher aux services applicatifs.

Convention :

- Toujours ciblés sur l'environnement **staging** de Let's Encrypt (certificats non reconnus par les navigateurs — c'est attendu).
- FQDN dédié `test-*.benthami.eu` — jamais un domaine de service réel.
- Namespace `test` — namespace dédié aux essais, pas de collision avec un namespace applicatif.
- À **supprimer immédiatement après validation** (voir section *Nettoyage* ci-dessous).

## Manifests disponibles

| Fichier | Issuer testé | FQDN | Objet |
|---|---|---|---|
| [test-certificate-staging-cloudflare.yaml](test-certificate-staging-cloudflare.yaml) | `letsencrypt-staging-cloudflare` | `test-cf.benthami.eu` | Valide la chaîne cert-manager → API Cloudflare → Let's Encrypt staging |

## Utilisation

```bash
# Appliquer
kubectl apply -f services/cert-manager/tests/test-certificate-staging-cloudflare.yaml

# Suivre l'émission (1 à 3 min)
kubectl -n test get certificate test-cloudflare-staging -w

# Vérifier l'issuer du cert émis (doit mentionner "STAGING")
kubectl -n test get secret test-cloudflare-staging-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -noout -subject -issuer -dates
```

Pendant l'émission, un enregistrement TXT `_acme-challenge.test-cf` apparaît temporairement dans la zone Cloudflare puis est nettoyé automatiquement par cert-manager.

## Nettoyage

```bash
kubectl -n test delete certificate test-cloudflare-staging
kubectl -n test delete secret test-cloudflare-staging-tls
```

Les ressources `Order` et `Challenge` associées disparaissent via finalizers.
