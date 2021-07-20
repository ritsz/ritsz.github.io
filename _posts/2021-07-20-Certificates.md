### Reference

* https://kubernetes.io/docs/reference/access-authn-authz/authentication/

#### Certificate Structure:
* A public key certificate, digital certificate or identity certificate is an electronic document to prove the ownership of the public key.
* A public key certificate provides a safe way for an entity to pass on its public key to be used in asymmetric cryptography (PKI).
* Most common format is x509.
* Certificate has the information about
	1. Issuer
	2. Subject, and subject alternate names.
	3. Validity
	4. Subject's public key.
	5. Digital signature of the issuer that has verified the certificate's content.

#### comparison of csr and crt.
* CSR has all the information about the subject, including the public key, and a self signature.
* Certificate has the same information, as well as the issuer information, and the digital signature from the issuer.
* If the CSR is signed by the intermediate CA, then client will use the entire certificate bundle (from leaf upto root CA certificates) for certificate validation. 

#### csr signing and signature by CA.
* The certificate signing request is used by the certificate authority (CA), and signed by its private key to generate a certificate.
* The certificate can be used by the subject to prove it's ownership of the public key to any party/client that trusts the signature of the CA. ie, both the server and client have to trust the root CA.
* The root CA can be a well known CA (like Digicert or Entrust), or in case of local communication between microservices, it can be a local CA (for example VECS/VMCA for vmware)

#### CA chain.
* Multiple certificates may be linked in a certificate chain. When a certificate chain is used, the first certificate is always that of the sender. The next is the certificate of the entity that issued the sender's certificate. 
* If more certificates are in the chain, then each is that of the authority that issued the previous certificate. The final certificate in the chain is the certificate for a root CA. 
* A root CA is a public Certificate Authority that is widely trusted. Information for several root CAs is typically stored in the client's Internet browser. This information includes the CA's public key. Well-known CAs include DigiCert, Entrust, and GlobalSign.

### curl and openssl commands.
* Command to verify ssl connection:	
```
$ openssl s_client -connect <ip>:<port>
```
* Command to decode the pem-encoded certificate:
```
$ openssl x509 -inform pem -noout -text -in <file>
```
* Command to get the certificate information of a service account in kubernetes.
```
$ kubectl get secrets default-token-brbz6 -n vmware-system-nsop -o json | \
	   jq -r '.data."ca.crt"' | \
	   base64 -d | \
	   openssl x509 -inform pem -noout -text
```
* Validating a certificate chain
```
$ openssl crl2pkcs7 -nocrl -certfile /tmp/chain.crt | openssl pkcs7 \
-print_certs -text -noout
```

