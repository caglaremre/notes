# tls
## generating self-sign root certificate
- generate key
  - `openssl genrsa -out ca.key 4096`
- generate certificate signing request
  - `openssl req -new -key ca.key -shubj "/CN=kubernetes-ca" -out ca.csr`
- sign certificate request
  - `openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt`

## generating new certificate with self-sign root certificate
- generate key
  - `openssl genrsa -out admin.key 4096`
- generate certificate signing request
  - `openssl req -new -key admin.key -subj "/CN=kube-admin" -out admin.csr`
- sign certificate request with ca key
  - `openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt`

## view certificate details
`openssl x509 -in <certificate file> -text -noout`