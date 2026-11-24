---
v: 3
title: "Additions to C509 Structures"
docname: draft-liao-cose-c509-additions-01
abbrev: C509 Additions

cat: std
submissiontype: IETF
coding: utf-8
pi:
  toc: yes
  sortrefs: yes
  symrefs: yes
  tocdepth: 3
venue:
  group: "CBOR Object Signing and Encryption"
  type: "Working Group"
  mail: "cose@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/cose/"

author:
  - name: Lijun Liao
    org: NIO Inc.
    email: lijun.liao@nio.io

normative:
  RFC3739:
  RFC4648:
  RFC5280:
  RFC7468:
  RFC9090:
  RFC9277:
  RFC9371:
  RFC9608:
  I-D.ietf-cose-cbor-encoded-cert:
  IANA.cbor-tags:
  IANA.PEN:
    title: Private Enterprise Numbers (PENs)
    author:
      org: IANA
    target: https://www.iana.org/assignments/enterprise-numbers/

informative:
  RFC6838:
  I-D.liao-cose-c509-revocation:
  IANA.SMI:
    title: Structure of Management Information (SMI) Numbers (MIB Module Registrations)
    author:
      org: IANA
    target: https://www.iana.org/assignments/smi-numbers/

--- abstract

This document defines additions to CBOR Encoded X.509 Certificates (C509).

This document defines a new C509SubjectPublicKeyInfo type, CBOR representations of the X.509 SubjectDirectoryAttributes and No Revocation Available extensions, additional extended key usage identifiers, and a mechanism that allows organizations assigned a Private Enterprise Number (PEN) to define organization-specific integer identifiers without registering each identifier in the "C509 RDN Attributes", "C509 Attributes", "C509 Extensions", "C509 Certificate Policies", or "C509 Extended Key Usages" registries.

This document also defines textual encoding labels for C509 objects using the textual encoding conventions specified in RFC 7468.

This document updates the C509 specification by renaming the certification request attribute type to a general attribute type and the "C509 CR Attributes" registry to "C509 Attributes".

--- middle

# Introduction {#intro}

{{I-D.ietf-cose-cbor-encoded-cert}} defines the C509 certificate structure, a compact CBOR encoding of X.509 certificates. C509 supports a large subset of the X.509 certificate profile defined in {{RFC5280}} and is designed for use in constrained environments as well as general certificate-based protocols.

The base C509 specification defines the public key as fields within the certificate structure. It also defines compact integer identifiers for selected X.509 RDN attribute types, certification request (CR) attribute types, extension IDs, certificate policies, and extended key usages.

This document updates and extends the C509 specification in the following areas.

- Updates {{I-D.ietf-cose-cbor-encoded-cert}}: the CDDL types CRAttributes and CRAttribute are renamed to Attributes and Attribute, so that they can be used outside certification requests (see {{updates}}).

  [[RFC Editor: please add "Updates: RFC XXXX" to the header of this document, where XXXX is the RFC number assigned to {{I-D.ietf-cose-cbor-encoded-cert}}, and remove this note.]]

- Defines a new structure C509SubjectPublicKeyInfo which corresponds to SubjectPublicKeyInfo in X.509 (see {{Section 4.1 of RFC5280}}). This permits a C509 public key to be represented independently of a complete C509 certificate.

- Defines the C509 representation of the X.509 SubjectDirectoryAttributes extension specified in {{Section 4.2.1.8 of RFC5280}}. The "C509 CR Attributes" registry is renamed to "C509 Attributes", which is a general registry for attributes, and is extended with entries for subject directory attributes.

- Defines the C509 representation of the X.509 No Revocation Available extension specified in {{RFC9608}}.

- Registers additional extended key usages in the "C509 Extended Key Usages" registry, for example id-kp-ipsecIKE for use with IKEv2 (see {{EKU}}).

- Defines PEN-scoped identifiers. An organization that has been assigned a Private Enterprise Number (PEN) can assign integer values within its own namespace. Such identifiers can be used for RDN attribute types, attribute types, extension IDs, certificate policies, and extended key usages without requiring each identifier to be registered in a C509 IANA registry. Unlike values from the Private Use ranges of these registries, PEN-scoped identifiers are globally unique, so identifiers defined by different organizations do not collide.

- Defines PEM labels for C509 objects using the textual encoding framework specified by {{RFC7468}}.

This document is intended to be used together with {{I-D.ietf-cose-cbor-encoded-cert}}. It does not define a new certificate format; it adds types and encoding mechanisms to the C509 data model.

# Notational Conventions {#notation}

{::boilerplate bcp14-tagged}

# Updates to C509 {#updates}

This section updates {{I-D.ietf-cose-cbor-encoded-cert}}. The updates do not change the encoding of any C509 structure that conforms to both specifications.

## Attribute {#updates-attribute}

{{I-D.ietf-cose-cbor-encoded-cert}} defines the attributes of a C509 certification request with the CDDL types CRAttributes and CRAttribute. The same structure is also needed outside certification requests, for example in the SubjectDirectoryAttributes extension (see {{ext-SubjectDirectoryAttributes}}). This document renames these types as follows:

- CRAttributes is renamed to Attributes.
- CRAttribute is renamed to Attribute.
- The term "CR Attribute" is renamed to "Attribute".

