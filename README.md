<!-- *********************************************************************** -->
<!--                                                                         -->
<!--                                +###****.                                -->
<!--                                =***@@@+                                 -->
<!--            *%*   -%%:  -*%%%#     :@@@=:   #%##%%#=-*%%%*:              -->
<!--            %@%   =@@: #@@*=+*.    -==*@@+  @@@*=+@@@%+=#@@=             -->
<!--            %@%   -@@:-@@-     .==.    *@@. %@#   =@@:   %@#             -->
<!--            +@@+-=%@@..%@@+--+..%@@+--*@@*  @@#   +@@:   @@#             -->
<!--             -*%@@#+.   =#%@@%.  -*%@@%*-   %@*   =@@:   %@*             -->
<!--                                                                         -->
<!-- README.md                                                               -->
<!--                                                                         -->
<!-- By: aperez-b <100429952@alumnos.uc3m.es>                                -->
<!--                                                                         -->
<!-- Created: 2022/03/07 09:50:11 by aperez-b                                -->
<!-- Updated: 2022/03/07 14:11:17 by aperez-b                                -->
<!--                                                                         -->
<!-- *********************************************************************** -->

# OpenSSL Practices 2022 - Data Protection & Cybersecurity

*Welcome to the Open Secure Sockets layer 🔑*

<p align="center">
  <a href="https://www.uc3m.es/Home">
    <img src="https://user-images.githubusercontent.com/40824677/157040289-6bddd590-ba4a-4578-a1b9-6a35be9ff7b0.png">
  </a>
</p>

### Table of Contents

