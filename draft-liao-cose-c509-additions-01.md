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
  RFC8949:
  RFC9090:
  I-D.ietf-cose-cbor-encoded-cert:
  IANA.cbor-tags:
  X.690:
    title: ASN.1 encoding rules. Specification of Basic Encoding Rules (BER), Canonical Encoding Rules (CER) and Distinguished Encoding Rules (DER)
    target: https://www.itu.int/rec/T-REC-X.690

informative:

--- abstract

This document defines additions to CBOR Encoded X.509 Certificates (C509).

This document defines a new C509SubjectPublicKeyInfo type, a CBOR representation of the X.509 SubjectDirectoryAttributes extension, and a mechanism that allows organizations assigned a Private Enterprise Number (PEN) to define organization-specific integer identifiers without registering each identifier in the C509 RDN attribute type, attribute type, extension ID, certificate policy, or extended key usage registries.

This document also defines textual encoding labels for C509 objects using the textual encoding conventions specified in RFC 7468.

--- middle

# Introduction {#intro}

{{I-D.ietf-cose-cbor-encoded-cert}} defines C509 certificate structure, a compact CBOR encoding of X.509 certificates. C509 supports a large subset of the X.509 certificate profile defined in {{RFC5280}} and is designed for use in constrained environments as well as general certificate-based protocols.

The base C509 specification defines the public key as fields within the certificate structure. It also defines compact integer identifiers for selected X.509 RDN attribute types, certification request (CR) attribute types, extension IDs, certificate policies, and extended key usages.

This document defines additions to the C509 specification in four areas.

- Defines a new structure C509SubjectPublicKeyInfo which corresponds to SubjectPublicKeyInfo in X.509 {{Section 4.1 of RFC5280}}. This permits a C509 public key to be represented independently of a complete C509 certificate.

- Defines the C509 representation of the X.509 SubjectDirectoryAttributes extension specified in {{Section 4.2.1.8 of RFC5280}}. The "C509 CR Attributes" registry is renamed to "C509 Attributes", which is a general registry for attributes, and is extended with entries for subject directory attributes.

- Defines PEN-scoped identifiers. An organization that has been assigned a Private Enterprise Number (PEN) can assign integer values within its own namespace. Such identifiers can be used for RDN attribute types, attribute types, extension IDs, certificate policies, and extended key usages without requiring each identifier to be registered in a C509 IANA registry.

- Defines PEM labels for C509 objects using the textual encoding framework specified by {{RFC7468}}.

This document is intended to be used together with {{I-D.ietf-cose-cbor-encoded-cert}}. It does not define a new certificate format; it adds types and encoding mechanisms to the C509 data model.

# Notational Conventions {#notation}

{::boilerplate bcp14-tagged}

# C509 Public Key {#publickey}

This document defines the following reusable C509 subject public key type:

