### Reference

* https://kubernetes.io/docs/reference/access-authn-authz/authentication/

#### Certificate Structure
* A public key certificate, digital certificate or identity certificate is an electronic document to prove the ownership of the public key.
* A public key certificate provides a safe way for an entity to pass on its public key to be used in asymmetric cryptography (PKI).
* Most common format is x509.
* Certificate has the information about
	1. Issuer
	2. Subject, and subject alternate names.
	3. Validity
	4. Subject's public key.
	5. Digital signature of the issuer that has verified the certificate's content.

#### Comparison of CSR and CRT
* CSR has all the information about the subject, including the public key, and a self signature.
* Certificate has the same information, as well as the issuer information, and the digital signature from the issuer.
* If the CSR is signed by the intermediate CA, then client will use the entire certificate bundle (from leaf upto root CA certificates) for certificate validation.

#### CSR signing and signature by CA
* The certificate signing request is used by the certificate authority (CA), and signed by its private key to generate a certificate.
* The certificate can be used by the subject to prove it's ownership of the public key to any party/client that trusts the signature of the CA. ie, both the server and client have to trust the root CA.
* The root CA can be a well known CA (like Digicert or Entrust), or in case of local communication between microservices, it can be a local CA (for example VECS/VMCA for vmware)

#### CA chain.
* Multiple certificates may be linked in a certificate chain. When a certificate chain is used, the first certificate is always that of the sender. The next is the certificate of the entity that issued the sender's certificate.
* If more certificates are in the chain, then each is that of the authority that issued the previous certificate. The final certificate in the chain is the certificate for a root CA.
* A root CA is a public Certificate Authority that is widely trusted. Information for several root CAs is typically stored in the client's Internet browser. This information includes the CA's public key. Well-known CAs include DigiCert, Entrust, and GlobalSign.

#### TLS
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

#### Generating certificates
* Generating self signed certificates

```sh
openssl genrsa -des3 -out server/server.key 1024 # Generate private key

Generate self signed certificate.
openssl req -new -key server/server.key -x509 -days 365 -out server/server.crt

Checking the self signed certificate:
openssl x509 -inform pem -noout -text -in server/server.crt
	Certificate:
	    Data:
	        Version: 1 (0x0)
	        Serial Number: 15988035885256798120 (0xdde0e9edecedefa8)
	    Signature Algorithm: sha256WithRSAEncryption
	        Issuer: C=In, ST=Karnataka, L=Bengaluru, O=VMware, OU=Wcp, CN=aditi/emailAddress=house@pres
	        Validity
	            Not Before: Jul 20 14:33:27 2021 GMT
	            Not After : Jul 20 14:33:27 2022 GMT
	        Subject: C=In, ST=Karnataka, L=Bengaluru, O=VMware, OU=Wcp, CN=aditi/emailAddress=house@pres
	        Subject Public Key Info:
	            Public Key Algorithm: rsaEncryption
```

* Signing a CSR

```sh

# Generate new key and new CSR

$ openssl req -new -newkey rsa:1024 -nodes -keyout leaf/leaf-ca.key -out leaf-ca.csr


# Check the CSR

$ openssl req -text -noout -verify -in  leaf/leaf-ca.csr
	verify OK
	Certificate Request:
	    Data:
	        Version: 0 (0x0)
	        Subject: C=IN, ST=KN, L=Bengalore, O=leaf-ca.net, OU=OU, CN=leaf-ca.net/emailAddress=nobody@leaf-ca.net
	        Subject Public Key Info:
	            Public Key Algorithm: rsaEncryption
	                Public-Key: (1024 bit)
	                Modulus:
	                    00:d7:69:e7:ea:78:91:bb:6a:39:f9:db:ec:61:fc:
	                    46:2a:fa:1c:25:79:45:8d:1e:c2:50:c2:1a:40:92:
	                    4b:c4:26:f2:6d:97:fa:b3:7d:17:24:46:b3:eb:c1:
	                    b1:09:e9:f1:e4:36:e4:4f:50:3e:52:8f:35:b3:3d:
	                    85:97:0f:28:60:b6:1a:bc:77:81:ab:41:8b:f5:00:
	                    e4:dc:1e:ed:82:a4:f6:8e:fa:22:28:07:04:06:81:
	                    47:77:a3:a2:b0:74:b8:d9:9d:8b:cb:15:b5:4e:70:
	                    66:a4:b9:11:9f:ee:9c:b4:46:56:fd:16:1f:e8:e7:
	                    9d:d3:f9:64:53:62:c4:e1:e7
	                Exponent: 65537 (0x10001)
	        Attributes:
	            challengePassword        :unable to print attribute
	    Signature Algorithm: sha256WithRSAEncryption
	         79:8d:f7:ff:13:9d:5a:85:2a:5a:f4:97:16:f6:54:00:8f:7f:
	         08:ba:a6:d1:be:75:d2:a9:80:15:ab:cc:04:dd:0b:c3:e1:f7:
	         71:34:45:36:59:45:26:42:27:a4:df:11:0f:ec:0c:e1:40:16:
	         8a:59:8f:6f:fb:2d:94:30:55:bc:b4:39:8b:e4:d2:8a:2a:e4:
	         25:39:25:50:22:a3:7a:ef:34:25:4c:a6:75:9b:de:4d:ea:2b:
	         65:11:f5:fc:16:b7:7d:47:f9:d1:6b:9b:3d:18:5d:da:e3:df:
	         c9:ff:07:6b:72:73:50:a7:69:f9:b9:7b:45:1c:9e:3a:02:95:
	         35:23

```

