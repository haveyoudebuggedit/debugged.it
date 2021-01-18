---
slug: working-with-certificates-in-java
guid: https://pasztor.at/blog/working-with-certificates-in-java/
authors:
- janos
categories:
- Technology
date: "2017-09-28T00:00:00Z"
publishDate: "2017-09-28T00:00:00Z"
summary: Let's face it, the Java crypto API is a mess. It is extremely hard to use,
  and very confusing. Let's try and make sense of this sorry excuse for an API.
images:
- posts/working-with-certificates-in-java.jpg
preview: posts/working-with-certificates-in-java.jpg
tags:
- Java
- Software Development
title: Working with Certificates in Java
---

## Install Bouncy Castle

Before we continue, you really really (really) want to install [Bouncy Castle](https://www.bouncycastle.org/). It provides many much needed implementations and this tutorial will assume you are using it.

If you are using Maven, simply add this to your POM file:

```xml
<dependency>
    <groupId>org.bouncycastle.bcprov-jdk15on.1.57.org.bouncycastle</groupId>
    <artifactId>bcprov-jdk15on</artifactId>
    <version>1.57</version>
</dependency>
``` 

## Loading Private Keys

There seems to be a lot of confusion between the different certificate formats, so let's clear it up a bit. First off,
the type of certificate your traditional certificate needs is the PKCS#8 format. They keys look like this:

```
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

In contrast, PKCS#1 keys look like this:

```
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

See the subtle difference? It's not just the header. Taking a look in an
[ASN.1 decoder](https://lapo.it/asn1js/), we see this for PKCS#8:

```
SEQUENCE(3 elem)
    INTEGER 0
    SEQUENCE(2 elem)
        OBJECT IDENTIFIER 1.2.840.113549.1.1.1 rsaEncryption (PKCS #1)
        NULL
    OCTET STRING(1 elem)
        SEQUENCE(9 elem)
            INTEGER 0
            INTEGER(2048 bit) ...
            INTEGER 65537
            INTEGER (2046 bit) ...
```

In contrast, a PKCS#1 looks like this:

```
SEQUENCE(9 elem)
    INTEGER 0
    INTEGER (2048 bit) ...
    INTEGER 65537
    INTEGER (2046 bit) ...
```

If you observe closely, both have the same data, but in a different structure. Both formats contain the private and
public key, yet, we have to load them differently.

Let's take a look how this is done in Java:

```java
String privateKeyString = "----...";

PemObject privateKeyObject;
try (
    PemReader pemReader = new PemReader(
        new InputStreamReader(
            new ByteArrayInputStream(privateKeyString.getBytes())
        )
    )
) {
    privateKeyObject = pemReader.readPemObject();
}

RSAPrivateCrtKeyParameters privateKeyParameter;
if (privateKeyObject.getType().endsWith("RSA PRIVATE KEY")) {
    //PKCS#1 key
    RSAPrivateKey rsa   = RSAPrivateKey.getInstance(privateKeyObject.getContent());
    privateKeyParameter = new RSAPrivateCrtKeyParameters(
        rsa.getModulus(),
        rsa.getPublicExponent(),
        rsa.getPrivateExponent(),
        rsa.getPrime1(),
        rsa.getPrime2(),
        rsa.getExponent1(),
        rsa.getExponent2(),
        rsa.getCoefficient()
    );
} else if (privateKeyObject.getType().endsWith("PRIVATE KEY")) {
    //PKCS#8 key
    privateKeyParameter = (RSAPrivateCrtKeyParameters) PrivateKeyFactory.createKey(
        privateKeyObject.getContent()
    );
} else {
    throw new RuntimeException("Unsupported key type: " + privateKeyObject.getType());
}

return new JcaPEMKeyConverter()
    .getPrivateKey(
        PrivateKeyInfoFactory.createPrivateKeyInfo(
            privateKeyParameter
        )
    );
```

OK, so now we have a private key, and it will always be in a PKCS#8 format.

## Writing a Private Key from PKCS#8

Now that we have a PKCS#8 private key, encoding to a PKCS#8 string is simple:

```java
ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
try (PemWriter pemWriter = new PemWriter(new OutputStreamWriter(outputStream))) {
    pemWriter.writeObject(new PemObject("PRIVATE KEY", this.privateKey.getEncoded()));
} catch (IOException e) {
    throw new RuntimeException(e);
}
return new String(outputStream.toByteArray());
```

To encode for PKCS#1 we need a little more legwork:

```java
ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
try (PemWriter pemWriter = new PemWriter(new OutputStreamWriter(outputStream))) {
    //Need to convert to PKCS#1
    PrivateKeyInfo pkInfo = PrivateKeyInfo.getInstance(privateKey.getEncoded());
    ASN1Encodable privateKeyPKCS1ASN1Encodable = pkInfo.parsePrivateKey();
    ASN1Primitive privateKeyPKCS1ASN1 = privateKeyPKCS1ASN1Encodable.toASN1Primitive();

    pemWriter.writeObject(new PemObject("RSA PRIVATE KEY", privateKeyPKCS1ASN1.getEncoded()));
} catch (IOException e) {
    throw new RuntimeException(e);
}
return new String(outputStream.toByteArray());
```

## Loading Certificates

All right, so private keys have been taken care of, let's load the certificates. Thankfully, this is simpler if the
certificate is in PEM format:

```java
try {
    List<X509Certificate> certificateChain = new ArrayList<>();
    CertificateFactory cf = CertificateFactory.getInstance("X.509");
    Collection c = cf.generateCertificates(
        new ByteArrayInputStream(
            certificateChainString.getBytes()
        )
    );
    Iterator i = c.iterator();
    while (i.hasNext()) {
        certificateChain.add((X509Certificate) i.next());
    }

    return certificateChain;
} catch (IOException|CertificateException e) {
    throw new RuntimeException(e);
}
```

## Writing Certificates

Writing them is similarly simple:

```java
StringBuilder encodedChain = new StringBuilder();
for (X509Certificate certificate : certificateChain) {
    encodedChain.append("-----BEGIN CERTIFICATE-----\n");
    try {
        encodedChain.append(new String(encoder.encode(certificate.getEncoded())));
    } catch (CertificateEncodingException e) {
        throw new RuntimeException(e);
    }
    encodedChain.append("\n-----END CERTIFICATE-----\n");
}
return encodedChain.toString();
```

## Getting the Public Key

Sometimes having just the private key is not enough. Fortunately both PKCS#1 and PKCS#8 encode enough information
to generate a public key as well. More specifically, we need the modulus and the public key exponent from our
loading code to do that:

```java
PublicKey publicKey;
try {
    RSAPublicKeySpec publicKeySpec = new RSAPublicKeySpec(
        privateKeyParameter.getModulus(),
        privateKeyParameter.getPublicExponent()
    );
    KeyFactory keyFactory = KeyFactory.getInstance("RSA");
    publicKey = keyFactory.generatePublic(publicKeySpec);
} catch (NoSuchAlgorithmException | InvalidKeySpecException e) {
    throw new RuntimeException(e);
}
```

## Generating Keys

Here comes the easy part, let's generate a private and public key:

```java
KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
generator.initialize(2048);
KeyPair keyPair = generator.generateKeyPair();
Key publicKey = keyPair.getPublic();
Key privateKey = keyPair.getPrivate();
```

You can now save the private key with the method I described above.

## Generating a CSR

Now that we have the basics dealt with, let's generate a certificate signing request:

```java
//Put your keypair from the previous parts here
KeyPair pair = ...;

PKCS10CertificationRequestBuilder p10Builder = new JcaPKCS10CertificationRequestBuilder(
    new X500Principal("C=US, L=Vienna, O=Your Company Inc, CN=yourdomain.com/emailAddress=your@email.com"),
    pair.getPublic()
);
JcaContentSignerBuilder csBuilder = new JcaContentSignerBuilder("SHA256withRSA");
ContentSigner signer = null;
try {
    signer = csBuilder.build(pair.getPrivate());
} catch (OperatorCreationException e) {
    throw new RuntimeException(e);
}
PKCS10CertificationRequest csr = p10Builder.build(signer);
```

Writing it to a file works similar to our previous ones:

```java
ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
try (PemWriter pemWriter = new PemWriter(new OutputStreamWriter(outputStream))) {
    pemWriter.writeObject(new PemObject("CERTIFICATE REQUEST", csr.getEncoded()));
} catch (IOException e) {
    throw new RuntimeException(e);
}
return new String(outputStream.toByteArray());
```

## Signing a Certificate

Now to our last bit:

```java
PKCS10CertificationRequest csr = the previous CSR object;
KeyPair caKeyPair = the key pair of the certificate authority;

PKCS10CertificationRequest csrHolder = new PKCS10CertificationRequest(csr.getEncoded());
X509v3CertificateBuilder certificateGenerator = new X509v3CertificateBuilder(
    //These are the details of the CA
    new X500Name("C=US, L=Vienna, O=Your CA Inc"),
    //This should be a serial number that the CA keeps track of
    new BigInteger("1"),
    //Certificate validity start
    Date.from(LocalDateTime.now().toInstant(ZoneOffset.UTC)),
    //Certificate validity end
    Date.from(LocalDateTime.now().plusDays(365).toInstant(ZoneOffset.UTC)),
    //Blanket grant the subject as requested in the CSR
    //A real CA would want to vet this.
    csrHolder.getSubject(),
    //Public key of the certificate authority
    SubjectPublicKeyInfo.getInstance(ASN1Sequence.getInstance(caKeyPair.getPublic().getEncoded()))
);
ContentSigner sigGen = new BcRSAContentSignerBuilder(sigAlgId, digAlgId)
    .build(PrivateKeyFactory.createKey(caKeyPair.getPrivate().getEncoded()));

X509CertificateHolder holder = certificateGenerator.build(sigGen);
CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509", "BC");
X509Certificate certificate = (X509Certificate) certificateFactory.generateCertificate(
    new ByteArrayInputStream(
        holder.toASN1Structure().getEncoded()
    )
);

ByteArrayOutputStream certOutputStream = new ByteArrayOutputStream();
try (PemWriter pemWriter = new PemWriter(new OutputStreamWriter(certOutputStream))) {
    pemWriter.writeObject(new PemObject("CERTIFICATE", certificate.getEncoded()));
} catch (IOException|CertificateEncodingException e) {
    throw new RuntimeException(e);
}
```

## Conclusion

That's it! You should now have about 80-90% of what you need to work with certificates in Java. I personally hope that
I never have to deal with it again, or if I have to, I can just copy-paste stuff from my own blog post. :)

Happy hunting!