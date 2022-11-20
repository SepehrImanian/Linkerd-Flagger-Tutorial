###tus mtls

**mutual authentication**

The root certificate is self-signed, meaning that the organization creates it themselves. 
(This approach does not work for one-way TLS on the public Internet because an external certificate authority has to issue those certificates.)

mTLS prevents various kinds of attacks, including:
* On-path attacks
* Spoofing attacks
* Credential stuffing
* Brute force attacks
* Phishing attacks
* Malicious API requests

![mtls](./image/how_mtls_works-what_is_mutual_tls.png)
