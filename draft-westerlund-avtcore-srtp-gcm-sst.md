---
title: "GCM-SST Authenticated Encryption in the Secure Real-time Transport Protocol (SRTP)"
abbrev: "GCM-SST for SRTP"
category: std

docname: draft-westerlund-avtcore-srtp-gcm-sst-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "ART"
workgroup: "Audio/Video Transport Core Maintenance"
keyword:
 - Cipher Suite
 - GCM-SST
 - SRTP

venue:
  group: "Audio/Video Transport Core Maintenance (AVTCORE)"
  type: "Working Group"
  mail: "avt@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/avt/"
  github: "gloinul/draft-westerlund-avtcore-srtp-gcm-sst"
  latest: "https://gloinul.github.io/draft-westerlund-avtcore-srtp-gcm-sst/draft-westerlund-avtcore-srtp-gcm-sst.html"

author:
 -
   ins: M. Westerlund
   name: Magnus Westerlund
   org: Ericsson
   email: magnus.westerlund@ericsson.com

 -
   ins: J. Preuß Mattsson
   name: John Preuß Mattsson
   org: Ericsson
   email: john.mattsson@ericsson.com

normative:

  RFC3550:
  RFC3711:
  RFC3830:
  RFC4568:
  RFC5116:
  RFC5234:
  RFC5764:
  RFC6188:
  RFC6904:
  I-D.mattsson-cfrg-aes-gcm-sst:

informative:

  RFC4771:
  RFC4303:
  RFC6479:

...

--- abstract

This document defines how the GCM-SST (Galois Counter Mode with Strong Secure Tags) Authenticated Encryption with Associated Data (AEAD) algorithm family can be used to provide confidentiality and data authentication in the Secure Real-time Transport Protocol (SRTP). GCM-SST addresses known weaknesses in AES-GCM for short authentication tags, making it well suited for media encryption use cases where low overhead is critical.


--- middle

# Introduction

The Secure Real-time Transport Protocol (SRTP) {{RFC3711}} is a profile of the Real-time Transport Protocol (RTP) {{RFC3550}}, which can provide confidentiality, message authentication, and replay protection to the RTP traffic and to the control traffic for RTP, the Real-time Transport Control Protocol (RTCP).

Authenticated Encryption with Associated Data (AEAD) {{RFC5116}} provides both confidentiality and integrity in a single cryptographic operation. This specification makes use of the GCM-SST (Galois Counter Mode with Strong Secure Tags) AEAD algorithm family defined in {{I-D.mattsson-cfrg-aes-gcm-sst}}.

AES-GCM is widely used but has known weaknesses when used with short authentication tags. The forgery probability for AES-GCM is significantly worse than ideal, and a successful forgery reveals the authentication subkey H, enabling an attacker to forge all subsequent messages. GCM-SST addresses these weaknesses by introducing an additional subkey H2 and deriving fresh subkeys H and H2 for each nonce, resulting in near-ideal forgery probabilities even for short tags and even after multiple forgery attempts.

Short authentication tags are common in media encryption. 32-bit tags are standard in most radio link layers, 64-bit tags are common in IoT transport and application layers, and 32-, 64-, and 80-bit tags are common in media encryption applications. Audio packets are small, numerous, and ephemeral, making them highly sensitive to cryptographic overhead. GCM-SST enables the use of short tags with strong security guarantees, avoiding the need to either use larger-than-necessary tags or fall back to slower constructions such as AES-CTR combined with HMAC.

This document defines how to use the AES-128-GCM-SST, AES-256-GCM-SST, and Rijndael-GCM-SST AEAD algorithms in SRTP and SRTCP, with authentication tag lengths of 48, 96, and 112 bits. The following cipher suites are defined:

