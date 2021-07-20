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