The CDDL of TBSCertificationRequest is updated accordingly:

~~~~~~~~~~~ cddl
; The elements of the following group are used in a CBOR Sequence:
TBSCertificationRequest = (
   c509CertificationRequestType: int,
   subjectSignatureAlgorithm: AlgorithmIdentifier,
   subject: Name,
   subjectPublicKeyAlgorithm: AlgorithmIdentifier,
   subjectPublicKey: Defined,
   attributes: Attributes,
)

Attributes = [ * Attribute ]

Attribute = (( attributeType: int, attributeValue: Defined ) //
             ( attributeType: ~oid, attributeValue: bytes ))
~~~~~~~~~~~
{: sourcecode-name="c509additions.cddl"}
{: #fig-AttributeCDDL title="Updated CDDL for TBSCertificationRequest and Attribute"}

The corresponding IANA registry is renamed from "C509 CR Attributes" to "C509 Attributes" (see {{attrtype}}). This document also constrains the encoding of attributes as follows.

If 'attributeType' is encoded as a CBOR int, 'attributeValue' carries exactly one AttributeValue. An attribute with more than one value is encoded as follows:

- In a C509 certificate or certification request that is re-encoded from X.509 (type 3), the attribute MUST be encoded with an unwrapped CBOR OID 'attributeType', followed by the DER-encoded 'values'.
- In a natively signed C509 certificate or certification request (type 2), each value is encoded as a separate attribute with the same 'attributeType'. For example, {{RFC3739}} recommends that multiple countries of citizenship or residence be specified through separate, single-valued attributes.

# C509 Public Key {#publickey}

This document defines the following reusable C509 subject public key type:

~~~~~~~~~~~ cddl
C509SubjectPublicKeyInfo = [
  subjectPublicKeyAlgorithm: AlgorithmIdentifier,
  subjectPublicKey:          Defined
]
~~~~~~~~~~~
{: sourcecode-name="c509additions.cddl"}
{: #fig-C509SubjectPublicKeyInfoCDDL title="CDDL for C509SubjectPublicKeyInfo"}

The C509SubjectPublicKeyInfo item is served with the application/cose-c509-pubkey+cbor media type, see {{cose-c509-pubkey}}, with the corresponding CoAP Content-Format defined in {{content-format}}. A stored file format is defined with "magic number" TBD2 using the reserved CBOR tag 55799 and the Content-Format TBD1, enveloped as described in {{Section 2.2 of RFC9277}}.

The subjectPublicKeyAlgorithm and subjectPublicKey fields have the same encoding and semantics as the corresponding fields of TBSCertificate in {{I-D.ietf-cose-cbor-encoded-cert}}.

In particular, algorithm-specific C509 encodings defined by {{I-D.ietf-cose-cbor-encoded-cert}} apply equally when the public key is represented using C509SubjectPublicKeyInfo. For example, the optimized RSA encoding and elliptic curve public key encoding apply to C509SubjectPublicKeyInfo.

An elliptic curve public key is encoded as in a natively signed C509 certificate (type 2). The leading bytes 0xfe and 0xfd MUST NOT be used. When a Weierstrass-form elliptic curve public key is converted to an X.509 SubjectPublicKeyInfo, a point encoded with leading byte 0x02 or 0x03 is encoded as a compressed point, and a point encoded with leading byte 0x04 is encoded as an uncompressed point.

# Extensions {#extensions}

## Extension Subject Directory Attributes {#ext-SubjectDirectoryAttributes}

X.509 defines the SubjectDirectoryAttributes extension in {{Section 4.2.1.8 of RFC5280}}. This document defines a C509 representation of that extension and registers the extension ID 24 for it (see {{extype}}). {{RFC5280}} requires this extension to be non-critical, so the 'extensionID' is encoded as the positive int 24 in conforming certificates. When re-encoding an X.509 certificate that marks this extension critical as a type 3 C509 certificate, the 'extensionID' is encoded as -24 to preserve the criticality. The extension value is represented as SubjectDirectoryAttributes:

~~~~~~~~~~~ cddl
SubjectDirectoryAttributes = [ + Attribute ]

DateOfBirth = int

PlaceOfBirth = text / #6.121(text)
~~~~~~~~~~~
{: sourcecode-name="c509additions.cddl"}
{: #fig-SubjectDirectoryAttributes title="CDDL for SubjectDirectoryAttributes"}

[[RFC Editor: please remove this paragraph before publication.]] The value 24 was assigned to subjectDirectoryAttributes in earlier versions of {{I-D.ietf-cose-cbor-encoded-cert}} with a different encoding of the extension value. Implementations of those versions are not interoperable with this document.

Attribute is defined in {{updates-attribute}}. Each 'attributeType' is encoded either as a CBOR int (see {{attrtype}}) or as an unwrapped CBOR OID {{RFC9090}}.

- If 'attributeType' is encoded as a CBOR int, it is followed by a CBOR item of any type other than undefined. The specific type of the item is determined by 'attributeType'.
- If 'attributeType' is encoded as an unwrapped CBOR OID, it is followed by the DER-encoded 'values' (which is SET OF AttributeValue) encoded as a CBOR byte string.

The subsections below constrain the values that can be carried with an int 'attributeType'. In a C509 certificate or certification request that is re-encoded from X.509 (type 3), an attribute with a value that does not satisfy these constraints MUST be encoded with an unwrapped CBOR OID 'attributeType', followed by the DER-encoded 'values'.

### DateOfBirth

The personal data attribute 'DateOfBirth' is defined in {{RFC3739}} as 'GeneralizedTime'. {{RFC3739}} specifies that the value SHOULD specify GMT 12.00.00 (noon), for example "19590927120000Z". In C509, the 'attributeValue' field has type DateOfBirth: the date is encoded as a CBOR int containing the number of days since 1970-01-01. Dates before 1970-01-01 are encoded as negative integers. For example, September 27, 1959 is encoded as -3749.

When a C509 DateOfBirth value is converted to X.509, it is encoded as the GeneralizedTime value "YYYYMMDD120000Z". In a C509 certificate or certification request that is re-encoded from X.509 (type 3), a DateOfBirth attribute whose value does not have this form MUST be encoded with an unwrapped CBOR OID 'attributeType', followed by the DER-encoded 'values'.

### PlaceOfBirth {#placeofbirth}

The personal data attribute 'PlaceOfBirth' is defined in {{RFC3739}} as 'DirectoryString'. In C509, the 'attributeValue' field has type PlaceOfBirth. A UTF8String is encoded as CBOR text, and a PrintableString is encoded as CBOR text tagged with number 121 (alternative 0, as defined in {{IANA.cbor-tags}}). A PlaceOfBirth value of any other DirectoryString type cannot be carried with the int 'attributeType'; in a re-encoded C509 certificate or certification request (type 3), it MUST be encoded with an unwrapped CBOR OID 'attributeType', followed by the DER-encoded 'values'. In natively signed C509 certificates and certification requests (type 2), only UTF8String (untagged CBOR text) is allowed.

### Gender

The personal data attribute 'Gender' is defined in {{RFC3739}} as 'PrintableString (SIZE(1))', with the value "M", "F", "m", or "f". In C509, the 'attributeValue' field has type CBOR text, which MUST consist of exactly one character from the PrintableString character set.

### CountryOfCitizenship

The personal data attribute 'CountryOfCitizenship' is defined in {{RFC3739}} as 'PrintableString (SIZE(2))', containing an ISO 3166 country code as specified in {{RFC3739}}. In C509, the 'attributeValue' field has type CBOR text, which MUST consist of exactly two characters from the PrintableString character set.

### CountryOfResidence

The personal data attribute 'CountryOfResidence' is defined in {{RFC3739}} as 'PrintableString (SIZE(2))', containing an ISO 3166 country code as specified in {{RFC3739}}. In C509, the 'attributeValue' field has type CBOR text, which MUST consist of exactly two characters from the PrintableString character set.

When a C509 Gender, CountryOfCitizenship, or CountryOfResidence value is converted to X.509, it is encoded as a PrintableString. In a C509 certificate or certification request that is re-encoded from X.509 (type 3), an attribute of any of these types whose value is not a PrintableString MUST be encoded with an unwrapped CBOR OID 'attributeType', followed by the DER-encoded 'values'.

## Extension No Revocation Available {#ext-noRevAvail}

{{RFC9608}} defines the No Revocation Available (noRevAvail) extension, which indicates that the issuer does not provide revocation status information for the certificate. This document defines a C509 representation of that extension and registers the extension ID 39 for it (see {{extype}}). {{RFC9608}} requires this extension to be non-critical, so the 'extensionID' is encoded as the positive int 39 in conforming certificates. When re-encoding an X.509 certificate that marks this extension critical as a type 3 C509 certificate, the 'extensionID' is encoded as -39 to preserve the criticality.

In X.509, the extension value is the DER encoding of NULL. In C509, 'extensionValue' is encoded as the CBOR simple value null (0xf6), in the same way as for the OCSP No Check extension in {{I-D.ietf-cose-cbor-encoded-cert}}:

~~~~~~~~~~~ cddl
NoRevAvail = null
~~~~~~~~~~~
{: sourcecode-name="c509additions.cddl"}
{: #fig-NoRevAvail title="CDDL for NoRevAvail"}

The rules in {{RFC9608}} for the use of this extension apply unchanged to C509 certificates.

# PEN-Scoped Identifiers {#pen-id}

{{I-D.ietf-cose-cbor-encoded-cert}} defines integer identifiers for commonly used RDN attributes, attributes, extensions, extended key usages, and certificate policies. This document registers further integer identifiers in the "C509 Attributes" registry (formerly "C509 CR Attributes", see {{attrtype}}), the "C509 Extensions" registry (see {{extype}}), and the "C509 Extended Key Usages" registry (see {{EKU}}). These integer identifiers provide compact encodings.

An organization may also need identifiers that are meaningful only within its own ecosystem. Requiring each such identifier to be allocated from an IANA registry is unnecessary when the organization already has a globally unique namespace through a Private Enterprise Number {{IANA.PEN}}.

This document therefore reserves two intervals for PEN-scoped identifiers for RDN attributes, attributes (see {{attrtype}}), extensions, extended key usages, and certificate policies. A PEN-scoped identifier is constructed from the numeric component of a PEN and an internal ID assigned by the organization to which the PEN is assigned. As specified in {{Section 3 of RFC9371}}, the numeric component ranges from 0 to 2^32 - 1. In the following, PEN denotes this numeric component.

The form of a PEN-scoped identifier is determined by the PEN and the internal ID as follows:

- For a PEN in [0, 2^21 - 1] and an internal ID in [0, 2^10 - 1], the short form MUST be used. The identifier is in the interval [2^31, 2^32 - 1] and is

  ~~~
  2^31 + PEN * 2^10 + internal ID
  ~~~

  i.e., the bit 1 followed by the 21-bit PEN and the 10-bit internal ID. It is encoded in 5 bytes as a CBOR int.

- For a PEN in [2^21, 2^32 - 1] and an internal ID in [0, 2^15 - 1], or for a PEN in [0, 2^32 - 1] and an internal ID in [2^10, 2^15 - 1], the long form MUST be used. The identifier is in the interval [2^47, 2^48 - 1] and is

  ~~~
  2^47 + PEN * 2^15 + internal ID
  ~~~

  i.e., the bit 1 followed by the 32-bit PEN and the 15-bit internal ID. It is encoded in 9 bytes as a CBOR int.

- An internal ID greater than or equal to 2^15 is not allowed; no PEN-scoped identifier can be constructed for it.

These cases do not overlap, so each PEN-scoped identifier has exactly one encoding. A decoder MUST reject a structure containing a long-form identifier with a PEN less than 2^21 and an internal ID less than 2^10.

PEN-scoped identifiers do not prohibit the registration of an OID under the arc of a PEN in a C509 registry. Such an OID can be registered with an integer value outside the intervals reserved for PEN-scoped identifiers, following the registration procedure of the applicable registry. A registered value can be used in all C509 structures, including those re-encoded from X.509 (type 3).

PEN-scoped identifiers MAY be used in natively signed C509 certificates and certification requests (type 2), and in C509CertificationRequestTemplate. A template that requires a PEN-scoped identifier can only be satisfied by a type 2 certification request. PEN-scoped identifiers MAY also be used in C509 CRL and C509 OCSP structures defined in {{I-D.liao-cose-c509-revocation}} where those structures use the C509 registries listed above.

PEN-scoped identifiers MUST NOT be used in C509 certificates and certification requests that are re-encoded from X.509 (type 3). In such structures, an object identifier under the arc of a PEN for which no integer is registered in the applicable C509 registry is encoded as an unwrapped CBOR OID (~oid). If an integer is registered, the encoding follows the rules of {{I-D.ietf-cose-cbor-encoded-cert}} and this document. A decoder MUST treat a type 3 structure that contains a PEN-scoped identifier as invalid. This keeps the encoding of a re-encoded structure independent of whether the encoder knows the definitions of the organization, and allows any party to reconstruct its DER encoding.

## Sign of PEN-Scoped Identifiers {#pen-id-sign}

The constructions above determine the absolute value of a PEN-scoped identifier. Where {{I-D.ietf-cose-cbor-encoded-cert}} uses the sign of an integer identifier to carry additional information, the same rules apply to PEN-scoped identifiers:

- For RDN attributes, a PEN-scoped 'attributeType' is always positive.

- For extensions, the sign indicates criticality: critical extensions are encoded with a negative sign and non-critical extensions with a positive sign. For example, a critical extension with the PEN-scoped identifier v is encoded as -v.

In registries where the sign of an identifier carries no additional information, only the positive values in the intervals above are PEN-scoped identifiers.

## Values and Mapping to X.509 {#pen-id-oid}

A PEN-scoped identifier does not encode the object identifier (OID) that it stands for, and the mapping is not recorded in an IANA registry.

The item that follows a PEN-scoped identifier, for example the 'extensionValue' after a PEN-scoped 'extensionID' or the 'attributeValue' after a PEN-scoped 'attributeType', is a native CBOR item. Its type and semantics are defined by the organization to which the PEN is assigned, subject to the CDDL of the containing structure. In certificates and certification requests, the value of an RDN attribute MUST be SpecialText, and the value of an extension or Attribute MUST be Defined. In certification request templates, undefined may be used where permitted by the template CDDL. That organization also defines the mapping of the identifier to an OID and the conversion between the CBOR item and the corresponding X.509 (DER) value.

When a signature is calculated over CBOR-encoded data containing a PEN-scoped identifier, no OID mapping or value conversion is needed to verify the signature. A relying party still needs to know the definitions of a PEN-scoped identifier to process the item it identifies.

# Textual Encoding (PEM) {#pem}

This section defines textual (PEM) encodings for C509 structures, following the conventions of {{RFC7468}}. The textual encodings use base64-encoded binary data enclosed between "-----BEGIN LABEL-----" and "-----END LABEL-----" lines and carry CBOR-encoded C509 objects rather than DER-encoded ASN.1 objects.

This document does not define a PEM label for C509CertificationRequestTemplate.

## General Rules {#pem-general-rules}

Each C509 PEM block MUST conform to the following rules:

- The binary payload is the CBOR encoding of the structure as specified in {{I-D.ietf-cose-cbor-encoded-cert}} or in this document.

- The binary payload is base64-encoded according to {{Section 4 of RFC4648}} (standard base64 with padding characters `=`). Generators MUST wrap the base64-encoded lines as specified in {{Section 2 of RFC7468}}. Parsers SHOULD ignore whitespace and other non-base64 characters as described in {{Section 2 of RFC7468}}.

- The encapsulation boundaries follow {{Section 2 of RFC7468}}:

  ~~~
  -----BEGIN LABEL-----
  (base64-encoded CBOR payload)
  -----END LABEL-----
  ~~~

- Explanatory text before the pre-encapsulation boundary is allowed. Implementations MUST NOT require or rely on such text.

## C509 Certificate {#pem-c509-cert}

PEM label: `C509 CERTIFICATE`

The payload is a CBOR-encoded C509Certificate as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN C509 CERTIFICATE-----
(base64-encoded CBOR C509Certificate)
-----END C509 CERTIFICATE-----
~~~
{: #fig-pem-c509-cert title="C509 Certificate PEM Encoding"}

## COSE_C509 {#pem-cose-c509}

PEM label: `COSE C509`

The payload is a CBOR-encoded COSE_C509 as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN COSE C509-----
(base64-encoded CBOR COSE_C509)
-----END COSE C509-----
~~~
{: #fig-pem-cose-c509 title="COSE_C509 PEM Encoding"}

## C509 Certification Request {#pem-c509-csr}

PEM label: `C509 CERTIFICATE REQUEST`

The payload is a CBOR-encoded C509CertificationRequest as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN C509 CERTIFICATE REQUEST-----
(base64-encoded CBOR C509CertificationRequest)
-----END C509 CERTIFICATE REQUEST-----
~~~
{: #fig-pem-c509-csr title="C509 Certification Request PEM Encoding"}

## C509 Public Key {#pem-c509-pubkey}

PEM label: `C509 PUBLIC KEY`

The payload is a CBOR-encoded C509SubjectPublicKeyInfo as defined in {{publickey}}.

~~~
-----BEGIN C509 PUBLIC KEY-----
(base64-encoded CBOR C509SubjectPublicKeyInfo)
-----END C509 PUBLIC KEY-----
~~~
{: #fig-pem-c509-pubkey title="C509 Public Key PEM Encoding"}

## C509 Private Key {#pem-c509-privkey}

PEM label: `C509 PRIVATE KEY`

The payload is a CBOR-encoded C509PrivateKey as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN C509 PRIVATE KEY-----
(base64-encoded CBOR C509PrivateKey)
-----END C509 PRIVATE KEY-----
~~~
{: #fig-pem-c509-privkey title="C509 Private Key PEM Encoding"}

## C509 PEM {#pem-c509-pem}

PEM label: `C509 PEM`

The payload is a CBOR-encoded C509PEM as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN C509 PEM-----
(base64-encoded CBOR C509PEM)
-----END C509 PEM-----
~~~
{: #fig-pem-c509-pem title="C509 PEM Encoding"}

## PEM Label Summary {#pem-label-summary}

The following table summarizes all PEM labels defined in this document and compares them with RFC 7468 labels for equivalent DER-encoded structures. C509SubjectPublicKeyInfo is defined in {{publickey}}; all other structures are defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

| PEM Label | C509 Structure | RFC 7468 Label |
|:---|:---|:---|
| `C509 CERTIFICATE` | `C509Certificate` | `CERTIFICATE` |
| `COSE C509` | `COSE_C509` | N/A |
| `C509 CERTIFICATE REQUEST` | `C509CertificationRequest` | `CERTIFICATE REQUEST` |
| `C509 PUBLIC KEY` | `C509SubjectPublicKeyInfo` | `PUBLIC KEY` |
| `C509 PRIVATE KEY` | `C509PrivateKey` | `PRIVATE KEY` |
| `C509 PEM` | `C509PEM` | N/A |
{: #tab-pem-labels title="PEM Labels Defined in This Document"}

{::comment}
Note to the author: ask the authors of draft-bonnell-lamps-pem-label-registry to add the
PEM labels defined in this document to the initial contents of their "PEM Labels" registry.
{:/comment}

# Security Considerations {#security}

The security considerations of {{I-D.ietf-cose-cbor-encoded-cert}} apply to this document.

## C509 Public Key {#sec-publickey}

C509SubjectPublicKeyInfo carries the same information as the subject public key fields of a C509 certificate. A C509SubjectPublicKeyInfo alone does not bind the public key to an identity. Applications that receive a public key in this form need a separate means, such as a certificate or a pre-established trust relationship, to decide whether to trust the key.

## Subject Directory Attributes {#sec-sda}

The attributes in {{ext-SubjectDirectoryAttributes}} (date of birth, place of birth, gender, country of citizenship, and country of residence) contain personal data. Certificates are often public and can be sent in the clear, for example in protocols that do not encrypt the certificate exchange. Certification Authorities SHOULD include these attributes only when they are required by the application, and applications SHOULD consider the privacy impact of including them. The compact C509 encoding does not change the privacy properties of these attributes compared to X.509.

## No Revocation Available {#sec-noRevAvail}

The security considerations of {{RFC9608}} apply to C509 certificates that contain the No Revocation Available extension. The C509 encoding does not change the semantics of the extension.

## PEN-Scoped Identifiers {#sec-pen-id}

A PEN-scoped identifier is meaningful only to parties that know the definition published by the organization to which the PEN is assigned. Different parties can therefore interpret the same identifier differently if they obtain the mapping from different sources or if the organization assigns an internal ID to more than one OID. Such a misinterpretation cannot be detected by the relying party: the signature of a natively signed C509 certificate is calculated over the CBOR encoding and remains valid regardless of how the identifier is interpreted. An organization MUST NOT assign an internal ID to more than one OID, and it MUST NOT reassign an internal ID after it has been used.

PEN-scoped identifiers are not used in re-encoded C509 certificates and certification requests (see {{pen-id}}), so the verification of their signatures never depends on the definitions of an organization. A party that does not recognize an extension identified by a PEN-scoped identifier MUST reject the certificate if the extension is critical and MAY ignore it if it is non-critical, as specified in {{Section 4.2 of RFC5280}}.

The PEN only identifies the organization that defines an identifier. It does not authenticate any statement in the certificate; the authenticity of the certificate content is based only on the signature of the issuer.

## Textual Encoding {#sec-pem}

The `C509 PRIVATE KEY` and `C509 PEM` labels carry private keys without encryption. Such PEM blocks MUST be protected against unauthorized disclosure, for example by restricting access to the files in which they are stored. The general security considerations of {{RFC7468}} apply.

Explanatory text is outside the base64-encoded payload. Such text is not authenticated and MUST NOT be used for security decisions.

# IANA Considerations {#iana}

## C509 RDN Attributes Registry {#rdnatttype}

This document updates the "C509 RDN Attributes" registry in the registry group "CBOR Encoded X.509 (C509)" defined in {{I-D.ietf-cose-cbor-encoded-cert}}. The intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1], which are part of the Future Use interval [65536, 2^63 - 1], are set aside for PEN-scoped identifiers as specified in {{pen-id}}. IANA is requested to add the following note to the registry:

> Values in the intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1] are PEN-scoped identifiers as specified in [[this document]]. They are not assigned by IANA.

## C509 Attributes Registry {#attrtype}

This document updates the "C509 CR Attributes" registry in the registry group "CBOR Encoded X.509 (C509)" defined in {{I-D.ietf-cose-cbor-encoded-cert}} as follows:

- The registry is renamed from "C509 CR Attributes" to "C509 Attributes". The registry is a general registry for attributes (the Attribute type, which pairs an attribute type with attribute values), wherever they are used in C509 structures. Examples include attributes in C509 certification requests and attributes in the SubjectDirectoryAttributes extension (see {{ext-SubjectDirectoryAttributes}}).

- An attribute registered in the "C509 Attributes" registry can be used in C509 certification requests, in the SubjectDirectoryAttributes extension, and in fields defined in future specifications that carry attributes. Whether an attribute is meaningful in a given structure is determined by the specification that defines the attribute. Expert reviewers are expected to check that a registration is not specific to a single structure in a way that prevents its use in the others.

- The column heading "CR Attribute" is renamed to "Attribute".

- The intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1], which are part of the Future Use interval [65536, 2^63 - 1], are set aside for PEN-scoped identifiers as specified in {{pen-id}}. IANA is requested to add the following note to the registry:

  > Values in the intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1] are PEN-scoped identifiers as specified in [[this document]]. They are not assigned by IANA.

IANA is requested to add the following entries to the "C509 Attributes" registry:

~~~~~~~~~~~
+-------+-----------------------------------------------------------+
| Value | Attribute                                                 |
+=======+===========================================================+
|     3 | Name:            DateOfBirth                              |
|       | Identifiers:     id-pda-dateOfBirth                       |
|       | OID:             1.3.6.1.5.5.7.9.1                        |
|       | DER:             06 08 2B 06 01 05 05 07 09 01            |
|       | Comments:        RFC 3739                                 |
|       | attributeValue:  DateOfBirth                              |
+-------+-----------------------------------------------------------+
|     4 | Name:            PlaceOfBirth                             |
|       | Identifiers:     id-pda-placeOfBirth                      |
|       | OID:             1.3.6.1.5.5.7.9.2                        |
|       | DER:             06 08 2B 06 01 05 05 07 09 02            |
|       | Comments:        RFC 3739                                 |
|       | attributeValue:  PlaceOfBirth                             |
+-------+-----------------------------------------------------------+
|     5 | Name:            Gender                                   |
|       | Identifiers:     id-pda-gender                            |
|       | OID:             1.3.6.1.5.5.7.9.3                        |
|       | DER:             06 08 2B 06 01 05 05 07 09 03            |
|       | Comments:        RFC 3739                                 |
|       | attributeValue:  text                                     |
+-------+-----------------------------------------------------------+
|     6 | Name:            CountryOfCitizenship                     |
|       | Identifiers:     id-pda-countryOfCitizenship              |
|       | OID:             1.3.6.1.5.5.7.9.4                        |
|       | DER:             06 08 2B 06 01 05 05 07 09 04            |
|       | Comments:        RFC 3739                                 |
|       | attributeValue:  text                                     |
+-------+-----------------------------------------------------------+
|     7 | Name:            CountryOfResidence                       |
|       | Identifiers:     id-pda-countryOfResidence                |
|       | OID:             1.3.6.1.5.5.7.9.5                        |
|       | DER:             06 08 2B 06 01 05 05 07 09 05            |
|       | Comments:        RFC 3739                                 |
|       | attributeValue:  text                                     |
+-------+-----------------------------------------------------------+
~~~~~~~~~~~
{: #fig-attrtype-additions title="Additional Entries in the C509 Attributes Registry"}
{: artwork-align="center"}

## C509 Extensions Registry {#extype}

This document updates the "C509 Extensions" registry in the registry group "CBOR Encoded X.509 (C509)" defined in {{I-D.ietf-cose-cbor-encoded-cert}}. The intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1], which are part of the Future Use interval [65536, 2^63 - 1], are set aside for PEN-scoped identifiers as specified in {{pen-id}}. IANA is requested to add the following note to the registry:

> Values in the intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1] are PEN-scoped identifiers as specified in [[this document]]. They are not assigned by IANA.

IANA is requested to add the following entries to the "C509 Extensions" registry:

~~~~~~~~~~~
+-------+-----------------------------------------------------------+
| Value | Extension                                                 |
+=======+===========================================================+
|    24 | Name:            Subject Directory Attributes             |
|       | Identifiers:     subjectDirectoryAttributes               |
|       | OID:             2.5.29.9                                 |
|       | DER:             06 03 55 1D 09                           |
|       | Comments:        RFC 5280                                 |
|       | extensionValue:  SubjectDirectoryAttributes               |
+-------+-----------------------------------------------------------+
|    39 | Name:            No Revocation Available                  |
|       | Identifiers:     noRevAvail                               |
|       | OID:             2.5.29.56                                |
|       | DER:             06 03 55 1D 38                           |
|       | Comments:        RFC 9608                                 |
|       | extensionValue:  NoRevAvail                               |
+-------+-----------------------------------------------------------+
~~~~~~~~~~~
{: #fig-extype-additions title="Additional Entries in the C509 Extensions Registry"}
{: artwork-align="center"}

## C509 Certificate Policies Registry {#CP}

This document updates the "C509 Certificate Policies" registry in the registry group "CBOR Encoded X.509 (C509)" defined in {{I-D.ietf-cose-cbor-encoded-cert}}. The intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1], which are part of the Future Use interval [65536, 2^63 - 1], are set aside for PEN-scoped identifiers as specified in {{pen-id}}. IANA is requested to add the following note to the registry:

> Values in the intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1] are PEN-scoped identifiers as specified in [[this document]]. They are not assigned by IANA.

## C509 Extended Key Usages Registry {#EKU}

This document updates the "C509 Extended Key Usages" registry in the registry group "CBOR Encoded X.509 (C509)" defined in {{I-D.ietf-cose-cbor-encoded-cert}}. The intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1], which are part of the Future Use interval [65536, 2^63 - 1], are set aside for PEN-scoped identifiers as specified in {{pen-id}}. IANA is requested to add the following note to the registry:

> Values in the intervals [2^31, 2^32 - 1] and [2^47, 2^48 - 1] are PEN-scoped identifiers as specified in [[this document]]. They are not assigned by IANA.

IANA is requested to add the following entries to the "C509 Extended Key Usages" registry. The iKEIntermediate OID is listed in the SMI Security for Mechanism IPSec Certificate Usage Codes registry in {{IANA.SMI}}.

~~~~~~~~~~~
+-------+--------------------------------------------------------------+
| Value | Extended Key Usage                                           |
+=======+==============================================================+
|    19 | Name:            IPsec IKE                                   |
|       | Identifiers:     id-kp-ipsecIKE                              |
|       | OID:             1.3.6.1.5.5.7.3.17                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 11               |
|       | Comments:        RFC 4945                                    |
+-------+--------------------------------------------------------------+
|    21 | Name:            IP Security IKE Intermediate                |
|       | Identifiers:     iKEIntermediate                             |
|       | OID:             1.3.6.1.5.5.8.2.2                           |
|       | DER:             06 08 2B 06 01 05 05 08 02 02               |
|       | Comments:        IANA SMI Numbers                            |
+-------+--------------------------------------------------------------+
|    22 | Name:            Document Signing                            |
|       | Identifiers:     id-kp-documentSigning                       |
|       | OID:             1.3.6.1.5.5.7.3.36                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 24               |
|       | Comments:        RFC 9336                                    |
+-------+--------------------------------------------------------------+
|    24 | Name:            EAP over LAN                                |
|       | Identifiers:     id-kp-eapOverLAN                            |
|       | OID:             1.3.6.1.5.5.7.3.14                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 0E               |
|       | Comments:        RFC 4334                                    |
+-------+--------------------------------------------------------------+
|    25 | Name:            CAPWAP Access Controller                    |
|       | Identifiers:     id-kp-capwapAC                              |
|       | OID:             1.3.6.1.5.5.7.3.18                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 12               |
|       | Comments:        RFC 5415                                    |
+-------+--------------------------------------------------------------+
|    26 | Name:            CAPWAP Wireless Termination Point           |
|       | Identifiers:     id-kp-capwapWTP                             |
|       | OID:             1.3.6.1.5.5.7.3.19                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 13               |
|       | Comments:        RFC 5415                                    |
+-------+--------------------------------------------------------------+
|    27 | Name:            BGPsec Router                               |
|       | Identifiers:     id-kp-bgpsec-router                         |
|       | OID:             1.3.6.1.5.5.7.3.30                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 1E               |
|       | Comments:        RFC 8209                                    |
+-------+--------------------------------------------------------------+
|    28 | Name:            RPC over TLS Client                         |
|       | Identifiers:     id-kp-rpcTLSClient                          |
|       | OID:             1.3.6.1.5.5.7.3.33                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 21               |
|       | Comments:        RFC 9289                                    |
+-------+--------------------------------------------------------------+
|    29 | Name:            RPC over TLS Server                         |
|       | Identifiers:     id-kp-rpcTLSServer                          |
|       | OID:             1.3.6.1.5.5.7.3.34                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 22               |
|       | Comments:        RFC 9289                                    |
+-------+--------------------------------------------------------------+
|    30 | Name:            JSON Web Token Signing                      |
|       | Identifiers:     id-kp-jwt                                   |
|       | OID:             1.3.6.1.5.5.7.3.37                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 25               |
|       | Comments:        RFC 9509                                    |
+-------+--------------------------------------------------------------+
|    31 | Name:            HTTP Content Encryption                     |
|       | Identifiers:     id-kp-httpContentEncrypt                    |
|       | OID:             1.3.6.1.5.5.7.3.38                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 26               |
|       | Comments:        RFC 9509                                    |
+-------+--------------------------------------------------------------+
|    32 | Name:            OAuth Access Token Signing                  |
|       | Identifiers:     id-kp-oauthAccessTokenSigning               |
|       | OID:             1.3.6.1.5.5.7.3.39                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 27               |
|       | Comments:        RFC 9509                                    |
+-------+--------------------------------------------------------------+
|    33 | Name:            Configuration Signing                       |
|       | Identifiers:     id-kp-configSigning                         |
|       | OID:             1.3.6.1.5.5.7.3.41                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 29               |
|       | Comments:        RFC 9809                                    |
+-------+--------------------------------------------------------------+
|    34 | Name:            Trust Anchor Configuration Signing          |
|       | Identifiers:     id-kp-trustAnchorConfigSigning              |
|       | OID:             1.3.6.1.5.5.7.3.42                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 2A               |
|       | Comments:        RFC 9809                                    |
+-------+--------------------------------------------------------------+
|    35 | Name:            Update Package Signing                      |
|       | Identifiers:     id-kp-updatePackageSigning                  |
|       | OID:             1.3.6.1.5.5.7.3.43                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 2B               |
|       | Comments:        RFC 9809                                    |
+-------+--------------------------------------------------------------+
|    36 | Name:            Safety Communication                        |
|       | Identifiers:     id-kp-safetyCommunication                   |
|       | OID:             1.3.6.1.5.5.7.3.44                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 2C               |
|       | Comments:        RFC 9809                                    |
+-------+--------------------------------------------------------------+
|    37 | Name:            OAuth Status List Signing                   |
|       | Identifiers:     id-kp-oauthStatusSigning                    |
|       | OID:             1.3.6.1.5.5.7.3.45                          |
|       | DER:             06 08 2B 06 01 05 05 07 03 2D               |
|       | Comments:        RFC-for-draft-ietf-oauth-status-list        |
+-------+--------------------------------------------------------------+
~~~~~~~~~~~
{: #fig-eku-additions title="Additional Entries in the C509 Extended Key Usages Registry"}
{: artwork-align="center"}

## CoAP Content-Formats Registry {#content-format}

IANA is requested to add an entry for `application/cose-c509-pubkey+cbor` to the "CoAP Content-Formats" registry in the registry group "Constrained RESTful Environments (CoRE) Parameters".

| Content Format | Content Coding | Media Type | ID | Reference |
|:---|:---|:---|:---|:---|
| application/cose-c509-pubkey+cbor | - | {{cose-c509-pubkey}} | TBD1 | [[this document]] |
{: #tab-format-ids title="CoAP Content-Format IDs"}

## Media Type Application Registry {#media-type}

IANA is requested to register the following media type in the "application" registry in the registry group "Media Types" with this document as reference, following the procedures in {{RFC6838}}.

### Media Type application/cose-c509-pubkey+cbor {#cose-c509-pubkey}

When the 'application/cose-c509-pubkey+cbor' media type is used, the data is a C509SubjectPublicKeyInfo structure.

Type name: application

Subtype name: cose-c509-pubkey+cbor

Required parameters: N/A

Optional parameters: N/A

Encoding considerations: binary

Security considerations: See the Security Considerations section of [[this document]].

Interoperability considerations: N/A

Published specification: [[this document]]

Applications that use this media type: Applications that employ COSE and use C509 public keys.

Fragment identifier considerations: N/A

Additional information:

* Deprecated alias names for this type: N/A
* Magic number(s): TBD2
* File extension(s): .c509
* Macintosh file type code(s): N/A

Person & email address to contact for further information: iesg@ietf.org

Intended usage: COMMON

Restrictions on usage: N/A

Author: COSE WG

Change controller: IETF

--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

The author would like to thank Gefei Li for the detailed review and comments.
