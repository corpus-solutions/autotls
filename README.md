# Credit

This repository was forked [patrick0057/genericssl](https://github.com/patrick0057/genericssl) from [superseb/omgwtfssl](https://github.com/superseb/omgwtfssl) which was forked from [paulczar/omgwtfssl](https://github.com/paulczar/omgwtfssl). Paul Czarkowski is the original author of this project, SuperSeb and patrick0057 made some changes to the original project that are useful for my projects.

---

<details>
<summary>There are 2 reasons for forking</summary>

- to update guide to better usage of creating SSL for modern certificate requirents.
- remove the docker image dependency, and make it resilient to run on a sandbox enviroment.
</details>

---

`Subject Alt Names`(SAN) is require for modern certificate. And the shell code that create SAN in [generate-certs](generate-certs) was deprecated (see [SSL_DNS](generate-certs#line69) & [SSL_IP](generate-certs#line76)), and not longer support the creation as I need.

---

# GENERICSSL - Self Signed SSL Certificate Generator

## About

Sick of googling every time you need a self signed certificate?

GENERICSSL is a small (< 8 mb) docker image based off `alpine linux` which makes creating self signed SSL certs easier.

It will dump the certs it generators into `/certs` by default and will also output them to stdout in a standard
YAML form making them easy to consume in Ansible or other tools that use YAML.

<details>
<summary> Simple Example </summary>

```
docker build -f Dockerfile -t genericssl:v1
docker run -e SSL_SUBJECT="example.com" genericssl:v1
```

OUTPUT

```
----------------------------
| GENERICSSL Cert Generator |
----------------------------

--> Certificate Authority
====> Generating new CA key ca-key.pem
====> Generating new CA Certificate ca.pem
====> Generating new config file openssl.cnf
====> Generating new SSL KEY key.pem
====> Generating new SSL CSR key.csr
====> Generating new SSL CERT cert.pem
Certificate request self-signature ok
subject=CN = example.com
====> Complete
keys can be found in volume mapped to /certs

====> Output results as YAML
---
ca_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQCvRTHlSFboO0eU
  ...
  iappx6KZepVglqYXiFty9mxhxA==
  -----END PRIVATE KEY-----

ca_crt: |
  -----BEGIN CERTIFICATE-----
  MIIDBTCCAe2gAwIBAgIUQ7jgvp7gqzG2DZLlKjF4WzNf6BgwDQYJKoZIhvcNAQEL
  ...
  Pt8zgM55Gz9n
  -----END CERTIFICATE-----

ssl_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQC2RTGLjZyggz0V
  ...
  xjcg516qX8ZTk/FYXTCIaw==
  -----END PRIVATE KEY-----

ssl_csr: |
  -----BEGIN CERTIFICATE REQUEST-----
  MIICozCCAYsCAQAwFjEUMBIGA1UEAwwLZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3
  ...
  1FUQnn3eUQ==
  -----END CERTIFICATE REQUEST-----

ssl_crt: |
  -----BEGIN CERTIFICATE-----
  MIIDLzCCAhegAwIBAgIUY0xy6MP/GDcdpPFICtFUTnD3LFQwDQYJKoZIhvcNAQEL
  ...
  klKY
  -----END CERTIFICATE-----
```

</details>

## Advanced Usage

Customize the certs using the following Environment Variables:

- `CA_KEY` CA Key file, default `ca-key.pem` **[1]**
- `CA_CERT` CA Certificate file, default `ca.pem` **[1]**
- `CA_SUBJECT` CA Subject, default `test-ca`
- `CA_EXPIRE` CA Expiry, default `60` days
- `SSL_CONFIG` SSL Config, default `openssl.cnf` **[1]**
- `SSL_KEY` SSL Key file, default `key.pem`
- `SSL_CSR` SSL Cert Request file, default `key.csr`
- `SSL_CERT` SSL Cert file, default `cert.pem`
- `SSL_SIZE` SSL Cert size, default `2048` bits
- `SSL_EXPIRE` SSL Cert expiry, default `60` days
- `SSL_SUBJECT` SSL Subject default `example.com`
- `SSL_DNS` semicolon seperate list of alternative hostnames, no default **[2]**
- `SSL_IP` semicolon seperate list of alternative IPs, no default **[2]**

**[1] If file already exists will re-use.**

**[2] If `SSL_DNS` or `SSL_IP` is set will add `SSL_SUBJECT` to alternative hostname list."**

---

## Examples

---

<details>
<summary>Create Certificates for NGINX</summary>

_Creating web certs for testing SSL just got a hell of a lot easier..._

Create Certificate:

```
docker build -f Dockerfile -t genericssl:v1
docker run -v /tmp/certs:/certs \
  -e SSL_SUBJECT=test.example.com \
  -e SSL_IP=127.0.0.1 genericssl:v1
```

Enable SSL in `/etc/nginx/sites-enabled/default`:

```
server {
        listen 443;
        server_name test.example.com;
        root html;
        index index.html index.htm;
        ssl on;
        ssl_certificate /tmp/certs/cert.pem;
        ssl_certificate_key /tmp/certs/key.pem;
        ssl_session_timeout 5m;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;
        location / {
                try_files $uri $uri/ =404;
        }
}
```

Restart NGINX and test:

```
$ service nginx restart
$ echo '127.0.2.1       test.example.com' >> /etc/hosts
$ curl --cacert /tmp/certs/ca.pem https://test.example.com
<!DOCTYPE html>
<html>
<head>
...
```

</details>

---

<details>
<summary>Create keys for docker registry</summary>
_Slightly more interesting example of using `patrick0057/genericssl` as a volume container to build and host SSL certs for the Docker Registry image_

Create the volume container for the registry from `patrick0057/genericssl`:

```
docker build -f Dockerfile -t genericssl:v1
docker run --name certs \
  -e SSL_SUBJECT=test.example.com \
  -e SSL_IP=127.0.0.1 genericssl:v1
```

```
----------------------------
| GENERICSSL Cert Generator |
----------------------------
--> Certificate Authority
====> Generating new CA key ca-key.pem
Generating RSA private key, 2048 bit long modulus (2 primes)
ssl_crt: |
  -----BEGIN CERTIFICATE-----
  MIIDFzCCAf+gAwIBAgIUSS2jgxSefw0WuV4h+L7bdYHhslwwDQYJKoZIhvcNAQEL
  BQAwEjEQMA4GA1UEAwwHdGVzdC1jYTAeFw0yMzA4MTYwODEwMTVaFw0yMzEwMTUw
  ...
  v8UITaFhroNDfCp5U8BOhl+98PdP3wankyP7
  -----END CERTIFICATE-----
```

Run the registry using `--volumes-from` to use the volume container created above:

```
$ docker run -d \
    --name registry \
    --volumes-from certs \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/cert.pem \
    -e REGISTRY_HTTP_TLS_KEY=/certs/key.pem \
    -e SSL_IP=127.0.0.1 \
    -p 5000:5000 \
    registry:2
```

Make sure it works:

```
$ echo "127.0.0.1       test.example.com" >> /etc/hosts
$ docker tag patrick0057/genericssl test.example.com:5000/omgwtfbbq
$ docker push test.example.com:5000/omgwtfbbq
The push refers to a repository [test.example.com:5000/omgwtfbbq] (len: 1)
e34964fe7cfa: Pushed
d52b82eb9ff3: Pushed
6b030e7d76a6: Pushed
8a648f689ddb: Pushed
latest: digest: sha256:8a97202b0ad9b375ff478d84ed948ae7ddd298196fd3b341fc8391a0fe71345a size: 7617
```

</details>

---

<details>
<summary>Generate Keys for Kubernetes Secret for use with Ingress:</summary>

The following environment variables will help control your Kubernetes secret:

- `K8S_NAME` (genericssl)
- `K8S_NAMESPACE` (default)
- `K8S_SAVE_CA_KEY` (false)
- `K8S_SAVE_CA_CRT` (false)
- `SILENT` (true)

**Requirements** (push image to remote registry and change [username/genericssl:v1](examples/minikube/genericssl.yaml#line21)):

```
docker build -f Dockerfile -t genericssl:v1
docker tag genericssl:v1 username/genericssl:v1
docker push username/genericssl:v1
```

An example manifest can be found at `examples/minikube/genericssl.yaml`.

```
$ kubectl apply -f examples/minikube
configmap "genericssl" created
job "genericssl" created

$ kc get pods -a
NAME              READY     STATUS      RESTARTS   AGE
genericssl-blz7m   0/1       Completed   0          2m

$ kubectl logs genericssl-blz7m
secret "genericssl" created
kubectl get secret genericssl -o yaml

apiVersion: v1
kind: Secret
metadata:
  name: genericssl
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURHakNDQWdLZ0F3SUJBZ0lKQU5VWFdBaFJ4U0RqTUEwR0NTcUdTSWIzRFFFQkN3VUFNQkl4RURBT0JnTlYKQkFNTUIzUmxjM1F0WTJFd0hoY05NVGd3TXpBMU1qSXhOakF3V2hjTk1qZ3dNVEV5TWpJeE5qQXdXakFpTVNBdwpIZ1lEVlFRRERCY3FMakU1TWk0eE5qZ3VPVGt1TVRBd0xuaHBjQzVwYnpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCCkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUtldG5qcWVXY1liWktvQ0JVWHp5NWxQRGszRFo3S0R2cDZWWFBQTGhPTy8Ka2w3NDAwd2JjcGQ5aXdHNFY5elV3RkRiTG83dkFEMVVIMVRDL2lzWUtCZ1dvK2s1UEQwVzVWTlFiRnlmRUtzYQpaUjNycHFsMC9vR2M2eXdvWi9rUlVaZlF2M0s1TUV1WHQ1enA1LzVycllVdHFpVUxadTVlYjc3UW1pWUpOemMyCjFEWVYvaWNYc2pxTmhNL05ZQmtGZWpjd0lvUWp0YmZTejl5YldXS2VESURLQVY4a0RsN2pFVlNHVFRGYllwSGkKMnVjS1BpOEZxTGNRZEVCSDB5NnR5Q3N6ZW01cnhWM2VBeFMrN2EvZ1JaWXpZN2RwdDlia1R3M1AyRytKOWFidgpIa0FHbHN0Z01nY2l0VThZb0c4dmRNM01rRy9ReGl0Sk5xVDNwQVJFN0JNQ0F3RUFBYU5qTUdFd0NRWURWUjBUCkJBSXdBREFMQmdOVkhROEVCQU1DQmVBd0hRWURWUjBsQkJZd0ZBWUlLd1lCQlFVSEF3SUdDQ3NHQVFVRkJ3TUIKTUNnR0ExVWRFUVFoTUIrQ0Z5b3VNVGt5TGpFMk9DNDVPUzR4TURBdWVHbHdMbWx2aHdUQXFHTmtNQTBHQ1NxRwpTSWIzRFFFQkN3VUFBNElCQVFCQ1BOeEpHNUIrQjFiaEZ4U09oTHNVdVhSb0s1QldHeS94OGw4dDdGU2VWYjhuCmJmd2VSSnZZV0U0WDZkRnZJV2dOdDRyYTFodHhGc2k1b0Nad1FBaXd0U0xSMzEydU11d0RxN09VY256LzNXSUwKQlRaeEJIaGZCZkNCMnlJZEp6a3YrQUsrSTFsQitxY1VFKzd2bVJOb3EzK1BOeXk2SUNBQUlpbTc5RW1BcVVodwp6YnQ1YnhqYmlteFJUMWQ5dWdnTC9lM3NqZFpRM1VJTEZzNkdNemNkWXQ5WHBTdWdsb1RKTHZOTThCampXOTl5CkFxWUdWYkltc3craFdUZFhIVGljZXgwOVFCa3p4RUxzYmszb0hLOWFoV3VDSmU2ZDB3emJiTldEUWlJcEgvN1EKSDB0V2kzNTBYOVZBV2lqQXdSZ3UybG1lcVp6bVlwNk1ibWQzcjFuMQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcFFJQkFBS0NBUUVBcDYyZU9wNVp4aHRrcWdJRlJmUExtVThPVGNObnNvTytucFZjODh1RTQ3K1NYdmpUClRCdHlsMzJMQWJoWDNOVEFVTnN1anU4QVBWUWZWTUwrS3hnb0dCYWo2VGs4UFJibFUxQnNYSjhRcXhwbEhldW0KcVhUK2daenJMQ2huK1JGUmw5Qy9jcmt3UzVlM25Pbm4vbXV0aFMycUpRdG03bDV2dnRDYUpnazNOemJVTmhYKwpKeGV5T28yRXo4MWdHUVY2TnpBaWhDTzF0OUxQM0p0WllwNE1nTW9CWHlRT1h1TVJWSVpOTVZ0aWtlTGE1d28rCkx3V290eEIwUUVmVExxM0lLek42Ym12RlhkNERGTDd0citCRmxqTmp0Mm0zMXVSUERjL1liNG4xcHU4ZVFBYVcKeTJBeUJ5SzFUeGlnYnk5MHpjeVFiOURHSzBrMnBQZWtCRVRzRXdJREFRQUJBb0lCQVFDRnZleVVFdFBHT1BrOAp4T25SMXRnUlMwWThibHlhdll4Z1R3QmFFSDNKYm5iZ081WEZnYXNQKy9uUkFHbE1ZWUdYdkl0UlJINnJiQnFsCmIvWnRCeEtMekJzbkhoalhIUmtETUFXT2h1MHpuSlVFblg1TWNWM0NvaGZPRzloNmgvN05tWm5xZHAxMzNlWjkKU1BCYk5TV3RNVFFoNGd0U200NkQ0enpnazc4djBNZjAyMWh4UmFtRVRHM0MxTGd0Q3ZXY0dhWWh5TGlTNXhkKwpPeWVSb08zQXR0ZU5ObUszZkhIR2ZNSDRRR2FScnQzSktabUQrcnQ5NHQzbEhNY2YwOFV5d1A4Tjg0K2xOUlVLClpOZkd4bjNnYXZyNWJPT2twYUNvUTMxWnVPL3JFM0RzTGMwd25NTnVCMU9SeEJjR2tLYVZPVWI0YmhwVEE4dGwKR1MwejU2RHhBb0dCQU5oMzV0NGZzRjJxbVlwaGdxU2NvWEU0WUgzNVV2NzFVai9hWndDK0NlcythS2pHczRJSwpXeXpvQnIwR21Na20yQ3AreDkwbFhjano0WmFKUG1NWXdNcEVyQnluaTFVY3Z0Z08rb1VheTdWNGZWOUNsUUdLCnNENmlCUVFnSjloTEtZdkJnRDBkUjV2TWFWYTM2SFF3S3EzTjRHdzNIRFVwZ0NBeEFFenVWUDJaQW9HQkFNWk0KdXZCRVRBWU5odE5SdHFRUlgzUHhtTVJCWmt3YlZrK2ZFWjY1cWd4MUFGeTd4UURxNGxJanpKZHh5T3J5OWd0ZApxMWRLQkFhRzVrN0tWWWJzaXlSdUwxS0pwRlhKQm1WZURyaG14NEV0OWc3dlM0dmRRSklPQzlROWtMTS9TRE1lCnNjc0VqS243UXFVR0o0UHJwY0lIakY0ZW5lRE4xK3k2VzYwSitVcUxBb0dBSkhCS2xLbVE3ck9CRlNKRTg2REsKTEZ6cElVdVBCUXdXeEZqbmJlQ1BtdUh1akRxbWpRVmhRN1hyTEhhbjBYU1FmdGJJbmhsa0tDZWxtY21RanUzagp4aWk1TURtajRyZnNDRUs5T1JyQm45S2dpQ0NWSktWTDliOGdTUW1BcTVBN2RpTWtpeVVhb01kUUZDRHhLRjNUClVWNk9vS2pHUHN5MW5MV2k3MUJQVGtFQ2dZRUFzK1dpWmh5Zmw1SW44WWdkR0lVR1FvbzRYRHMwa2ZEdkFYYSsKcG0rclhIZThwMlJWV2ZxODdXVzYwdDJRTjgzSTl4QzRRNDFMVDV5TVRZaHp4TjdOY0hSaGpCQ0F2SzZObGVLWgptaUxyOVQ1OERwcDZ2OTB1R2hLU0dxN3JtaUhiM3p5R2NUYWtZZ1VuTmN6NmhreCs2U0t0N2lqNmM1cHF2RUZvCnIvZnZaL2NDZ1lFQXZGU29QZmhxaTkxOTFVWXA2K3R6YVBIQmcxeGRSWlVjQ2o2UFI4L0hQZVJhWWFCQkdMVysKdUVmWm1VbGhOUVkwblozT1p3aEw1VGlWNzUyTVFtdEMwQlVtSUhrQmg1MnEzcUorMW1qdi9wS0xZWkhxcnpSVwpXcHp6cTdlWFdwNVE2R0YwZTBQa0pFVzdMWmMwYjN3UmFMRTBhUStLeGc0ODZSUlFEVGFJQ0xjPQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

</details>