~~~
  AEAD_AES_128_GCM_SST_6       AES-128 with a 6-octet (48-bit) tag
  AEAD_AES_128_GCM_SST_12      AES-128 with a 12-octet (96-bit) tag
  AEAD_AES_128_GCM_SST_14      AES-128 with a 14-octet (112-bit) tag
  AEAD_AES_256_GCM_SST_6       AES-256 with a 6-octet (48-bit) tag
  AEAD_AES_256_GCM_SST_12      AES-256 with a 12-octet (96-bit) tag
  AEAD_AES_256_GCM_SST_14      AES-256 with a 14-octet (112-bit) tag
  AEAD_RIJNDAEL_GCM_SST_6      Rijndael-256 with a 6-octet (48-bit) tag
  AEAD_RIJNDAEL_GCM_SST_12     Rijndael-256 with a 12-octet (96-bit) tag
  AEAD_RIJNDAEL_GCM_SST_14     Rijndael-256 with a 14-octet (112-bit) tag
~~~

When using cipher suites with 48-bit (6-octet) tags for SRTP, SRTCP uses 96-bit (12-octet) tags. This provides adequate security for the less frequent SRTCP packets while minimizing overhead for the more numerous SRTP packets.

The Rijndael-GCM-SST cipher suites use Rijndael-256 (256-bit key, 256-bit block) in counter mode as the keystream generator. Rijndael-256 uses a 28-octet (224-bit) nonce, which requires a different IV formation than the AES-based cipher suites (see {{rijndael-srtp-iv}} and {{rijndael-srtcp-iv}}).


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms have specific meanings in this document:

Instantiation:
: In AEAD, an instantiation is an (Encryption_key, salt) pair together with all data structures needed for it to function properly. In SRTP/SRTCP, each endpoint needs two instantiations of the AEAD algorithm for each master key: one for SRTP traffic and one for SRTCP traffic.

Invocation:
: SRTP/SRTCP data streams are broken into packets. Each packet is processed by a single invocation of the appropriate instantiation of the AEAD algorithm.

Associated Data:
: Data that is authenticated but not encrypted.

Plaintext:
: Data that is both encrypted and authenticated.

Raw Data:
: Data that is neither encrypted nor authenticated.


# Overview of the SRTP/SRTCP AEAD Security Architecture

SRTP/SRTCP AEAD security is based upon the following principles:

a. Both privacy and authentication are based upon the use of symmetric algorithms. An AEAD algorithm such as GCM-SST combines privacy and authentication into a single process.

b. A secret master key is shared by all participating endpoints. Any given master key MAY be used simultaneously by several endpoints to originate SRTP/SRTCP packets.

c. A Key Derivation Function (KDF) is applied to the shared master key to form separate encryption keys and salting keys for SRTP and SRTCP. Since AEAD algorithms combine encryption and authentication into a single process, AEAD algorithms do not make use of separate authentication keys.

d. The details of how the master key is established and shared between participants are outside the scope of this document. Any mechanism for rekeying an existing session is also outside the scope of this document.

e. Each time GCM-SST is invoked to encrypt and authenticate an SRTP or SRTCP packet, a new Initialization Vector (IV) is used. For AES-based cipher suites, SRTP combines the 4-octet Synchronization Source (SSRC) identifier, the 4-octet Rollover Counter (ROC), and the 2-octet Sequence Number (SEQ) with the 12-octet encryption salt to form a 12-octet IV (see {{srtp-iv}}). SRTCP combines the SSRC and 31-bit SRTCP index with the encryption salt to form a 12-octet IV (see {{srtcp-iv}}). For Rijndael-GCM-SST cipher suites, the same packet fields are combined with a 28-octet encryption salt to form a 28-octet IV (see {{rijndael-srtp-iv}} and {{rijndael-srtcp-iv}}).


# Generic AEAD Processing

## AEAD Invocation Inputs and Outputs

### Encrypt Mode

~~~
  Inputs:
    Encryption_key         Octet string, 16 or 32 octets
    Initialization_Vector  Octet string, 12 or 28 octets
    Associated_Data        Octet string of variable length
    Plaintext              Octet string of variable length

  Outputs:
    Ciphertext             Octet string, length =
                             length(Plaintext) + tag_length
~~~

The ciphertext consists of the encrypted Plaintext followed by the authentication tag.

### Decrypt Mode

~~~
  Inputs:
    Encryption_key         Octet string, 16 or 32 octets
    Initialization_Vector  Octet string, 12 or 28 octets
    Associated_Data        Octet string of variable length
    Ciphertext             Octet string of variable length

  Outputs:
    Plaintext              Octet string, length = length(Ciphertext) - tag_length
    Validity_Flag          Boolean, TRUE if valid, FALSE otherwise
