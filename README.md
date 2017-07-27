# Specification for integrating CFRG algorithms in Java/JCE
This specification proposal is based on the following IETF RFCs and Drafts:
- https://tools.ietf.org/html/rfc7748
- https://tools.ietf.org/html/rfc8032
- https://tools.ietf.org/html/draft-ietf-curdle-pkix-04
- https://tools.ietf.org/html/draft-ietf-cose-msg-24
- https://tools.ietf.org/html/rfc8037

The core issue is if CFRG algorithms should reuse the current EC classes or not.  *This specification is based
on the idea that the CFRG algorithms differ too much from EC to be conveniently
and logically mapped into the current EC classes and interfaces.*

None of the existing external representations of CFRG keys specify parameters like `ECPoint`, `coFactor`, or `ECField`;  the PKIX draft does not reuse the ASN.1 EC definitions even for named curves.

RFC 8037: *Do not assume that there is an underlying elliptic curve,
   despite the existence of the "crv" and "x" parameters.  (For
   instance, this key type could be extended to represent Diffie-Hellman
   (DH) algorithms based on hyperelliptic surfaces.)*

Note: If provider implementations *internally* reuse EC structures is something else which does not have to exposed in "application-level" APIs.

The remaining question would then be what to call this new key type.
Since both RFC 8037 and the COSE draft use the name "OKP" (Octet Key Pair), it seems reasonable adopting this name here as well.

Below is a very condensed version of the propoposal:

```java
public interface OPKKey {
   public String getCurve();                // Algorithm | RFC 8037 "crv"

   public int USAGE_SIGNATURE = 1;          // For usage with "isPermitted()"
   public int USAGE_DH = 2;                 // For usage with "isPermitted()"
   public boolean isPermitted(int usages);  // According to specs a key is either Signature or DH
}
```

```java
public interface OKPPublicKey extends PublicKey, OKPKey {
   public byte[] getX();  // Public key value | RFC 8037 "x"
}
```

```java
public interface OKPPrivateKey extends PrivateKey, OKPKey {
  public byte[] getD();  // Private key value | RFC 8037 "d"
}
```

```java
String curve;  // Algorithm name
byte[] x;      // Public key value
byte[] d;      // Private key value
```

```java
KeyFactory.getInstance("OKP").generatePrivate(new OKPPrivateKeySpec(d, curve));
```

```java
KeyFactory.getInstance("OKP").generatePublic(new OKPPublicKeySpec(x, curve));
```

```java
AlgorithmParameterSpec keySpec = new OKPGenParameterSpec(curve);
KeyPairGenerator kpg = KeyPairGenerator.getInstance("OKP");
kpg.initialize(keySpec);
```

```java
PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(pkcs8PrivateKeyBlob);
KeyFactory.getInstance("OKP").generatePrivate(keySpec);
```

```java
Signature signature = Signature.getInstance("EdDSA");
```

The only place where it sort of breaks down is KeyAgreement, I would play it safe by inventing new name:
`KeyAgreement.getInstance("MoDH")` 
to not run into possible conflicts (_crashes_) with existing code and providers.  The additional test required to cope with "ECDH" and "MoDH" (for _Montgomery_ in analogy with _Edwards_) seems bearable:
```java
KeyAgreement.getInstance(publicKey instanceof ECKey ? "ECDH" : "MoDH");
```

 The key algorithm/curve identifiers needed are:
  ```
 id-X25519    OBJECT IDENTIFIER ::= { 1 3 101 110 }
 id-X448      OBJECT IDENTIFIER ::= { 1 3 101 111 }
 id-Ed25519   OBJECT IDENTIFIER ::= { 1 3 101 112 }
 id-Ed448     OBJECT IDENTIFIER ::= { 1 3 101 113 }
```
<table>
<tr><td><i>It would IMO be a pity if Oracle, Bouncycastle, and Google(Android) took different routes for CFRG support</i></td></tr>
</table>

## PKCS \#11 and .NET
Ideally other cryptographic APIs should also adopt OKP keys.