### VMware certificates
* Get the VMCA root certificate
```
root@wdc-10-191-178-31 [ ~ ]# dcli +show com vmware vcenter certificateauthority getroot getroot
	-----BEGIN CERTIFICATE-----
	MIIERzCCAy+gAwIBAgIJANtU/Cay6sPNMA0GCSqGSIb3DQEBCwUAMIGuMQswCQYD
	VQQDDAJDQTEXMBUGCgmSJomT8ixkARkWB3ZzcGhlcmUxFTATBgoJkiaJk/IsZAEZ
	FgVsb2NhbDELMAkGA1UEBhMCVVMxEzARBgNVBAgMCkNhbGlmb3JuaWExMDAuBgNV
	BAoMJ3dkYy0xMC0xOTEtMTc4LTMxLm5pbWJ1cy5lbmcudm13YXJlLmNvbTEbMBkG
	A1UECwwSVk13YXJlIEVuZ2luZWVyaW5nMB4XDTIxMDQyNTEyMDEzN1oXDTMxMDQy
	MzEyMDEzN1owga4xCzAJBgNVBAMMAkNBMRcwFQYKCZImiZPyLGQBGRYHdnNwaGVy
	ZTEVMBMGCgmSJomT8ixkARkWBWxvY2FsMQswCQYDVQQGEwJVUzETMBEGA1UECAwK
	Q2FsaWZvcm5pYTEwMC4GA1UECgwnd2RjLTEwLTE5MS0xNzgtMzEubmltYnVzLmVu
	Zy52bXdhcmUuY29tMRswGQYDVQQLDBJWTXdhcmUgRW5naW5lZXJpbmcwggEiMA0G
	CSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCvDhZhn5IcQ96u1DOY9JvCdI5OINBt
	pVubMHjiNr3ZEVshwDyQ08lph46aEq+oeQCVC4YtCxcnYD6biwtGXqy/nwhsowpP
	qoUBtfz+I07Rfds1Bx3yMQzQyBN/Z3+YICobGvt1zi62T+v1dR3ybxcsmTSrjcph
	tNPtl7RKEpyWkuoc0ynYx3SQSTnCEZn9mzlF9hTjwZFKRtxjNu4Irw4400ddD62T
	P5HUXgbWeyy3b8rQM9VUPsrHiUYmmZmUpkogu5+3m5KVQ7R9zFiKTdkKK90RG6EA
	Rbmqli08r58GUXi4R9VwWAAjNRGN562HWw/uYyfOrDa0uWYik4Rj9csBAgMBAAGj
	ZjBkMB0GA1UdDgQWBBSglHnWeUTJtbegFxF/5zJ8ZlYpaDAfBgNVHREEGDAWgQ5l
	bWFpbEBhY21lLmNvbYcEfwAAATAOBgNVHQ8BAf8EBAMCAQYwEgYDVR0TAQH/BAgw
	BgEB/wIBADANBgkqhkiG9w0BAQsFAAOCAQEASRSjW5Sa6NkdVrEmfEe6kNNakeuU
	0t0THE04fl3AUJMaTFlkDywFsFLVWZlL9WdWPxgVWRTokmfy8NzxFo4Dy7PolCr3
	eQfJ9pwxWZs66PPnbfnu/wO4FyGZ1ABqBM9kenOsjIF8ErhGgfjWqOCIdn6Jdv99
	qsR52Ctw7A6Exyno43k/0pPQJR8ghZcCvk71OIiic4gOjNIZZ/LfqyO+xilpz2UM
	oAdEe+KaEAkvQT1/VD9LzVw4UIenEQ5o075OQo3ZOwd1c+GtZsSqh09MJlyLpuNq
	XUGgGs+cR1zmKLcGvpOb5FcjXpfUOjykmlt2+SG/cTc3UZdnE2Cv+esgcg==
	-----END CERTIFICATE-----
	
$ dcli +show com vmware vcenter certificateauthority getroot getroot | openssl x509 -noout -text
	Signature Algorithm: sha256WithRSAEncryption                                                                                         
	   Issuer: CN=CA, DC=vsphere, DC=local, C=US, ST=California, O=wdc-10-191-178-31.nimbus.eng.vmware.com, OU=VMware Engineering
	   Validity
	       Not Before: Apr 25 12:01:37 2021 GMT
	       Not After : Apr 23 12:01:37 2031 GMT
	   Subject: CN=CA, DC=vsphere, DC=local, C=US, ST=California, O=wdc-10-191-178-31.nimbus.eng.vmware.com, OU=VMware Engineering
	   Subject Public Key Info:
```
* Signing a CSR on VC using VC root certificate
```
$ /usr/lib/vmware-vmca/bin/certool --gencertfromcsr --csrfile new.csr --cert new.crt
```
* Replacing root certificate:
```
$ /usr/lib/vmware-vmca/bin/certificate-manager
```

### TLS
* TLS gives us:
	* integrity
	* confidentialitty (encryption using symmetric key)
	* Authentication (using certificates)

* TCP Handshake done.
* Client sends ClientHello with list of supported cyphers.
* Server chooses a cypher suite and sends its certificate.
* Client validates the server certificate.
* Client performs *ClientKeyExchange*: 
	* Client has the signed public key but needs to verify that the server has the matching private key.
	* *RSA Handshake*:
		* Client sends a premaster secret encrypted by the public key.
		* Server decrypts the premaster secrted using it's private key from the certificate. Thus we can be sure that server actually has it's private key.
	* *Diffie Hellman Ephemeral (DHE) Handshake*:
		* Key agreement in plain-text.
		* Server sends DH parameters signed by it's private key. Thus we can be sure that server actually has it's private key if the client can verify the signature using the public key shared in the certificate.
* Client sends *CypherChangeSpec* (we are moving to symmetric encryption now.)
* Server confirms *CypherChangeSpec*.

### using openssl generated certificates, signed by K8s client CA, to use K8s API server. (TODO)
