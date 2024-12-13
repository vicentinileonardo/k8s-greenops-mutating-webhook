


TO BE DELETED


Source: https://www.openpolicyagent.org/docs/latest/kubernetes-tutorial/

```bash
kubectl create namespace opa
```

Label kube-system and the opa namespace so that OPA does not control the resources in those namespaces:
```bash
kubectl label ns kube-system openpolicyagent.org/webhook=ignore
kubectl label ns opa openpolicyagent.org/webhook=ignore
```

Communication between Kubernetes and OPA must be secured using TLS. 
To configure TLS, use `openssl` to create a certificate authority (CA) and certificate/key pair for OPA:
```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -sha256 -key ca.key -days 100000 -out ca.crt -subj "/CN=admission_ca"
```

Generate the TLS key and certificate for OPA:
```bash
cat >server.conf <<EOF
[ req ]
prompt = no
req_extensions = v3_ext
distinguished_name = dn

[ dn ]
CN = opa.opa.svc

[ v3_ext ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
subjectAltName = DNS:opa.opa.svc,DNS:opa.opa.svc.cluster,DNS:opa.opa.svc.cluster.local
EOF
```

```bash
openssl genrsa -out server.key 2048
openssl req -new -key server.key -sha256 -out server.csr -extensions v3_ext -config server.conf
openssl x509 -req -in server.csr -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 100000 -extensions v3_ext -extfile server.conf
```

Create a Secret to store the TLS credentials for OPA:
```bash
kubectl create secret tls opa-server --cert=server.crt --key=server.key --namespace opa
```

```bash
kubectl apply -f admission-controller.yaml
```


Generate the webhook configuration file:

```bash
cat > webhook-configuration.yaml <<EOF
kind: MutatingWebhookConfiguration
apiVersion: admissionregistration.k8s.io/v1
metadata:
  name: opa-mutating-webhook
webhooks:
  - name: mutating-webhook.openpolicyagent.org
    namespaceSelector:
      matchExpressions:
      - key: openpolicyagent.org/webhook
        operator: NotIn
        values:
        - ignore
    rules:
      - operations: ["CREATE"]
        apiGroups: ["composition.krateo.io"]  # Specify the correct API group
        apiVersions: ["v1-2-0"]  # Specify the correct version
        resources: ["vmtemplates"]  # Specify the correct resource
    clientConfig:
      caBundle: $(cat ./certs/ca.crt | base64 | tr -d '\n')
      service:
        namespace: opa
        name: opa
    admissionReviewVersions: ["v1"]
    sideEffects: None
EOF
```

# TODO

- helm chart
- in values.yaml, we can set bundle address, etc