~~~~~~~~~~~ cddl
C509SubjectPublicKeyInfo = [
  subjectPublicKeyAlgorithm : AlgorithmIdentifier,
  subjectPublicKey          : Defined
]
~~~~~~~~~~~
{: sourcecode-name="c509additions.cddl"}
{: #fig-C509SubjectPublicKeyInfoCDDL title="CDDL for C509SubjectPublicKeyInfo"}

The subjectPublicKeyAlgorithm field has the same encoding and semantics as the subjectPublicKeyAlgorithm field of TBSCertificate in {{I-D.ietf-cose-cbor-encoded-cert}}. The subjectPublicKey field has the same encoding and semantics as the subjectPublicKey field of TBSCertificate in {{I-D.ietf-cose-cbor-encoded-cert}}.

In particular, algorithm-specific C509 encodings defined by {{I-D.ietf-cose-cbor-encoded-cert}} apply equally when the public key is represented using C509SubjectPublicKeyInfo. For example, the optimized RSA encoding and elliptic-curve public key encoding apply to C509SubjectPublicKeyInfo.

# Extensions {#extensions}

## Extension Subject Directory Attributes {#ext-SubjectDirectoryAttributes}

X.509 defines the SubjectDirectoryAttributes extension in {{Section 4.2.1.8 of RFC5280}}. This document defines a C509 representation of that extension. The extension value is represented as SubjectDirectoryAttributes:

~~~~~~~~~~~ cddl
SubjectDirectoryAttributes = [
  + SubjectDirectoryAttribute
]

SubjectDirectoryAttribute = 
  (( attributeType: int,  attributeValue: Defined ) //
   ( attributeType: ~oid, attributeValue: bytes ))

~~~~~~~~~~~
{: sourcecode-name="c509additions.cddl"}
{: #fig-SubjectDirectoryAttributes title="CDDL for SubjectDirectoryAttributes"}

Each 'attributeType' is encoded either as a CBOR int (see {{attrtype}}) or as an unwrapped CBOR OID {{RFC9090}}.

- If 'attributeType' is encoded as a CBOR int, it is followed by a CBOR item of any type other than undefined. The specific type of the item is determined by 'attributeType'.
- If 'attributeType' is encoded as an unwrapped CBOR OID, it is followed by the DER-encoded 'values' (which is SET OF AttributeValue) encoded as a CBOR byte string.

### DateOfBirth

The personal data attribute 'DateOfBirth' is defined in {{RFC3739}} as 'GeneralizedTime'. In C509, the 'attributeValue' field has type CBOR text represented as 'YYYYMMDD', where the first 4 digits are the year, followed by 2 digits for the month and 2 digits for the day.

### PlaceOfBirth

The personal data attribute 'PlaceOfBirth' is defined in {{RFC3739}} as 'DirectoryString'. In C509, the 'attributeValue' field has type PlaceOfBirth. A UTF8 String is encoded as CBOR text, and a Printable String is tagged with number 121 (alternative 0, as defined in {{IANA.cbor-tags}}). All other string types are not supported. For native certificates and certification requests, only UTF8 String is allowed.

PlaceOfBirth = text / #6.121(text)

### Gender

The personal data attribute 'Gender' is defined in {{RFC3739}} as 'PrintableString'. In C509, the 'attributeValue' field has type CBOR text.

### CountryOfCitizenship

The personal data attribute 'CountryOfCitizenship' is defined in {{RFC3739}} as 'PrintableString'. In C509, only a single CountryOfCitizenship value is supported. The 'attributeValue' field has type CBOR text.

### CountryOfResidence

The personal data attribute 'CountryOfResidence' is defined in {{RFC3739}} as 'PrintableString'. In C509, the 'attributeValue' field has type CBOR text.

# PEN-Scoped Identifiers {#pen-id}

{{I-D.ietf-cose-cbor-encoded-cert}} defines integer identifiers for commonly used RDN attributes, CR attributes, extensions, extended key usages, and certificate policies, and this document registers integer identifiers for subject directory attributes in the general "C509 Attributes" registry (formerly "C509 CR Attributes"). These integer identifiers provide compact encodings.

An organization may also need identifiers that are meaningful only within its own ecosystem. Requiring each such identifier to be allocated from an IANA registry is unnecessary when the organization already has a globally unique namespace through a Private Enterprise Number.

This document therefore reserves two intervals for PEN-scoped identifiers for the RDN attributes, attributes (for example, CR attributes and subject directory attributes), extensions, extended key usages, and certificate policies. Both the PEN and the internal ID are represented as 'bn ... b1 b0', where bn is the most significant bit and b0 is the least significant bit.

- Interval [2^31, 2^32 - 1]:

  For PEN in [0, 2^21 - 1], and internal ID: [0, 2^10 - 1]

  The integer value is constructed as follows:

  ```
  b'1' || 21-bit PEN || 10-bit internal ID
  ```

- Interval [2^47, 2^48-1]:

  For all identifiers that cannot be represented in the interval [2^31, 2^32 - 1]:

  - PEN in the interval [2^21, 2^32 -1] and internal ID in [0, 2^15 - 1]
  - PEN in [0, 2^21 - 1] and internal ID in [2^10, 2^15 - 1]

  The integer value is constructed as follows:

  ```
  b'1' || 32-bit PEN || 15-bit internal ID
  ```

# Textual Encoding (PEM) {#pem}

This section defines textual (PEM) encodings for C509 structures, following the conventions of {{RFC7468}}. The textual encodings use base64-encoded binary data enclosed between "-----BEGIN LABEL-----" and "-----END LABEL-----" lines and carry CBOR-encoded C509 objects rather than DER-encoded ASN.1 objects.

## General Rules {#pem-general-rules}

Each C509 PEM block MUST conform to the following rules:

- The binary payload is the CBOR encoding of the structure. CBOR encoding MUST be deterministic as specified in {{Sections 4.2.1 and 4.2.2 of RFC8949}}.

- The binary payload is base64-encoded according to {{Section 4 of RFC4648}} (standard Base64 with padding characters `=`). Parsers SHOULD ignore whitespace and other non-base64 characters as per {{Section 2 of RFC7468}}.

- The encapsulation boundaries follow {{Section 2 of RFC7468}}:

  ~~~
  -----BEGIN LABEL-----
  (base64-encoded CBOR payload)
  -----END LABEL-----
  ~~~

- No header fields (as described in {{Section 2 of RFC7468}}) are defined for any of the labels in this document. Implementations MUST ignore any header fields encountered.

- Explanatory text before the pre-encapsulation boundary is allowed. Implementations MUST NOT require or rely on such text.

## C509 Certificate {#pem-c509-cert}

PEM label: `C509 CERTIFICATE`

The payload is a CBOR-encoded `C509Certificate` as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN C509 CERTIFICATE-----
(base64-encoded CBOR C509Certificate)
-----END C509 CERTIFICATE-----
~~~
{: #fig-pem-c509-cert title="C509 Certificate PEM Encoding"}

## COSE_C509 {#pem-cose-c509}

PEM label: `COSE C509`

The payload is a CBOR-encoded `COSE_C509` as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN COSE C509-----
(base64-encoded CBOR COSE_C509)
-----END COSE C509-----
~~~
{: #fig-pem-cose-c509 title="COSE_C509 PEM Encoding"}

## C509 Certification Request {#pem-c509-csr}

PEM label: `C509 CERTIFICATE REQUEST`

The payload is a CBOR-encoded `C509CertificationRequest` as defined in {{I-D.ietf-cose-cbor-encoded-cert, Section 4}}.

~~~
-----BEGIN C509 CERTIFICATE REQUEST-----
(base64-encoded CBOR C509CertificationRequest)
-----END C509 CERTIFICATE REQUEST-----
~~~
{: #fig-pem-c509-csr title="C509 Certification Request PEM Encoding"}

## C509 Public Key {#pem-c509-pubkey}

PEM label: `C509 PUBLIC KEY`

The payload is a CBOR-encoded `C509SubjectPublicKeyInfo` as defined in {{publickey}}.

~~~
-----BEGIN C509 PUBLIC KEY-----
(base64-encoded CBOR C509SubjectPublicKeyInfo)
-----END C509 PUBLIC KEY-----
~~~
{: #fig-pem-c509-pubkey title="C509 Public Key PEM Encoding"}

## C509 Private Key {#pem-c509-privkey}

PEM label: `C509 PRIVATE KEY`

The payload is a CBOR-encoded `C509PrivateKey` as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN C509 PRIVATE KEY-----
(base64-encoded CBOR C509PrivateKey)
-----END C509 PRIVATE KEY-----
~~~
{: #fig-pem-c509-privkey title="C509 Private Key PEM Encoding"}

## C509 PEM {#pem-c509-pem}

PEM label: `C509 PEM`

The payload is a CBOR-encoded `C509PEM` as defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~
-----BEGIN C509 PEM-----
(base64-encoded CBOR C509PEM)
-----END C509 PEM-----
~~~
{: #fig-pem-c509-pem title="C509 PEM Encoding"}

## PEM Label Summary {#pem-label-summary}

The following table summarizes all PEM labels defined in this document and compares them with RFC 7468 labels for equivalent DER-encoded structures.

| PEM Label | C509 Structure | Defined in | RFC 7468 Label |
|:---|:---|:---|:---|
| `C509 CERTIFICATE` | `C509Certificate` | {{I-D.ietf-cose-cbor-encoded-cert}} | `CERTIFICATE` |
| `COSE C509` | `COSE_C509` | {{I-D.ietf-cose-cbor-encoded-cert}} | N/A |
| `C509 CERTIFICATE REQUEST` | `C509CertificationRequest` | {{I-D.ietf-cose-cbor-encoded-cert}} | `CERTIFICATE REQUEST` |
| `C509 PUBLIC KEY` | `C509SubjectPublicKeyInfo` | [[this document]] | `PUBLIC KEY` |
| `C509 PRIVATE KEY` | `C509PrivateKey` | {{I-D.ietf-cose-cbor-encoded-cert}} | `PRIVATE KEY` |
| `C509 PEM` | `C509PEM` | {{I-D.ietf-cose-cbor-encoded-cert}} | N/A |
{: #tab-pem-labels title="PEM Labels Defined in This Document"}

{::comment}
Note to the author: ask the authors of draft-bonnell-lamps-pem-label-registry to add the
PEM labels defined in this document to the initial contents of their "PEM Labels" registry.
{:/comment}

# Security Considerations {#security}

TODO

# IANA Considerations {#iana}

## C509 RDN Attributes Registry {#rdnatttype}

This document updates the following ranges in the "C509 RDN Attributes" registry in the registry group "CBOR Encoded X.509 (C509)".

The ranges [2^31, 2^32 -1] and [2^47, 2^48 -1] are reserved for PEN identifiers as in {{pen-id}}.

## C509 Attributes Registry {#attrtype}

This document updates the "C509 CR Attributes" registry in the registry group "CBOR Encoded X.509 (C509)" defined in {{I-D.ietf-cose-cbor-encoded-cert}} as follows:

- The registry is renamed from "C509 CR Attributes" to "C509 Attributes". The registry is a general registry for attributes (the Attribute type, which pairs an attribute type with attribute values), wherever they are used in C509 structures. Examples include attributes in C509 certification requests and attributes in the SubjectDirectoryAttributes extension (see {{ext-SubjectDirectoryAttributes}}).

- The column heading "CR Attribute" is renamed to "Attribute".

- The ranges [2^31, 2^32 -1] and [2^47, 2^48 -1] are reserved for PEN identifiers as in {{pen-id}}.

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
|       | attributeValue:  text                                     |
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

This document updates the following ranges in the "C509 Extensions" registry in the registry group "CBOR Encoded X.509 (C509)".

The ranges [2^31, 2^32 -1] and [2^47, 2^48 -1] are reserved for PEN identifiers as in {{pen-id}}.

## C509 Certificate Policies Registry {#CP}

This document updates the following ranges in the "C509 Certificate Policies" registry in the registry group "CBOR Encoded X.509 (C509)".

The ranges [2^31, 2^32 -1] and [2^47, 2^48 -1] are reserved for PEN identifiers as in {{pen-id}}.

## C509 Extended Key Usages Registry {#EKU}

This document updates the following ranges in the "C509 Extended Key Usages" registry in the registry group "CBOR Encoded X.509 (C509)".

The ranges [2^31, 2^32 -1] and [2^47, 2^48 -1] are reserved for PEN identifiers as in {{pen-id}}.

## CoAP Content-Formats Registry {#content-format}

IANA is requested to add an entry for `application/cose-c509-pubkey+cbor` to the "CoAP Content-Formats" registry in the registry group "Constrained RESTful Environments (CoRE) Parameters".

~~~~~~~~~~~
+---------------------------+---------+-----------+-------+------------+
| Content                   | Content | Media     | ID    | Reference  |
| Format                    | Coding  | Type      |       |            |
+===========================+=========+===========+=======+============+
| application/              | -       | [[link    | TBD1  | [[this     |
| cose-c509-pubkey+cbor     |         | to TODO]] |       | document]] |
+---------------------------+---------+-----------+-------+------------+
~~~~~~~~~~~
{: #fig-format-ids title="CoAP Content-Format IDs"}

## Media Type Application Registry

IANA is requested to assign the following entries into the "application" registry in the registry group "Media Types" with this document as reference.

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

Applications that use this media type: Applications that employ COSE and use C509 as a certificate type.

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