* Setup configs to sign certificates

```sh
$ cat ca.conf
	[ ca ]
	default_ca = ca_default
	[ ca_default ]
	dir = ./ca
	certs = $dir
	new_certs_dir = $dir/ca.db.certs
	database = $dir/ca.db.index
	serial = $dir/ca.db.serial
	RANDFILE = $dir/ca.db.rand
	certificate = $dir/ca.crt
	private_key = $dir/ca.key
	default_days = 365
	default_crl_days = 30
	default_md = md5
	preserve = no
	policy = generic_policy
	[ generic_policy ]
	countryName = optional
	stateOrProvinceName = optional
	localityName = optional
	organizationName = optional
	organizationalUnitName = optional
	commonName = optional
	emailAddress = optional
```

* Create the CSR and sign it

```sh
$ openssl ca -config ca.conf -out leaf/leaf-ca.crt -infiles leaf/leaf-ca.csr

$ openssl x509 -inform pem -noout -text -in leaf/leaf-ca.crt
	Certificate:
	    Data:
	        Version: 1 (0x0)
	        Serial Number: 4660 (0x1234)
	    Signature Algorithm: md5WithRSAEncryption
	        Issuer: C=IN, ST=Karnataka, L=Bengalore, O=root-cert.net, OU=OU, CN=root-cert.et/emailAddress=nobody@root-cert.net
	        Validity
	            Not Before: Jul  7 17:11:37 2021 GMT
	            Not After : Jul  7 17:11:37 2022 GMT
	        Subject: C=IN, ST=KN, L=Bengalore, O=leaf-ca.net, OU=OU, CN=leaf-ca.net/emailAddress=nobody@leaf-ca.net
	        Subject Public Key Info:
	            Public Key Algorithm: rsaEncryption
	                Public-Key: (1024 bit)
	                Modulus:
	                    00:d7:69:e7:ea:78:91:bb:6a:39:f9:db:ec:61:fc:
	                    46:2a:fa:1c:25:79:45:8d:1e:c2:50:c2:1a:40:92:
	                    4b:c4:26:f2:6d:97:fa:b3:7d:17:24:46:b3:eb:c1:
	                    b1:09:e9:f1:e4:36:e4:4f:50:3e:52:8f:35:b3:3d:
	                    85:97:0f:28:60:b6:1a:bc:77:81:ab:41:8b:f5:00:
	                    e4:dc:1e:ed:82:a4:f6:8e:fa:22:28:07:04:06:81:
	                    47:77:a3:a2:b0:74:b8:d9:9d:8b:cb:15:b5:4e:70:
	                    66:a4:b9:11:9f:ee:9c:b4:46:56:fd:16:1f:e8:e7:
	                    9d:d3:f9:64:53:62:c4:e1:e7
	                Exponent: 65537 (0x10001)
	    Signature Algorithm: md5WithRSAEncryption
	         3b:12:04:cc:27:b1:1e:8d:0a:a6:3d:af:2c:48:3d:21:6b:99:
	         4e:36:bd:c4:89:c9:e7:dd:da:55:6e:7e:a9:cb:c0:d0:a8:61:
	         5f:ee:e8:fc:78:fb:92:08:ea:25:3f:1b:fa:04:af:10:8a:8b:
	         16:74:ea:6b:11:25:ca:78:97:07:c4:a5:d4:9f:59:0b:e2:11:
	         f4:ea:30:14:22:1d:39:d7:e8:21:b1:38:1c:95:43:b1:36:1f:
	         9a:5c:6e:0f:11:04:7d:fb:c8:ff:00:91:5f:71:8e:d8:2f:e3:
	         dc:29:27:d2:ba:9e:38:c5:01:f9:8b:0b:04:2a:b6:8d:a2:0c:
	         42:fe
```

### VMware certificates
* Get the VMCA root certificate

```sh
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

```sh
$ /usr/lib/vmware-vmca/bin/certool --gencertfromcsr --csrfile new.csr --cert new.crt
```

* Replacing root certificate:

```sh
$ /usr/lib/vmware-vmca/bin/certificate-manager
```

#### curl and openssl commands.
* Command to verify ssl connection:

```sh
openssl s_client -connect <ip>:<port>
```

* Command to decode the pem-encoded certificate:

```sh
openssl x509 -inform pem -noout -text -in <file>
```

* Command to get the certificate information of a service account in kubernetes.

```sh
kubectl get secrets default-token-brbz6 -n vmware-system-nsop -o json | \
	   jq -r '.data."ca.crt"' | \
	   base64 -d | \
	   openssl x509 -inform pem -noout -text
```

* Validating a certificate chain

```sh
openssl crl2pkcs7 -nocrl -certfile /tmp/chain.crt | openssl pkcs7 \
-print_certs -text -noout
```
