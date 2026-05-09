# CERTIFICATES

## Upload a TLS certificate

```bash
curl -X POST http://kong:8001/certificates \
  --form cert=@server.crt \
  --form key=@server.key \
  --form "snis[]=api.example.com" \
  --form "snis[]=*.example.com"
```

## Upload a CA certificate (for mTLS upstreams)

```bash
curl -X POST http://kong:8001/ca_certificates \
  --data cert=@ca.pem
```

## In YAML (decK)

```yaml
certificates:
  - cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----
    snis:
      - api.example.com

ca_certificates:
  - cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
```

> Never commit private keys to Git. Use environment variable substitution or a vault reference.

## Check expiry on a live endpoint

```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null \
  | openssl x509 -noout -dates
```

## Zero-downtime rotation

```bash
# 1. Upload new cert — Kong stores both old and new
NEW_ID=$(curl -s -X POST http://kong:8001/certificates \
  --form cert=@new.crt --form key=@new.key | jq -r '.id')

# 2. Point SNI to the new cert
curl -X PATCH http://kong:8001/snis/api.example.com \
  --data certificate.id=$NEW_ID

# 3. Verify
echo | openssl s_client -connect api.example.com:443 2>/dev/null \
  | openssl x509 -noout -dates

# 4. Delete old cert
curl -X DELETE http://kong:8001/certificates/<old-id>
```

## List certs and SNIs

```bash
curl http://kong:8001/certificates | jq '.data[] | {id, snis}'
curl http://kong:8001/snis | jq '.data[] | {name, certificate_id}'
```
