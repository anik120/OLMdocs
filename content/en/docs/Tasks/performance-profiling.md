---
title: "Performance Profiling Metrics"
weight: 10
description: >
  The goal of this document is to familiarize you with the steps to enable and review OLM's performance profiling metrics.
---

## Prerequisites 

- [go](https://golang.org/dl/)

## Background

OLM utilizes the [pprof package](https://golang.org/pkg/net/http/pprof/) from the standard go library to expose performance profiles for the OLM Operator, the Catalog Operator, and Registry Servers. Due to the sensitive nature of this data, OLM must be configured to use TLS Certificates before performance profiling can be enabled.

Requests against the performance profiling endpoint will be rejected unless the client certificate is validated by OLM. Unfortunately, Kubernetes does not provide a native way to prevent pods on cluster from iterating over the list of available ports and retrieving the data exposed. Without authenticating the requests, OLM could leak customer usage statistics on multitenant clusters.

This document will dive into the steps to [enable olm performance profiling](enable-performance-profiling) and retrieving pprof data from each component.

## Enabling Performance Profiling

### Creating a Certificate

A valid server certiciate must be created for each component before Performance Profiling can be enabled. If you are unfamiliar with certificate generation, I recomend using the [OpenSSL](https://www.openssl.org/) tool-kit and refer to the [request certificate](https://www.openssl.org/docs/manmaster/man1/openssl-req.html) documentation.

Once you have generated a private and public key, this data should be stored in a `TLS Secret`:

```bash
$ export PRIVATE_KEY_FILENAME=private.key # Replace with the name of the file that contains the private key you generated.
$ export PUBLIC_KEY_FILENAME=certificate.key # Replace with the name of the file that contains the public key you generated.

$ cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: olm-serving-secret
  namespace: olm
type: kubernetes.io/tls
data:
  tls.key: $(base64 $PRIVATE_KEY_FILENAME)
  tls.crt: $(base64 $PUBLIC_KEY_FILENAME)
EOF
```

### Retrieving the Performance Profile from the OLM Deployment

Patch the OLM Deployment's pod template to use the generated TLS secret:

- Defining a volume and volumeMount
- Adding the `client-ca`, `tls-key` and `tls-crt` arguments
- Replacing all mentions of port `8080` with `8443`
- Updating the `livenessProbe` and `readinessProbe` to use HTTPS as the scheme.

This can be done with the following commands:

```bash
$ export CERT_PATH=/etc/olm-serving-certs # Define where to mount the certs.
$ kubectl patch deployment olm-operator -n olm  --type json   -p='[
    # Mount the secret to the pod
    {"op": "add", "path": "/spec/template/spec/volumes", "value":[{"name": "olm-serving-cert", "secret": {"secretName": "olm-serving-cert"}}]},
    {"op": "add", "path": "/spec/template/spec/containers/0/volumeMounts", "value":[{"name": "olm-serving-cert", "mountPath": '$CERT_PATH'}]},
    
    # Add startup arguments
    {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"--client-ca"},
    {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"'$CERT_PATH'/tls.crt"},
    {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"--tls-key"},
    {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"'$CERT_PATH'/tls.key"},
    {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"--tls-cert"},
    {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"'$CERT_PATH'/tls.crt"},
    
    # Replace port 8080 with 8443
    {"op": "replace", "path": "/spec/template/spec/containers/0/ports/0", "value":{"containerPort": 8443}},
    {"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/httpGet/port", "value":8443},
    {"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/port", "value":8443},

    # Update livenessProbe and readinessProbe to use HTTPS
    {"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/scheme", "value":"HTTPS"},
    {"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/httpGet/scheme", "value":"HTTPS"},
]'
deployment.apps/olm-operator patched
```

You will need to be able to access OLM port, for dev purposes the following commands may prove useful:

```bash
$ kubectl port-forward deployment/olm-operator 8443:8443 -n olm
```

You can then curl the OLM `/debug/pprof` endpoint to retrieve default pprof profiles like so:

```bash
export BASE_URL=https://localhost:8443/debug/pprof
curl $BASE_URL/goroutine --cert certificate.crt --key private.key  --insecure -o goroutine
curl $BASE_URL/heap --cert certificate.crt --key private.key  --insecure -o heap
curl $BASE_URL/threadcreate --cert certificate.crt --key private.key  --insecure -o threadcreate
curl $BASE_URL/block --cert certificate.crt --key private.key  --insecure -o block
curl $BASE_URL/mutex --cert certificate.crt --key private.key  --insecure -o mutex
```
