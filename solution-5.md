# TLS Debugging Challenge – Root Cause Analysis & Resolution

## Objective

Fix the broken TLS communication between the `tls-client` pod and the `secure-app` service running inside the Kubernetes cluster.

---

# Initial Problem

When accessing the application using:

```bash
curl -v https://secure-app.t5.svc.cluster.local --cacert /etc/ssl/custom/ca.crt
```

the request failed with:

```text
SSL: certificate subject name 'wrong-hostname.example.com'
does not match target hostname
'secure-app.t5.svc.cluster.local'
```

Later, after regenerating the certificate, another error appeared:

```text
SSL certificate OpenSSL verify result:
certificate signature failure (7)
```

---

# Root Cause Analysis

## Root Cause 1 — Incorrect Certificate Common Name (CN)

The server certificate mounted inside the `secure-app` pod had:

```text
CN=wrong-hostname.example.com
```

but the application was being accessed through:

```text
secure-app.t5.svc.cluster.local
```

TLS hostname verification failed because the certificate hostname did not match the service DNS name.

---

## Root Cause 2 — Incorrect CA Bundle in Client Pod

The `tls-client` pod was initially using an invalid CA certificate bundle.

The mounted file inside the pod contained a base64-encoded certificate instead of the actual decoded CA certificate.

Because of this, OpenSSL could not validate the server certificate signature.

---

# Troubleshooting Steps Performed

## 1. Verified Existing Server Certificate

Checked the mounted certificate inside the application pod:

```bash
kubectl exec -it deploy/secure-app -n t5 -- \
openssl x509 -in /etc/nginx/ssl/tls.crt -noout -subject
```

Observed incorrect CN:

```text
subject=CN = wrong-hostname.example.com
```

---

## 2. Regenerated Server Certificate with Correct CN

Created a new certificate signing request using:

```text
CN=secure-app.t5.svc.cluster.local
```

Signed the certificate using the internal CA:

```bash
openssl x509 -req \
  -in /tmp/sanjay-server.csr \
  -CA /tmp/sanjay-ca.crt \
  -CAkey /tmp/sanjay-ca.key \
  -CAcreateserial \
  -out /tmp/sanjay-server.crt \
  -days 365
```

Verified the new certificate:

```bash
openssl x509 -in /tmp/sanjay-server.crt -noout -subject
```

Output:

```text
subject=CN = secure-app.t5.svc.cluster.local
```

---

## 3. Updated Kubernetes TLS Secret

Deleted old secret:

```bash
kubectl delete secret tls-secret -n t5
```

Created new secret:

```bash
kubectl create secret generic tls-secret -n t5 \
  --from-file=tls.crt=/tmp/sanjay-server.crt \
  --from-file=tls.key=/tmp/sanjay-server.key
```

---

## 4. Restarted Application Deployment

```bash
kubectl rollout restart deployment secure-app -n t5
```

Also manually deleted old pods to ensure the new certificate was loaded:

```bash
kubectl delete pod -n t5 -l app=secure-app
```

---

## 5. Verified Mounted Certificate Inside Pod

```bash
kubectl exec -it deploy/secure-app -n t5 -- \
openssl x509 -in /etc/nginx/ssl/tls.crt -noout -subject
```

Verified:

```text
subject=CN = secure-app.t5.svc.cluster.local
```

---

## 6. Fixed CA Bundle in Client Pod

Deleted incorrect ConfigMap:

```bash
kubectl delete configmap ca-bundle -n t5
```

Created a new ConfigMap using the actual CA certificate:

```bash
kubectl create configmap ca-bundle -n t5 \
  --from-file=ca.crt=/tmp/sanjay-ca.crt
```

---

## 7. Recreated Client Pod

Deleted old pod:

```bash
kubectl delete pod tls-client -n t5
```

Recreated the pod with the updated CA bundle mounted.

---

# Final Verification

Executed:

```bash
kubectl exec tls-client -n t5 -- \
curl -v https://secure-app.t5.svc.cluster.local \
--cacert /etc/ssl/custom/ca.crt
```

Successful output:

```text
common name: secure-app.t5.svc.cluster.local (matched)

OpenSSL verify result: 0

SSL certificate verified via OpenSSL.

HTTP/1.1 200 OK

TLS Challenge Complete!
```

---

# Final Result

The TLS issue was successfully resolved by:

1. Regenerating the server certificate with the correct hostname
2. Updating the Kubernetes TLS secret
3. Restarting the application deployment
4. Replacing the invalid CA bundle
5. Recreating the client pod

End-to-end TLS communication between the client and application now works successfully.