~~~

## Handling of AEAD Authentication

All incoming packets MUST pass AEAD authentication before any other action takes place. Plaintext and Associated Data MUST NOT be released until the AEAD authentication tag has been validated. Should the AEAD tag prove to be invalid, the packet MUST be discarded.


# GCM-SST Processing for SRTP

## SRTP IV Formation for GCM-SST {#srtp-iv}

~~~
              0  0  0  0  0  0  0  0  0  0  1  1
              0  1  2  3  4  5  6  7  8  9  0  1
            +--+--+--+--+--+--+--+--+--+--+--+--+
            |00|00|    SSRC   |     ROC   | SEQ |---+
            +--+--+--+--+--+--+--+--+--+--+--+--+   |
                                                     |
            +--+--+--+--+--+--+--+--+--+--+--+--+   |
            |         Encryption Salt           |->(+)
            +--+--+--+--+--+--+--+--+--+--+--+--+   |
                                                     |
            +--+--+--+--+--+--+--+--+--+--+--+--+   |
            |       Initialization Vector       |<--+
            +--+--+--+--+--+--+--+--+--+--+--+--+
~~~
{: #fig-srtp-iv title="GCM-SST SRTP Initialization Vector Formation (AES)"}

The 12-octet IV used by AES-GCM-SST SRTP is formed by first concatenating 2 octets of zeroes, the 4-octet SSRC, the 4-octet rollover counter (ROC), and the 2-octet sequence number (SEQ). The resulting 12-octet value is then XORed with the 12-octet salt to form the IV.

## SRTP IV Formation for Rijndael-GCM-SST {#rijndael-srtp-iv}

~~~
            0 0 0 0 0 0 0 0 0 0 1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2
            0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |         0 (18 octets)             |  SSRC | ROC   |SEQ|---+
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   |
                                                                       |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   |
           |             Encryption Salt (28 octets)               |->(+)
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   |
                                                                       |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   |
           |          Initialization Vector (28 octets)            |<--+
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-rijndael-srtp-iv title="Rijndael-GCM-SST SRTP Initialization Vector Formation"}

The 28-octet IV used by Rijndael-GCM-SST SRTP is formed by first concatenating 18 octets of zeroes, the 4-octet SSRC, the 4-octet rollover counter (ROC), and the 2-octet sequence number (SEQ). The resulting 28-octet value is then XORed with the 28-octet salt to form the IV. The larger salt provides significantly greater security against precomputation and multi-key attacks compared to the AES-based cipher suites.

## Data Types in SRTP Packets

All SRTP packets MUST be both authenticated and encrypted. The data fields within RTP packets are broken into Associated Data, Plaintext, and Raw Data as follows:

Associated Data:
: The RTP header fields: version V (2 bits), padding flag P (1 bit), extension flag X (1 bit), CSRC count CC (4 bits), marker M (1 bit), Payload Type PT (7 bits), sequence number (16 bits), timestamp (32 bits), SSRC (32 bits), optional CSRC identifiers, and optional RTP extension.

Plaintext:
: The RTP payload, RTP padding (if used), and RTP pad count (if used).

Raw Data:
: The optional SRTP Master Key Identifier (MKI) and SRTP authentication tag (whose use is NOT RECOMMENDED).

~~~
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|V=2|P|X|  CC   |M|     PT      |       sequence number         |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|                           timestamp                           |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|           synchronization source (SSRC) identifier            |
 +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
