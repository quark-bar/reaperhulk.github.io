---
author: admin
comments: true
date: 2009-03-14 15:26:13+00:00
layout: post
slug: checking-a-remote-certificate-chain-with-openssl
title: Checking A Remote Certificate Chain With OpenSSL
wordpress\_id: 398
tags:
- crypto
- openssl
- ssl
---

If you deal with SSL/TLS long enough you will run into situations where you need to examine what certificates are being presented by a server to the client.  The best way to examine the raw output is via (what else but) OpenSSL.[^1]

First let's do a standard webserver connection (-showcerts dumps the PEM encoded certificates themselves for more extensive parsing if you desire.  The output below snips them for readability.):

```bash
openssl s_client -showcerts -connect www.domain.com:443
```


```
CONNECTED(00000003)
--snip--
---
Certificate chain
 0 s:/C=US/ST=Texas/L=Carrollton/O=Woot Inc/CN=*.woot.com
   i:/C=US/O=SecureTrust Corporation/CN=SecureTrust CA
-----BEGIN CERTIFICATE-----
--snip--
-----END CERTIFICATE-----
 1 s:/C=US/O=SecureTrust Corporation/CN=SecureTrust CA
   i:/C=US/O=Entrust.net/OU=www.entrust.net/CPS incorp. by ref. (limits liab.)/OU=(c) 1999 Entrust.net Limited/CN=Entrust.net Secure Server Certification Authority
-----BEGIN CERTIFICATE-----
--snip--
-----END CERTIFICATE-----
---
Server certificate
subject=/C=US/ST=Texas/L=Carrollton/O=Woot Inc/CN=*.woot.com
issuer=/C=US/O=SecureTrust Corporation/CN=SecureTrust CA
---
No client certificate CA names sent
---
SSL handshake has read 2123 bytes and written 300 bytes
---
New, TLSv1/SSLv3, Cipher is RC4-MD5
Server public key is 1024 bit
--snip--
```

There's a lot of data here so I have truncated several sections to increase readability.  Points of interest:




  1. The certificate chain consists of two certificates.  At level 0 there is the server certificate with some parsed information.  s: is the subject line of the certificate and i: contains information about the issuing CA.


  2. This particular server (www.woot.com) has sent an intermediate certificate as well.  Subject and issuer information is provided for each certificate in the presented chain.  Chains can be much longer than 2 certificates in length.


  3. The server certificate section is a duplicate of level 0 in the chain.  If you're only looking for the end entity certificate then you can rapidly find it by looking for this section.


  4. No client certificate CAs were sent.  If the server was configured to potentially accept client certs the returned data would include a list of "acceptable client CAs".


  5. Connection was made via TLSv1/SSLv3  and the chosen cipher was RC4-MD5. Incidentally, this typically means that the server you're connecting to is IIS.


But what if you want to connect to something other than a bog standard webserver on port 443?  Well, if you need to use starttls that is also available.  As of OpenSSL 0.9.8 you can choose from smtp, pop3, imap, and ftp as starttls options.

```bash
openssl s_client -showcerts -starttls imap -connect mail.domain.com:139
```

If you need to check using a specific SSL version (perhaps to verify if that method is available) you can do that as well.  -ssl2, -ssl3, -tls1, and -dtls1 are all choices here.[^2]

```bash
openssl s_client -showcerts -ssl2 -connect www.domain.com:443
```

You can also present a client certificate if you are attempting to debug issues with a connection that requires one.[^3]

```bash
openssl s_client -showcerts -cert cert.cer -key cert.key -connect www.domain.com:443
```

And for those who really enjoy playing with SSL handshakes, you can even specify acceptable ciphers.[^4]
```bash
openssl s_client -showcerts -cipher DHE-RSA-AES256-SHA -connect www.domain.com:443
```

The cipher used above should work for almost any Apache server, but will fail on IIS since it doesn't support 256-bit AES encryption.

[^1]: The s\_client command we're using opens an interactive socket and does not automatically return to the shell prompt, so remember you will have to hit control-c or type something and hit return to terminate the process.

[^2]: This example shows an attempted SSLv2 only connection.  SSLv2 should be disabled on any web server you control.  It has a variety of flaws and has been superseded by SSLv3/TLSv1 for over a decade.

[^3]: This example expects the certificate and private key in PEM form.  You can provide them in DER if you add -certform DER and -keyform DER (OpenSSL 0.9.8 or newer only)

[^4]: A list of available ciphers can be found by typing "openssl ciphers", but there are also myriad ways to sort by type and strength.  See the [ciphers](http://www.openssl.org/docs/apps/ciphers.html) man page for more details.