- [Introduction](#introduction)
- []

## Introduction

The objective of this practice is to understand the concepts underlying a public key infrastructure
based on the hierarchical trust model.
Specifically, the objectives are the following:

- Understand required steps in order to create a mini PKI
- Understand required steps in order to an Authority issuing a certificate
- Understand what the certificate role is regarding the signing and verifying of documents

To achieve these objectives each student (or student group) becomes a ROOT CERTIFICATION AUTHORITY (equal to the role of **Fábrica Nacional de Moneda y Timbre** in the real world). Such Authority (AC1), according to organizational reasons (for example, to have a local office in all different districts), has some SUBORDINATE CERTIFICATION AUTHORITIES (AC2, AC3,…ACn). Moreover, these last Authorities are in charge of issuing public key certificates to people (A, B, C…).

The group of all the Authorities composes a Public Key Infrastructure (PKI).

## Configuration of AC1 (root AC)

We will apply the following commands: ``ca``, ``req``, ``x509`` and ``verify``.

The ``ca`` command is a minimal Certification Authority application. It can be used to sign certificate requests in a variety of forms. Its characteristics can be established by means of the file ``openssl.cnf``. Moreover, we must prepare the working environment as a specific directory structure.

The ``req`` command primarily creates and processes certificate requests. It can additionally create self signed certificates for use as root CAs for example.

Command ``x509`` can be used to display certificate information, convert certificates to various forms, and sign certificate requests.
Verify command verifies certificate chains.

1. Generate the directory structure necessary for AC1 and initialize the files ``serial`` and ``index.txt``.

```shell
cd AC1
mkdir requests crls newcerts private
echo '01' > serial
touch index.txt
cd ..
```

2. Generate the RSA key-pair and the self signed certificate for AC1. Analyze the output.

```shell
cd AC1
openssl req -x509 -newkey rsa:2048 -days 360 -out ac1cert.pem -outform PEM -config openssl_AC1.cnf
cd ..
```

The command requests a passphrase to generate AC1 private key. Remember it when
you want to use this key.

```shell
cd AC1
openssl x509 -in ac1cert.pem -text -noout
cd ..
```

## Configuration of AC2 (subordinate AC)

3. Generate the directory structure necessary for AC2 and initialize the files ``serial`` and ``index.txt``.

```shell
cd AC2
mkdir requests crls newcerts private
echo '01' > serial
touch index.txt
cd ..
```

4. Generate the RSA key-pair for AC2 and the certificate request which will be sent to AC1 and 'send' it to AC1. Analyze the results.

```shell
cd AC2
openssl req -newkey rsa:2048 -days 360 -out ac2req.pem -outform PEM -config openssl_AC2.cnf
cd ..
```

The command requests a passphrase to generate AC2 private key. Remember it when you want to use this key.

```shell
cd AC2
openssl req -in ac2req.pem -text -noout
cp ac2req.pem ../AC1/requests
cd ..
```

## Generation of AC2’s certificate by AC1

5. Verify the request **sent** by AC2.

```shell
cd AC1
openssl req -in ./requests/ac2req.pem -text -noout
cd ..
```

6. Generate the corresponding certificate for AC2 and **send** it back to AC2. Rename ``01.pem`` into ``ac2cert.pem``, because AC2 has this name in its configuration file.

```shell
cd AC1
openssl ca -in ./requests/ac2req.pem -notext -extensions v3_subca -config openssl_AC1.cnf
cd ..
```

AC1 needs its private key to generate AC2 certificate. AC1 passphrase is requested

```shell
cd AC1
cp ./newcerts/01.pem ../AC2/ac2cert.pem
cd ..
```

## Generation of keys for entity A as well as its corresponding certificate request

7. For entity A, generate an RSA key-pair as well as a certificate request and **send** it to AC2 (when generating the certificate requests, fill in ALL the requested fields and indicate ``ES`` as country, ``MADRID`` as province, **UC3M** as organization, and any common name (eg, one NIA) and the email is your student email.

```shell
cd A
openssl req -newkey rsa:1024 -days 360 -sha1 -keyout Akey.pem -out Areq.pem
cd ..
```

The command requests a passphrase to generate A private key. Remember it when you want to use this key.

```shell
cd A
openssl req -in Areq.pem -text -noout
cp Areq.pem ../AC2requests/
cd ..
```

## Generation of A certificate by AC2

8. Verify the request **sent** by A.

```shell
cd AC2
openssl req -in ./requests/Areq.pem -text -noout
cd ..
```

9. Generate certificate for A and **send** it back to this entity.

```shell
cd AC2
openssl ca -in ./requests/Areq.pem -notext -config ./openssl_AC2.cnf
cd ..
```

AC2 needs its private key to generate A certificate. AC2 passphrase is request

```shell
cd AC2
cp ./newcerts/01.pem ../A/Acert.pem
cd ..
```

10. Analyze changes in AC2 directory and check the resulting certificate:

```shell
cd A
openssl x509 -in Acert.pem -text -noout
cd ..
```

## Verification of A certificate

11. Obtain a copy of the public key certificates of AC1 and AC2 and verify (you need to concatenate both AC1 and AC2 certificates in a single file).

```shell
cd A
cp ../AC1/ac1cert.pem ./
cp ../AC2/ac2cert.pem ./
cat ac1cert.pem ac2cert.pem > certs.pem
openssl verify -CAfile certs.pem Acert.pem
cd ..
```

## Joining the certificate and the private key to sign in common applications (Word/ Email)

12. Export the certificate of entity A, its private key and both AC1 and AC2 certificates (file certs.pem) to PKCS12 format.

```shell
cd A
openssl pkcs12 -export -in Acert.pem -inkey Akey.pem -certfile certs.pem -out Acert.p12
cd ..
```

A passphrase is request to export A private key, and a new passphrase is request for ``Acert.p12``

## Use of A’s private key to sign a document

13. Create a Microsoft Word Office document and sign it electronically signed using A private key. In order to do that, you have to import ``Acert.p12`` in the browser (``Tools > Internet options > Content > Certificates > Import...``) and, then, using Microsoft Word make use of ``Office button > Prepare > Add digital signature...``.

## Questions

- What is the file **serial** used for?
- What is the file **index** used for?
- Could AC2 create a certificate applying step 2 of this script?
- If you or your lab group become a Certification Authority, explain and justify (i.e., advantages, disadvantages, alternatives…) the values you would use to configure the following parameters: ``default_days``, ``default_crl_days``, ``countryName``
- When you open the Word document, once signed, you may notice that a it gives a **Verification error**. Why does it happen? How can it be solved?
