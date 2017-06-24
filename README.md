# Specification for integrating CFRG algorithms in Java/JCE
This specification proposal is based on the following IETF RFCs and Drafts:
- https://tools.ietf.org/html/rfc7748
- https://tools.ietf.org/html/rfc8032
- https://tools.ietf.org/html/draft-ietf-curdle-pkix-04
- https://tools.ietf.org/html/draft-ietf-cose-msg-24
- https://tools.ietf.org/html/rfc8037

The core issue is if CFRG algorithms should overload current EC classes If distinct classes and interfaces are considered, "OKP" from RFC 8037 would be most logical.  Examples:
`OKPKey`, `OKPPublicKey`, and `OKPPrivateKey`

```
public interface OPKKey {
   public String getCurve();          // RFC 8037 "crv"
   public byte[] getX();              // RFC 8037 "x"
   public boolean isSignatureKey();   // According to specs a key is either Signature or DH
}
```

`KeyFactory.getInstance("OKP").generatePrivate(new OKPPrivateKeySpec(dValue, xValue, curveName))`

`KeyFactory.getInstance("OKP").generatePublic(new OKPPublicKeySpec(xValue, curveName))`

`AlgorithmParameterSpec spec =  new OKPGenParameterSpec(curveName)`
`KeyPairGenerator kpg = KeyPairGenerator.getInstance("OKP")`

`Signature signature = Signature.getInstance("EdDSA")`

The only place where it sort of breaks down is KeyAgreement, I would play it safe by inventing new name:
`KeyAgreement.getInstance("MoDH")` 
to not run into possible conflicts (_crashes_) with existing code and providers.  The additional test required to cope with "ECDH" and "MoDH" (for _Montgomery_ in analogy with _Edwards_) seems bearable:
`KeyAgreement.getInstance(publicKey instanceof ECKey ? "ECDH" : "MoDH")`

 The named curve identifiers needed seem to be:
  ```
 id-X25519    OBJECT IDENTIFIER ::= { 1 3 101 110 }
 id-X448      OBJECT IDENTIFIER ::= { 1 3 101 111 }
 id-Ed25519   OBJECT IDENTIFIER ::= { 1 3 101 112 }
 id-Ed448     OBJECT IDENTIFIER ::= { 1 3 101 113 }
```
_Although maybe not a prime consideration for Bouncycastle, it would IMO be a pity if Oracle took another route for CFRG support._

The COSE draft also refers to "OKP": https://tools.ietf.org/html/draft-ietf-cose-msg-24.