A|            contributing source (CSRC) identifiers             |
A|                             ....                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|                   RTP extension (OPTIONAL)                    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
P|                          payload  ...                         |
P|                               +-------------------------------+
P|                               | RTP padding   | RTP pad count |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  P = Plaintext   A = Associated Data
~~~
{: #fig-srtp-pre title="RTP Packet before Authenticated Encryption"}

~~~
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|V=2|P|X|  CC   |M|     PT      |       sequence number         |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|                           timestamp                           |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|           synchronization source (SSRC) identifier            |
 +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
A|            contributing source (CSRC) identifiers             |
A|                             ....                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|                   RTP extension (OPTIONAL)                    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
C|                          cipher ...                           |
C|                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
R:                     SRTP MKI (OPTIONAL)                       :
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
R:           SRTP authentication tag (NOT RECOMMENDED)           :
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  C = Ciphertext   A = Associated Data   R = Raw Data
~~~
{: #fig-srtp-post title="SRTP Packet after Authenticated Encryption"}

## Handling Header Extensions

When {{RFC6904}} is in use, a separate keystream is generated to encrypt selected RTP header extension elements. For GCM-SST cipher suites using AES-128 keys, this keystream MUST be generated using the AES_128_CM transform. For GCM-SST cipher suites using AES-256 keys (including the Rijndael-GCM-SST cipher suites), the keystream MUST be generated using the AES_256_CM transform. The originator MUST perform any required header extension encryption before the AEAD algorithm is invoked.

Both encrypted and unencrypted header extensions are treated by the AEAD algorithm as Associated Data. The AEAD algorithm therefore provides integrity and authentication for header extensions but no additional privacy beyond what RFC 6904 provides.

## Prevention of SRTP IV Reuse {#srtp-iv-reuse}

To prevent IV reuse, the (ROC, SEQ, SSRC) triple MUST never be used twice with the same master key. A rekey MUST be performed before the (ROC, SEQ) pair cycles back to its original value. For a given master key, the set of all SSRC values MUST be partitioned into disjoint pools, one per originating endpoint, and each endpoint MUST only use SSRC values from its assigned pool.


# GCM-SST Processing for SRTCP

All SRTCP compound packets MUST be authenticated. SRTCP packet encryption is optional and indicated by a 1-bit Encryption flag located just before the 31-bit SRTCP index.

When using the AEAD_AES_128_GCM_SST_6, AEAD_AES_256_GCM_SST_6, or AEAD_RIJNDAEL_GCM_SST_6 cipher suites (which use 48-bit tags for SRTP), implementations MUST use 96-bit (12-octet) authentication tags for SRTCP packets. For all other cipher suites, the SRTCP tag length MUST match the SRTP tag length.

## SRTCP IV Formation for AES-GCM-SST {#srtcp-iv}

~~~
              0  1  2  3  4  5  6  7  8  9 10 11
            +--+--+--+--+--+--+--+--+--+--+--+--+
            |00|00|    SSRC   |00|00|0+SRTCP Idx|---+
            +--+--+--+--+--+--+--+--+--+--+--+--+   |
                                                     |
            +--+--+--+--+--+--+--+--+--+--+--+--+   |
            |         Encryption Salt           |->(+)
            +--+--+--+--+--+--+--+--+--+--+--+--+   |
                                                     |
            +--+--+--+--+--+--+--+--+--+--+--+--+   |
            |       Initialization Vector       |<--+
            +--+--+--+--+--+--+--+--+--+--+--+--+
~~~
{: #fig-srtcp-iv title="GCM-SST SRTCP Initialization Vector Formation (AES)"}

The 12-octet IV used by AES-GCM-SST SRTCP is formed by concatenating 2 octets of zeroes, the 4-octet SSRC, 2 octets of zeroes, a single "0" bit, and the 31-bit SRTCP index. The resulting 12-octet value is then XORed with the 12-octet salt to form the IV.

## SRTCP IV Formation for Rijndael-GCM-SST {#rijndael-srtcp-iv}

~~~
        0                   1                   2
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7
       +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
       |  0 (18 octets)                  |    SSRC   |00|0+SRTCPi|---+
       +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+   |
                                                                      |
       +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+   |
       |              Encryption Salt (28 octets)                 |->(+)
       +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+   |
                                                                      |
       +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+   |
       |           Initialization Vector (28 octets)             |<--+
       +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
~~~
{: #fig-rijndael-srtcp-iv title="Rijndael-GCM-SST SRTCP Initialization Vector Formation"}

The 28-octet IV used by Rijndael-GCM-SST SRTCP is formed by concatenating 18 octets of zeroes, the 4-octet SSRC, 2 octets of zeroes, a single "0" bit, and the 31-bit SRTCP index. The resulting 28-octet value is then XORed with the 28-octet salt to form the IV.

## Data Types in Encrypted SRTCP Packets

When the Encryption flag is set to 1:

Associated Data:
: Version V (2 bits), padding flag P (1 bit), reception report count RC (5 bits), Packet Type (8 bits), length (2 octets), SSRC (4 octets), Encryption flag (1 bit), and SRTCP index (31 bits).

Plaintext:
: All other data.

Raw Data:
: The optional SRTCP MKI and SRTCP authentication tag (NOT RECOMMENDED).

~~~
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|V=2|P|   RC    |  Packet Type  |            length             |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|           synchronization source (SSRC) of sender             |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
P|                         sender info                           :
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
P|                        report blocks ...                      :
 +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
P|                       additional packets ...                  :
 +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
A|1|                         SRTCP index                         |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
R:                  SRTCP MKI (OPTIONAL)                         :
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
R:           SRTCP authentication tag (NOT RECOMMENDED)          :
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  P = Plaintext   A = Associated Data   R = Raw Data
~~~
{: #fig-srtcp-enc title="AEAD SRTCP Inputs When Encryption Flag = 1"}

## Data Types in Unencrypted SRTCP Packets

When the Encryption flag is set to 0:

Plaintext:
: None.

Associated Data:
: All data except the optional SRTCP MKI and authentication tag.

Raw Data:
: The optional SRTCP MKI and SRTCP authentication tag (NOT RECOMMENDED).

~~~
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|V=2|P|   RC    |  Packet Type  |            length             |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|           synchronization source (SSRC) of sender             |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|                         sender info                           :
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A|                        report blocks ...                      :
 +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
A|                       additional packets ...                  :
 +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
A|0|                         SRTCP index                         |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
R:                  SRTCP MKI (OPTIONAL)                         :
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
R:              authentication tag (NOT RECOMMENDED)             :
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  A = Associated Data   R = Raw Data
~~~
{: #fig-srtcp-unenc title="AEAD SRTCP Inputs When Encryption Flag = 0"}

## Prevention of SRTCP IV Reuse

A new master key MUST be established before the 31-bit SRTCP index cycles back to its original value. The comments on SSRC management in {{srtp-iv-reuse}} also apply to SRTCP.


# Unneeded SRTP/SRTCP Fields

## SRTP/SRTCP Authentication Tag Field

The AEAD message authentication mechanism MUST be the primary message authentication mechanism. Additional SRTP/SRTCP authentication mechanisms SHOULD NOT be used, and the optional SRTP/SRTCP authentication tags are NOT RECOMMENDED and SHOULD NOT be present.

## RTP Padding

GCM-SST does not require data to be padded to a specific block size. It is RECOMMENDED that the RTP padding mechanism not be used unless necessary to disguise the length of the underlying Plaintext.


# Constraints on AEAD for SRTP and SRTCP

All AEAD algorithms used with SRTP/SRTCP MUST satisfy the following constraints:

| Parameter | Meaning | Value |
| A_MAX | Maximum Associated Data length | MUST be at least 12 octets |
| N_MIN | Minimum nonce (IV) length | 12 octets (AES) or 28 octets (Rijndael) |
| N_MAX | Maximum nonce (IV) length | 12 octets (AES) or 28 octets (Rijndael) |
| P_MAX | Maximum Plaintext length | See {{I-D.mattsson-cfrg-aes-gcm-sst}} |
| C_MAX | Maximum ciphertext length | P_MAX + tag_length |
{: title="AEAD Constraints for SRTP/SRTCP"}

Additional parameters:

- Maximum number of invocations for a given instantiation: SRTP MUST be at most 2^48; SRTCP MUST be at most 2^31.


# Key Derivation Functions

A Key Derivation Function (KDF) is used to derive all required encryption keys and salts from the shared master key. The AES-128-GCM-SST cipher suites MUST use the AES_128_CM_PRF KDF described in {{RFC3711}}. The AES-256-GCM-SST and Rijndael-GCM-SST cipher suites MUST use the AES_256_CM_PRF KDF described in {{RFC6188}}.

For the Rijndael-GCM-SST cipher suites, the KDF MUST derive a 256-bit encryption key and a 224-bit (28-octet) encryption salt. The master salt for Rijndael-GCM-SST is 224 bits.


# Summary of GCM-SST Cipher Suites in SRTP/SRTCP

The following GCM-SST cipher suites are defined for use with SRTP/SRTCP:

| Name | Key Size | Tag Size |
| AEAD_AES_128_GCM_SST_6  | 16 octets | 6 octets  |
| AEAD_AES_128_GCM_SST_12 | 16 octets | 12 octets |
| AEAD_AES_128_GCM_SST_14 | 16 octets | 14 octets |
| AEAD_AES_256_GCM_SST_6  | 32 octets | 6 octets  |
| AEAD_AES_256_GCM_SST_12 | 32 octets | 12 octets |
| AEAD_AES_256_GCM_SST_14 | 32 octets | 14 octets |
| AEAD_RIJNDAEL_GCM_SST_6  | 32 octets | 6 octets  |
| AEAD_RIJNDAEL_GCM_SST_12 | 32 octets | 12 octets |
| AEAD_RIJNDAEL_GCM_SST_14 | 32 octets | 14 octets |
{: title="GCM-SST Cipher Suites for SRTP/SRTCP"}

| Parameter | AEAD_AES_128_GCM_SST_6 |
| Master key length | 128 bits |
| Master salt length | 96 bits |
| Key Derivation Function | AES_128_CM_PRF {{RFC3711}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 48 bits |
{: title="AEAD_AES_128_GCM_SST_6 Crypto Suite"}

| Parameter | AEAD_AES_128_GCM_SST_12 |
| Master key length | 128 bits |
| Master salt length | 96 bits |
| Key Derivation Function | AES_128_CM_PRF {{RFC3711}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 96 bits |
{: title="AEAD_AES_128_GCM_SST_12 Crypto Suite"}

| Parameter | AEAD_AES_128_GCM_SST_14 |
| Master key length | 128 bits |
| Master salt length | 96 bits |
| Key Derivation Function | AES_128_CM_PRF {{RFC3711}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 112 bits |
{: title="AEAD_AES_128_GCM_SST_14 Crypto Suite"}

| Parameter | AEAD_AES_256_GCM_SST_6 |
| Master key length | 256 bits |
| Master salt length | 96 bits |
| Key Derivation Function | AES_256_CM_PRF {{RFC6188}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 48 bits |
{: title="AEAD_AES_256_GCM_SST_6 Crypto Suite"}

| Parameter | AEAD_AES_256_GCM_SST_12 |
| Master key length | 256 bits |
| Master salt length | 96 bits |
| Key Derivation Function | AES_256_CM_PRF {{RFC6188}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 96 bits |
{: title="AEAD_AES_256_GCM_SST_12 Crypto Suite"}

| Parameter | AEAD_AES_256_GCM_SST_14 |
| Master key length | 256 bits |
| Master salt length | 96 bits |
| Key Derivation Function | AES_256_CM_PRF {{RFC6188}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 112 bits |
{: title="AEAD_AES_256_GCM_SST_14 Crypto Suite"}

| Parameter | AEAD_RIJNDAEL_GCM_SST_6 |
| Master key length | 256 bits |
| Master salt length | 224 bits |
| Key Derivation Function | AES_256_CM_PRF {{RFC6188}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 48 bits |
{: title="AEAD_RIJNDAEL_GCM_SST_6 Crypto Suite"}

| Parameter | AEAD_RIJNDAEL_GCM_SST_12 |
| Master key length | 256 bits |
| Master salt length | 224 bits |
| Key Derivation Function | AES_256_CM_PRF {{RFC6188}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 96 bits |
{: title="AEAD_RIJNDAEL_GCM_SST_12 Crypto Suite"}

| Parameter | AEAD_RIJNDAEL_GCM_SST_14 |
| Master key length | 256 bits |
| Master salt length | 224 bits |
| Key Derivation Function | AES_256_CM_PRF {{RFC6188}} |
| Maximum key lifetime (SRTP) | 2^48 packets |
| Maximum key lifetime (SRTCP) | 2^31 packets |
| AEAD authentication tag length | 112 bits |
{: title="AEAD_RIJNDAEL_GCM_SST_14 Crypto Suite"}

# Security Considerations

## Handling of Security-Critical Parameters

The following security-critical parameters must be handled properly:

- The master salt, if kept secret, MUST be properly erased when no longer needed.

- The secret master key and all keys derived from it MUST be kept secret and MUST be properly erased when no longer needed.

- Each time a rekey occurs, the initial values of both the 31-bit SRTCP index and the 48-bit SRTP packet index (ROC\|\|SEQ) MUST be saved to prevent IV reuse.

- Processing MUST cease if either the 31-bit SRTCP index or the 48-bit SRTP packet index cycles back to its initial value. Processing MUST NOT resume until a new session has been established using a new master key.

- GCM-SST MUST be used in a nonce-respecting setting. For a given key, a nonce MUST only be used once in the encryption function and only once in a successful decryption function call. Nonce reuse enables universal forgery.

## Size of the Authentication Tag

The GCM-SST tag_length SHOULD NOT be smaller than 4 bytes. Unlike AES-GCM, GCM-SST provides near-ideal forgery probabilities even for short tags, making 48-bit tags suitable for applications such as audio packet encryption where overhead is critical. The 96-bit and 112-bit tag lengths provide higher security margins suitable for most other SRTP use cases. Implementations MUST use the tag length associated with the negotiated cipher suite and MUST NOT truncate or extend the tag.

## Replay Protection

GCM-SST MUST be used with replay protection. The SRTP sequence number and rollover counter, or the SRTCP index, provide the basis for replay protection. For examples of replay protection mechanisms, see {{RFC4303}} and {{RFC6479}}.

# IANA Considerations

## SDES

"Session Description Protocol (SDP) Security Descriptions for Media Streams" {{RFC4568}} defines SRTP crypto suites. IANA is requested to register the following crypto suites in the "SRTP Crypto Suite Registrations" subregistry:

~~~
  srtp-crypto-suite-ext = "AEAD_AES_128_GCM_SST_6"  /
                          "AEAD_AES_128_GCM_SST_12" /
                          "AEAD_AES_128_GCM_SST_14" /
                          "AEAD_AES_256_GCM_SST_6"  /
                          "AEAD_AES_256_GCM_SST_12" /
                          "AEAD_AES_256_GCM_SST_14" /
                          "AEAD_RIJNDAEL_GCM_SST_6"  /
                          "AEAD_RIJNDAEL_GCM_SST_12" /
                          "AEAD_RIJNDAEL_GCM_SST_14" /
                          srtp-crypto-suite-ext
~~~

## DTLS-SRTP

DTLS-SRTP {{RFC5764}} defines SRTP protection profiles. IANA is requested to register the following SRTP protection profiles:

~~~
  SRTP_AEAD_AES_128_GCM_SST_6   = {0x00, TBD1}
  SRTP_AEAD_AES_128_GCM_SST_12  = {0x00, TBD2}
  SRTP_AEAD_AES_128_GCM_SST_14  = {0x00, TBD3}
  SRTP_AEAD_AES_256_GCM_SST_6   = {0x00, TBD4}
  SRTP_AEAD_AES_256_GCM_SST_12  = {0x00, TBD5}
  SRTP_AEAD_AES_256_GCM_SST_14  = {0x00, TBD6}
  SRTP_AEAD_RIJNDAEL_GCM_SST_6  = {0x00, TBD8}
  SRTP_AEAD_RIJNDAEL_GCM_SST_12 = {0x00, TBD9}
  SRTP_AEAD_RIJNDAEL_GCM_SST_14 = {0x00, TBD10}
~~~

The SRTP transform parameters for each protection profile are as follows:

~~~
  SRTP_AEAD_AES_128_GCM_SST_6
    cipher:                 AES_128_GCM_SST
    cipher_key_length:      128 bits
    cipher_salt_length:     96 bits
    aead_auth_tag_length:   6 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets

  SRTP_AEAD_AES_128_GCM_SST_12
    cipher:                 AES_128_GCM_SST
    cipher_key_length:      128 bits
    cipher_salt_length:     96 bits
    aead_auth_tag_length:   12 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets

  SRTP_AEAD_AES_128_GCM_SST_14
    cipher:                 AES_128_GCM_SST
    cipher_key_length:      128 bits
    cipher_salt_length:     96 bits
    aead_auth_tag_length:   14 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets

  SRTP_AEAD_AES_256_GCM_SST_6
    cipher:                 AES_256_GCM_SST
    cipher_key_length:      256 bits
    cipher_salt_length:     96 bits
    aead_auth_tag_length:   6 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets

  SRTP_AEAD_AES_256_GCM_SST_12
    cipher:                 AES_256_GCM_SST
    cipher_key_length:      256 bits
    cipher_salt_length:     96 bits
    aead_auth_tag_length:   12 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets

  SRTP_AEAD_AES_256_GCM_SST_14
    cipher:                 AES_256_GCM_SST
    cipher_key_length:      256 bits
    cipher_salt_length:     96 bits
    aead_auth_tag_length:   14 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets

  SRTP_AEAD_RIJNDAEL_GCM_SST_6
    cipher:                 RIJNDAEL_256_GCM_SST
    cipher_key_length:      256 bits
    cipher_salt_length:     224 bits
    aead_auth_tag_length:   6 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets

  SRTP_AEAD_RIJNDAEL_GCM_SST_12
    cipher:                 RIJNDAEL_256_GCM_SST
    cipher_key_length:      256 bits
    cipher_salt_length:     224 bits
    aead_auth_tag_length:   12 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets

  SRTP_AEAD_RIJNDAEL_GCM_SST_14
    cipher:                 RIJNDAEL_256_GCM_SST
    cipher_key_length:      256 bits
    cipher_salt_length:     224 bits
    aead_auth_tag_length:   14 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and at most 2^48 SRTP packets
~~~

## MIKEY

In accordance with {{RFC3830}}, IANA is requested to add the following to the "Encryption algorithm (Value 0)" subregistry:

| SRTP Encr. Algorithm | Value | Default Session Encr. Key Length | Default Auth. Tag Length |
| AES-GCM-SST | TBD7 | 16 octets | variable |
| RIJNDAEL-256-GCM-SST | TBD11 | 32 octets | variable |
{: title="MIKEY Encryption Algorithm Registration"}


# Parameters for Use with MIKEY

MIKEY specifies the algorithm family separately from the key length and authentication tag length.

| AEAD Algorithm | Encryption Algorithm | Encryption Key Length | AEAD Auth. Tag Length |
| AEAD_AES_128_GCM_SST_6  | AES-GCM-SST | 16 octets | 6 octets  |
| AEAD_AES_128_GCM_SST_12 | AES-GCM-SST | 16 octets | 12 octets |
| AEAD_AES_128_GCM_SST_14 | AES-GCM-SST | 16 octets | 14 octets |
| AEAD_AES_256_GCM_SST_6  | AES-GCM-SST | 32 octets | 6 octets  |
| AEAD_AES_256_GCM_SST_12 | AES-GCM-SST | 32 octets | 12 octets |
| AEAD_AES_256_GCM_SST_14 | AES-GCM-SST | 32 octets | 14 octets |
| AEAD_RIJNDAEL_GCM_SST_6  | RIJNDAEL-256-GCM-SST | 32 octets | 6 octets  |
| AEAD_RIJNDAEL_GCM_SST_12 | RIJNDAEL-256-GCM-SST | 32 octets | 12 octets |
| AEAD_RIJNDAEL_GCM_SST_14 | RIJNDAEL-256-GCM-SST | 32 octets | 14 octets |
{: title="Mapping MIKEY Parameters to GCM-SST AEAD Algorithms"}


--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank the authors of RFC 7714, David McGrew and Kevin Igoe, whose document structure this specification closely follows. The authors also thank the authors of {{I-D.mattsson-cfrg-aes-gcm-sst}}, Matthew Campagna, Alexander Maximov, and John Preuß Mattsson, for defining the GCM-SST algorithm.
