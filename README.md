# TOR_rend-spec-v2
Tor Rendezvous Specification

0. Overview and preliminaries

      The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
      NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
      "OPTIONAL" in this document are to be interpreted as described in
      RFC 2119.

   Read
   https://svn.torproject.org/svn/projects/design-paper/tor-design.html#sec:rendezvous
   before you read this specification. It will make more sense.

   Rendezvous points provide location-hidden services (server 
   anonymity) for the onion routing network. With rendezvous points,
   Bob can offer a TCP service (say, a webserver) via the onion
   routing network, without revealing the IP of that service.

   Bob does this by anonymously advertising a public key for his
   service, along with a list of onion routers to act as "Introduction
   Points" for his service.  He creates forward circuits to those
   introduction points, and tells them about his service.  To
   connect to Bob, Alice first builds a circuit to an OR to act as
   her "Rendezvous Point." She then connects to one of Bob's chosen
   introduction points, and asks it to tell him about her Rendezvous
   Point (RP).  If Bob chooses to answer, he builds a circuit to her
   RP, and tells it to connect him to Alice.  The RP joins their
   circuits together, and begins relaying cells.  Alice's 'BEGIN'
   cells are received directly by Bob's OP, which passes data to
   and from the local server implementing Bob's service.

   Below we describe a network-level specification of this service,
   along with interfaces to make this process transparent to Alice
   (so long as she is using an OP).

0.1. Notation, conventions and prerequisites

   In the specifications below, we use the same notation and terminology
   as in "tor-spec.txt".  The service specified here also requires the
   existence of an onion routing network as specified in that file.

        H(x) is a SHA1 digest of x.
        PKSign(SK,x) is a PKCS.1-padded RSA signature of x with SK.
        PKEncrypt(SK,x) is a PKCS.1-padded RSA encryption of x with SK.
        Public keys are all RSA, and encoded in ASN.1.
        All integers are stored in network (big-endian) order.
        All symmetric encryption uses AES in counter mode, except where
            otherwise noted.

   In all discussions, "Alice" will refer to a user connecting to a
   location-hidden service, and "Bob" will refer to a user running a
   location-hidden service.

   An OP is (as defined elsewhere) an "Onion Proxy" or Tor client.

   An OR is (as defined elsewhere) an "Onion Router" or Tor server.

   An "Introduction point" is a Tor server chosen to be Bob's medium-term
   'meeting place'.  A "Rendezvous point" is a Tor server chosen by Alice to
   be a short-term communication relay between her and Bob.  All Tor servers
   potentially act as introduction and rendezvous points.

0.2. Protocol outline

   1. Bob->Bob's OP: "Offer IP:Port as public-key-name:Port". [configuration]
      (We do not specify this step; it is left to the implementor of
      Bob's OP.)

   2. Bob's OP generates a long-term keypair.

   3. Bob's OP->Introduction point via Tor: [introduction setup]
        "This public key is (currently) associated to me."

   4. Bob's OP->directory service via Tor: publishes Bob's service descriptor
      [advertisement]
        "Meet public-key X at introduction point A, B, or C." (signed)

   5. Out of band, Alice receives a z.onion:port address.
      She opens a SOCKS connection to her OP, and requests z.onion:port.

   6. Alice's OP retrieves Bob's descriptor via Tor. [descriptor lookup.]

   7. Alice's OP chooses a rendezvous point, opens a circuit to that
      rendezvous point, and establishes a rendezvous circuit. [rendezvous
      setup.]

   8. Alice connects to the Introduction point via Tor, and tells it about
      her rendezvous point.  (Encrypted to Bob.)  [Introduction 1]

   9. The Introduction point passes this on to Bob's OP via Tor, along the
      introduction circuit. [Introduction 2]

  10. Bob's OP decides whether to connect to Alice, and if so, creates a
      circuit to Alice's RP via Tor.  Establishes a shared circuit.
      [Rendezvous 1]

  11. The Rendezvous point forwards Bob's confirmation to Alice's OP.
      [Rendezvous 2]

  12. Alice's OP sends begin cells to Bob's OP.  [Connection]

0.3. Constants and new cell types

  Relay cell types
      32 -- RELAY_COMMAND_ESTABLISH_INTRO
      33 -- RELAY_COMMAND_ESTABLISH_RENDEZVOUS
      34 -- RELAY_COMMAND_INTRODUCE1
      35 -- RELAY_COMMAND_INTRODUCE2
      36 -- RELAY_COMMAND_RENDEZVOUS1
      37 -- RELAY_COMMAND_RENDEZVOUS2
      38 -- RELAY_COMMAND_INTRO_ESTABLISHED
      39 -- RELAY_COMMAND_RENDEZVOUS_ESTABLISHED
      40 -- RELAY_COMMAND_INTRODUCE_ACK

0.4. Version overview

   There are several parts in the hidden service protocol that have
   changed over time, each of them having its own version number, whereas
   other parts remained the same. The following list of potentially
   versioned protocol parts should help reduce some confusion:

   - Hidden service descriptor: the binary-based v0 was the default for a
     long time, and an ASCII-based v2 has been added by proposal 114. The
     v0 descriptor format has been deprecated in 0.2.2.1-alpha. See 1.3.

   - Hidden service descriptor propagation mechanism: currently related to
     the hidden service descriptor version -- v0 publishes to the original
     hs directory authorities, whereas v2 publishes to a rotating subset
     of relays with the "HSDir" flag; see 1.4 and 1.6.

   - Introduction protocol for how to generate an introduction cell:
     v0 specified a nickname for the rendezvous point and assumed the
     relay would know about it, whereas v2 now specifies IP address,
     port, and onion key so the relay doesn't need to already recognize
     it. See 1.8.

1. The Protocol

1.1. Bob configures his local OP.

   We do not specify a format for the OP configuration file.  However,
   OPs SHOULD allow Bob to provide more than one advertised service
   per OP, and MUST allow Bob to specify one or more virtual ports per
   service.  Bob provides a mapping from each of these virtual ports
   to a local IP:Port pair.

1.2. Bob's OP establishes his introduction points.

   The first time the OP provides an advertised service, it generates
   a public/private keypair (stored locally).

   The OP chooses a small number of Tor servers as introduction points.
   The OP establishes a new introduction circuit to each introduction
   point.  These circuits MUST NOT be used for anything but hidden service
   introduction.  To establish the introduction, Bob sends a
   RELAY_COMMAND_ESTABLISH_INTRO cell, containing:

        KL   Key length                             [2 octets]
        PK   Bob's public key or service key        [KL octets]
        HS   Hash of session info                   [20 octets]
        SIG  Signature of above information         [variable]

   KL is the length of PK, in octets.

   To prevent replay attacks, the HS field contains a SHA-1 hash based on the
   shared secret KH between Bob's OP and the introduction point, as
   follows:
       HS = H(KH | "INTRODUCE")
   That is:
       HS = H(KH | [49 4E 54 52 4F 44 55 43 45])
   (KH, as specified in tor-spec.txt, is H(g^xy | [00]) .)

   Upon receiving such a cell, the OR first checks that the signature is
   correct with the included public key.  If so, it checks whether HS is
   correct given the shared state between Bob's OP and the OR.  If either
   check fails, the OP discards the cell; otherwise, it associates the
   circuit with Bob's public key, and dissociates any other circuits
   currently associated with PK.  On success, the OR sends Bob a
   RELAY_COMMAND_INTRO_ESTABLISHED cell with an empty payload.

   Bob's OP uses either Bob's public key or a freshly generated, single-use
   service key in the RELAY_COMMAND_ESTABLISH_INTRO cell, depending on the
   configured hidden service descriptor version.  The public key is used for
   v0 descriptors, the service key for v2 descriptors.  In the latter case, the
   service keys of all introduction points are included in the v2 hidden
   service descriptor together with the other introduction point information.
   The reason is that the introduction point does not need to and therefore
   should not know for which hidden service it works, so as to prevent it from
   tracking the hidden service's activity.  If the hidden service is configured
   to publish both v0 and v2 descriptors, two separate sets of introduction
   points are established.

1.3. Bob's OP generates service descriptors.

   For versions before 0.2.2.1-alpha, Bob's OP periodically generates and
   publishes a descriptor of type "V0".

   The "V0" descriptor contains:

         KL    Key length                            [2 octets]
         PK    Bob's public key                      [KL octets]
         TS    A timestamp                           [4 octets]
         NI    Number of introduction points         [2 octets]
         Ipt   A list of NUL-terminated ORs          [variable]
         SIG   Signature of above fields             [variable]

   TS is the number of seconds elapsed since Jan 1, 1970.

   The members of Ipt may be either (a) nicknames, or (b) identity key
   digests, encoded in hex, and prefixed with a '$'.  Clients must
   accept both forms. Services must only generate the second form.
   Once 0.0.9.x is obsoleted, we can drop the first form.

   [It's ok for Bob to advertise 0 introduction points. He might want
    to do that if he previously advertised some introduction points,
    and now he doesn't have any. -RD]

   Beginning with 0.2.0.10-alpha, Bob's OP encodes "V2" descriptors in
   addition to (or instead of) "V0" descriptors. The format of a "V2"
   descriptor is as follows:

     "rendezvous-service-descriptor" SP descriptor-id NL

       [At start, exactly once]
       [No extra arguments]

       Indicates the beginning of the descriptor. "descriptor-id" is a
       periodically changing identifier of 160 bits formatted as 32 base32
       chars that is calculated by the hidden service and its clients. The
       "descriptor-id" is calculated by performing the following operation:

         descriptor-id =
             H(permanent-id | H(time-period | descriptor-cookie | replica))

       "permanent-id" is the permanent identifier of the hidden service,
       consisting of 80 bits. It can be calculated by computing the hash value
       of the public hidden service key and truncating after the first 80 bits:

         permanent-id = H(public-key)[:10]

       Note: If Bob's OP has "stealth" authorization enabled (see Section 2.2),
       it uses the client key in place of the public hidden service key.

       "H(time-period | descriptor-cookie | replica)" is the (possibly
       secret) id part that is necessary to verify that the hidden service is
       the true originator of this descriptor and that is therefore contained
       in the descriptor, too. The descriptor ID can only be created by the
       hidden service and its clients, but the "signature" below can only be
       created by the service.

       "time-period" changes periodically as a function of time and
       "permanent-id". The current value for "time-period" can be calculated
       using the following formula:

         time-period = (current-time + permanent-id-byte * 86400 / 256)
                         / 86400

       "current-time" contains the current system time in seconds since
       1970-01-01 00:00, e.g. 1188241957. "permanent-id-byte" is the first
       (unsigned) byte of the permanent identifier (which is in network
       order), e.g. 143. Adding the product of "permanent-id-byte" and
       86400 (seconds per day), divided by 256, prevents "time-period" from
       changing for all descriptors at the same time of the day. The result
       of the overall operation is a (network-ordered) 32-bit integer, e.g.
       13753 or 0x000035B9 with the example values given above.

       "descriptor-cookie" is an optional secret password of 128 bits that
       is shared between the hidden service provider and its clients. If the
       descriptor-cookie is left out, the input to the hash function is 128
       bits shorter.  [No extra arguments]

       "replica" denotes the number of the replica. A service publishes
       multiple descriptors with different descriptor IDs in order to
       distribute them to different places on the ring.

     "version" SP version-number NL

       [Exactly once]
       [No extra arguments]

       The version number of this descriptor's format. Version numbers are a
       positive integer.

     "permanent-key" NL a public key in PEM format

       [Exactly once]
       [No extra arguments]

       The public key of the hidden service which is required to verify the
       "descriptor-id" and the "signature".

     "secret-id-part" SP secret-id-part NL

       [Exactly once]
       [No extra arguments]

       The result of the following operation as explained above, formatted as
       32 base32 chars. Using this secret id part, everyone can verify that
       the signed descriptor belongs to "descriptor-id".

         secret-id-part = H(time-period | descriptor-cookie | replica)

     "publication-time" SP YYYY-MM-DD HH:MM:SS NL

       [Exactly once]

       A timestamp when this descriptor has been created.  It should be
       rounded down to the nearest hour.

     "protocol-versions" SP version-string NL

       [Exactly once]
       [No extra arguments]

       A comma-separated list of recognized and permitted version numbers
       for use in INTRODUCE cells; these versions are described in section
       1.8 below. Version numbers are positive integers.

     "introduction-points" NL encrypted-string

       [At most once]
       [No extra arguments]

       A list of introduction points. If the optional "descriptor-cookie" is
       used, this list is encrypted with AES in CTR mode with a random
       initialization vector of 128 bits that is written to
       the beginning of the encrypted string, and the "descriptor-cookie" as
       secret key of 128 bits length.

       The string containing the introduction point data (either encrypted
       or not) is encoded in base64, and surrounded with
       "-----BEGIN MESSAGE-----" and "-----END MESSAGE-----".

       A maximum of 10 introduction point entries may follow, each containing
       the following data:

         "introduction-point" SP identifier NL

           [At start, exactly once]
           [No extra arguments]

           The identifier of this introduction point: the base32 encoded
           hash of this introduction point's identity key.

         "ip-address" SP ip4 NL

           [Exactly once]
           [No extra arguments]

           The IP address of this introduction point.

         "onion-port" SP port NL

           [Exactly once]
           [No extra arguments]

           The TCP port on which the introduction point is listening for
           incoming onion requests.

         "onion-key" NL a public key in PEM format

           [Exactly once]
           [No extra arguments]

           The public key that can be used to encrypt messages to this
           introduction point.

         "service-key" NL a public key in PEM format

           [Exactly once]
           [No extra arguments]

           The public key that can be used to encrypt messages to the hidden
           service.

         "intro-authentication" auth-type auth-data NL

           [Any number]

           The introduction-point-specific authentication data can be used
           to perform client authentication. This data depends on the
           selected introduction point as opposed to "service-authentication"
           above. The format of auth-data (base64-encoded or PEM format)
           depends on auth-type. See section 2 of this document for details
           on auth mechanisms.

        (This ends the fields in the encrypted portion of the descriptor.)

       [It's ok for Bob to advertise 0 introduction points. He might want
        to do that if he previously advertised some introduction points,
        and now he doesn't have any. -RD]

     "signature" NL signature-string

       [At end, exactly once]
       [No extra arguments]

       A signature of all fields above including '"signature" NL' with
       the private key of the hidden service.

1.3.1. Other descriptor formats we don't use.

   Support for the V0 descriptor format was dropped in 0.2.2.0-alpha-dev:

         KL    Key length                            [2 octets]
         PK    Bob's public key                      [KL octets]
         TS    A timestamp                           [4 octets]
         NI    Number of introduction points         [2 octets]
         Ipt   A list of NUL-terminated ORs          [variable]
         SIG   Signature of above fields             [variable]

   KL is the length of PK, in octets.
   TS is the number of seconds elapsed since Jan 1, 1970.

   The members of Ipt may be either (a) nicknames, or (b) identity key
   digests, encoded in hex, and prefixed with a '$'.

   The V1 descriptor format was understood and accepted from
   0.1.1.5-alpha-cvs to 0.2.0.6-alpha-dev, but no Tors generated it and
   it was removed:

         V     Format byte: set to 255               [1 octet]
         V     Version byte: set to 1                [1 octet]
         KL    Key length                            [2 octets]
         PK    Bob's public key                      [KL octets]
         TS    A timestamp                           [4 octets]
         PROTO Protocol versions: bitmask            [2 octets]
         NI    Number of introduction points         [2 octets]
         For each introduction point: (as in INTRODUCE2 cells)
             IP     Introduction point's address     [4 octets]
             PORT   Introduction point's OR port     [2 octets]
             ID     Introduction point identity ID   [20 octets]
             KLEN   Length of onion key              [2 octets]
             KEY    Introduction point onion key     [KLEN octets]
         SIG   Signature of above fields             [variable]

   A hypothetical "V1" descriptor, that has never been used but might
   be useful for historical reasons, contains:

         V     Format byte: set to 255               [1 octet]
         V     Version byte: set to 1                [1 octet]
         KL    Key length                            [2 octets]
         PK    Bob's public key                      [KL octets]
         TS    A timestamp                           [4 octets]
         PROTO Rendezvous protocol versions: bitmask [2 octets]
         NA    Number of auth mechanisms accepted    [1 octet]
         For each auth mechanism:
             AUTHT  The auth type that is supported  [2 octets]
             AUTHL  Length of auth data              [1 octet]
             AUTHD  Auth data                        [variable]
         NI    Number of introduction points         [2 octets]
         For each introduction point: (as in INTRODUCE2 cells)
             ATYPE  An address type (typically 4)    [1 octet]
             ADDR   Introduction point's IP address  [4 or 16 octets]
             PORT   Introduction point's OR port     [2 octets]
             AUTHT  The auth type that is supported  [2 octets]
             AUTHL  Length of auth data              [1 octet]
             AUTHD  Auth data                        [variable]
             ID     Introduction point identity ID   [20 octets]
             KLEN   Length of onion key              [2 octets]
             KEY    Introduction point onion key     [KLEN octets]
         SIG   Signature of above fields             [variable]

   AUTHT specifies which authentication/authorization mechanism is
   required by the hidden service or the introduction point. AUTHD
   is arbitrary data that can be associated with an auth approach.
   Currently only AUTHT of [00 00] is supported, with an AUTHL of 0.
   See section 2 of this document for details on auth mechanisms.

1.4. Bob's OP advertises his service descriptor(s).

   Bob's OP advertises his service descriptor to a fixed set of v0 hidden
   service directory servers and/or a changing subset of all v2 hidden service
   directories.

   For versions before 0.2.2.1-alpha, Bob's OP opens a stream to each v0
   directory server's directory port via Tor.  (He may re-use old circuits for
   this.)  Over this stream, Bob's OP makes an HTTP 'POST' request, to a URL
   "/tor/rendezvous/publish" relative to the directory server's root,
   containing as its body Bob's service descriptor.

   Upon receiving a descriptor, the directory server checks the signature,
   and discards the descriptor if the signature does not match the enclosed
   public key.  Next, the directory server checks the timestamp.  If the
   timestamp is more than 24 hours in the past or more than 1 hour in the
   future, or the directory server already has a newer descriptor with the
   same public key, the server discards the descriptor.  Otherwise, the
   server discards any older descriptors with the same public key and
   version format, and associates the new descriptor with the public key.
   The directory server remembers this descriptor for at least 24 hours
   after its timestamp.  At least every 18 hours, Bob's OP uploads a
   fresh descriptor.

   If Bob's OP is configured to publish v2 descriptors, it does so to a
   changing subset of all v2 hidden service directories instead of the
   authoritative directory servers. Therefore, Bob's OP opens a stream via
   Tor to each responsible hidden service directory. (He may re-use old
   circuits for this.) Over this stream, Bob's OP makes an HTTP 'POST'
   request to a URL "/tor/rendezvous2/publish" relative to the hidden service
   directory's root, containing as its body Bob's service descriptor.

   [XXX022 Reusing old circuits for HS dir posts is very bad. Do we really
    do that? --RR]

   At any time, there are 6 hidden service directories responsible for
   keeping replicas of a descriptor; they consist of 2 sets of 3 hidden
   service directories with consecutive onion IDs. Bob's OP learns about
   the complete list of hidden service directories by filtering the
   consensus status document received from the directory authorities. A
   hidden service directory is deemed responsible for a descriptor ID if
   it has the HSDir flag and its identity digest is one of the first three
   identity digests of HSDir relays following the descriptor ID in a
   circular list. A hidden service directory will only accept a descriptor
   whose timestamp is no more than three days before or one day after the
   current time according to the directory's clock.

   Bob's OP publishes a new v2 descriptor once an hour or whenever its
   content changes. V2 descriptors can be found by clients within a given
   time period of 24 hours, after which they change their ID as described
   under 1.3. If a published descriptor would be valid for less than 60
   minutes (= 2 x 30 minutes to allow the server to be 30 minutes behind
   and the client 30 minutes ahead), Bob's OP publishes the descriptor
   under the ID of both, the current and the next publication period.

1.5. Alice receives a z.onion address.

   When Alice receives a pointer to a location-hidden service, it is as a
   hostname of the form "z.onion", where z is a base32 encoding of a
   10-octet hash of Bob's service's public key, computed as follows:

         1. Let H = H(PK).
         2. Let H' = the first 80 bits of H, considering each octet from
            most significant bit to least significant bit.
         3. Generate a 16-character encoding of H', using base32 as defined
            in RFC 4648.

   (We only use 80 bits instead of the 160 bits from SHA1 because we
   don't need to worry about arbitrary collisions, and because it will
   make handling the url's more convenient.)

   [Yes, numbers are allowed at the beginning.  See RFC 1123. -NM]

1.6. Alice's OP retrieves a service descriptor.

   Alice's OP fetches the service descriptor from the fixed set of v0 hidden
   service directory servers and/or a changing subset of all v2 hidden service
   directories.

   For versions before 0.2.2.1-alpha, Alice's OP opens a stream to a directory
   server via Tor, and makes an HTTP GET request for the document
   '/tor/rendezvous/<z>', where '<z>' is replaced with the encoding of Bob's
   public key as described above. (She may re-use old circuits for this.) The
   directory replies with a 404 HTTP response if it does not recognize <z>,
   and otherwise returns Bob's most recently uploaded service descriptor.

   If Alice's OP receives a 404 response, it tries the other directory
   servers, and only fails the lookup if none recognize the public key hash.

   Upon receiving a service descriptor, Alice verifies with the same process
   as the directory server uses, described above in section 1.4.

   The directory server gives a 400 response if it cannot understand Alice's
   request.

   Alice should cache the descriptor locally, but should not use
   descriptors that are more than 24 hours older than their timestamp.
   [Caching may make her partitionable, but she fetched it anonymously,
    and we can't very well *not* cache it. -RD]

   If Alice's OP is running 0.2.1.10-alpha or higher, it fetches v2 hidden
   service descriptors. Versions before 0.2.2.1-alpha are fetching both v0 and
   v2 descriptors in parallel. Similar to the description in section 1.4,
   Alice's OP fetches a v2 descriptor from a randomly chosen hidden service
   directory out of the changing subset of 6 nodes. If the request is
   unsuccessful, Alice retries the other remaining responsible hidden service
   directories in a random order. Alice relies on Bob to care about a potential
   clock skew between the two by possibly storing two sets of descriptors (see
   end of section 1.4).

   Alice's OP opens a stream via Tor to the chosen v2 hidden service
   directory. (She may re-use old circuits for this.) Over this stream,
   Alice's OP makes an HTTP 'GET' request for the document
   "/tor/rendezvous2/<z>", where z is replaced with the encoding of the
   descriptor ID. The directory replies with a 404 HTTP response if it does
   not recognize <z>, and otherwise returns Bob's most recently uploaded
   service descriptor.

1.7. Alice's OP establishes a rendezvous point.

   When Alice requests a connection to a given location-hidden service,
   and Alice's OP does not have an established circuit to that service,
   the OP builds a rendezvous circuit.  It does this by establishing
   a circuit to a randomly chosen OR, and sending a
   RELAY_COMMAND_ESTABLISH_RENDEZVOUS cell to that OR.  The body of that cell
   contains:

        RC   Rendezvous cookie    [20 octets]

   The rendezvous cookie is an arbitrary 20-byte value, chosen randomly by
   Alice's OP. Alice SHOULD choose a new rendezvous cookie for each new
   connection attempt.

   Upon receiving a RELAY_COMMAND_ESTABLISH_RENDEZVOUS cell, the OR associates
   the RC with the circuit that sent it.  It replies to Alice with an empty
   RELAY_COMMAND_RENDEZVOUS_ESTABLISHED cell to indicate success.

   Alice's OP MUST NOT use the circuit which sent the cell for any purpose
   other than rendezvous with the given location-hidden service.

1.8. Introduction: from Alice's OP to Introduction Point

   Alice builds a separate circuit to one of Bob's chosen introduction
   points, and sends it a RELAY_COMMAND_INTRODUCE1 cell containing:

       Cleartext
          PK_ID  Identifier for Bob's PK      [20 octets]
       Encrypted to Bob's PK: (in the v0 intro protocol)
          RP     Rendezvous point's nickname  [20 octets]
          RC     Rendezvous cookie            [20 octets]
          g^x    Diffie-Hellman data, part 1 [128 octets]
        OR (in the v1 intro protocol)
          VER    Version byte: set to 1.        [1 octet]
          RP     Rendezvous point nick or ID  [42 octets]
          RC     Rendezvous cookie            [20 octets]
          g^x    Diffie-Hellman data, part 1 [128 octets]
        OR (in the v2 intro protocol)
          VER    Version byte: set to 2.        [1 octet]
          IP     Rendezvous point's address    [4 octets]
          PORT   Rendezvous point's OR port    [2 octets]
          ID     Rendezvous point identity ID [20 octets]
          KLEN   Length of onion key           [2 octets]
          KEY    Rendezvous point onion key [KLEN octets]
          RC     Rendezvous cookie            [20 octets]
          g^x    Diffie-Hellman data, part 1 [128 octets]
        OR (in the v3 intro protocol)
          VER    Version byte: set to 3.        [1 octet]
          AUTHT  The auth type that is used     [1 octet]
          If AUTHT != [00]:
              AUTHL  Length of auth data           [2 octets]
              AUTHD  Auth data                     [variable]
          TS     A timestamp                   [4 octets]
          IP     Rendezvous point's address    [4 octets]
          PORT   Rendezvous point's OR port    [2 octets]
          ID     Rendezvous point identity ID [20 octets]
          KLEN   Length of onion key           [2 octets]
          KEY    Rendezvous point onion key [KLEN octets]
          RC     Rendezvous cookie            [20 octets]
          g^x    Diffie-Hellman data, part 1 [128 octets]

   PK_ID is the hash of Bob's public key or the service key, depending on the
   hidden service descriptor version. In case of a v0 descriptor, Alice's OP
   uses Bob's public key. If Alice has downloaded a v2 descriptor, she uses
   the contained public key ("service-key").

   RP is NUL-padded and terminated. In version 0 of the intro protocol, RP
   must contain a nickname. In version 1, it must contain EITHER a nickname or
   an identity key digest that is encoded in hex and prefixed with a '$'.

   The hybrid encryption to Bob's PK works just like the hybrid
   encryption in CREATE cells (see tor-spec, section 0.4). Thus the
   payload of the version 0 RELAY_COMMAND_INTRODUCE1 cell on the
   wire will contain 20+42+16+20+20+128=246 bytes, and the version 1
   and version 2 introduction formats have other sizes.

   Through Tor 0.2.0.6-alpha, clients only generated the v0 introduction
   format, whereas hidden services have understood and accepted v0,
   v1, and v2 since 0.1.1.x. As of Tor 0.2.0.7-alpha and 0.1.2.18,
   clients switched to using the v2 intro format.

   The Timestampe (TS) field is no longer used in Tor 0.2.3.9-alpha and
   later.  Clients MAY refrain from sending it; see the
   "Support022HiddenServices" parameter in dir-spec.txt. Clients SHOULD
   NOT send a precise timestamp, and should instead round to the nearest
   10 minutes.

1.9. Introduction: From the Introduction Point to Bob's OP

   If the Introduction Point recognizes PK_ID as a public key which has
   established a circuit for introductions as in 1.2 above, it sends the body
   of the cell in a new RELAY_COMMAND_INTRODUCE2 cell down the corresponding
   circuit. (If the PK_ID is unrecognized, the RELAY_COMMAND_INTRODUCE1 cell is
   discarded.)

   After sending the RELAY_COMMAND_INTRODUCE2 cell to Bob, the OR replies to
   Alice with an empty RELAY_COMMAND_INTRODUCE_ACK cell.  If no
   RELAY_COMMAND_INTRODUCE2 cell can be sent, the OR replies to Alice with a
   non-empty cell to indicate an error.  (The semantics of the cell body may be
   determined later; the current implementation sends a single '1' byte on
   failure.)

   When Bob's OP receives the RELAY_COMMAND_INTRODUCE2 cell, it first
   checks for a replay.  Because of the (undesirable!) malleability of
   the legacy hybrid encryption algorithm, Bob's OP should only check
   whether the RSA-encrypted part is replayed.  It does this by keeping,
   for each introduction key, a list of cryptographic digests of all the
   RSA-encrypted parts of the INTRODUCE2 cells that it's seen, and
   dropping any INTRODUCE2 cell whose RSA-encrypted part it has seen
   before.  When Bob's OP stops using a given introduction key, it drops
   the replay cache corresponding to that key.

   (Versions of Tor before 0.2.3.9-alpha used the timestamp in the INTRODUCE2
   cell to limit the lifetime of entries in the replay cache. This proved to
   be fragile, due to insufficiently synchronized clients.)

   Assuming that the cell has not been replayed, Bob's server decrypts it
   with the private key for the corresponding hidden service, and extracts the
   rendezvous point's nickname, the rendezvous cookie, and the value of g^x
   chosen by Alice.

1.10. Rendezvous

   Bob's OP builds a new Tor circuit ending at Alice's chosen rendezvous
   point, and sends a RELAY_COMMAND_RENDEZVOUS1 cell along this circuit,
   containing:
       RC       Rendezvous cookie  [20 octets]
       g^y      Diffie-Hellman     [128 octets]
       KH       Handshake digest   [20 octets]

   (Bob's OP MUST NOT use this circuit for any other purpose.)

   (By default, Bob builds a circuit of at least three hops, *not including*
   Alice's chosen rendezvous point.)

   If the RP recognizes RC, it relays the rest of the cell down the
   corresponding circuit in a RELAY_COMMAND_RENDEZVOUS2 cell, containing:

       g^y      Diffie-Hellman     [128 octets]
       KH       Handshake digest   [20 octets]

   (If the RP does not recognize the RC, it discards the cell and
   tears down the circuit.)

   Rendezvous points running Tor version 0.2.9.1-alpha and later are
   willing to pass on RENDEZVOUS2 cells so long as they contain at least
   the 20 bytes of cookie. Prior to 0.2.9.1-alpha, the RP refused the
   cell if it had a payload length different from 20+128+20.

   When Alice's OP receives a RELAY_COMMAND_RENDEZVOUS2 cell on a circuit which
   has sent a RELAY_COMMAND_ESTABLISH_RENDEZVOUS cell but which has not yet
   received a reply, it uses g^y and H(g^xy) to complete the handshake as in
   the Tor circuit extend process: they establish a 60-octet string as
       K = SHA1(g^xy | [00]) | SHA1(g^xy | [01]) | SHA1(g^xy | [02])
   and generate KH, Df, Db, Kf, and Kb as in the KDF-TOR key derivation
   approach documented in tor-spec.txt.

   As in the TAP handshake, if the KH value derived from KDF-Tor does not
   match the value in the RENDEZVOUS2 cell, the client must close the
   circuit.

   Subsequently, the rendezvous point passes RELAY cells, unchanged, from
   each of the two circuits to the other.  When Alice's OP sends RELAY cells
   along the circuit, it authenticates with Df, and encrypts them with the
   Kf, then with all of the keys for the ORs in Alice's side of the circuit;
   and when Alice's OP receives RELAY cells from the circuit, it decrypts
   them with the keys for the ORs in Alice's side of the circuit, then
   decrypts them with Kb, and checks integrity with Db.  Bob's OP does the
   same, with Kf and Kb interchanged.

1.11. Creating streams

   To open TCP connections to Bob's location-hidden service, Alice's OP sends
   a RELAY_COMMAND_BEGIN cell along the established circuit, using the special
   address "", and a chosen port.  Bob's OP chooses a destination IP and
   port, based on the configuration of the service connected to the circuit,
   and opens a TCP stream.  From then on, Bob's OP treats the stream as an
   ordinary exit connection.
   [ Except he doesn't include addr in the connected cell or the end
     cell. -RD]

   Alice MAY send multiple RELAY_COMMAND_BEGIN cells along the circuit, to open
   multiple streams to Bob.  Alice SHOULD NOT send RELAY_COMMAND_BEGIN cells
   for any other address along her circuit to Bob; if she does, Bob MUST reject
   them.

1.12. Closing streams

   The payload of a RELAY_END cell begins with a single 'reason' byte to
   describe why the stream is closing, plus optional data (depending on the
   reason.) These can be found in section 6.3 of tor-spec. The following
   describes some of the hidden service related reasons.

       1 -- REASON_MISC

       Catch-all for unlisted reasons. Shouldn't happen much in practice.

       2 -- REASON_RESOLVEFAILED

       Tor tried to fetch the hidden service descriptor from the hsdirs but
       none of them had it. This implies that the hidden service has not
       been running in the past 24 hours.

       3 -- REASON_CONNECTREFUSED

       Every step of the rendezvous worked great, and that the hidden
       service is indeed up and running and configured to use the virtual
       port you asked for, but there was nothing listening on the other end
       of that virtual port. For example, the HS's Tor client is running
       fine but its apache service is down.

       4 -- REASON_EXITPOLICY

       The destination port that you tried is not configured on the hidden
       service side. That is, the hidden service was up and reachable, but
       it isn't listening on this port. Since Tor 0.2.6.2-alpha and later
       hidden service don't send this error code; instead they send back an
       END cell with reason DONE reason then close the circuit on you. This
       behavior can be controlled by a config option.

       5 -- REASON_DESTROY

       The circuit closed before you could get a response back -- transient
       failure, e.g. a relay went down unexpectedly. Trying again might
       work.

       6 -- REASON_DONE

       Anonymized TCP connection was closed. If you get an END cell with
       reason DONE, *before* you've gotten your CONNECTED cell, that
       indicates a similar situation to EXITPOLICY, but the hidden service
       is running 0.2.6.2-alpha or later, and it has now closed the circuit
       on you.

       7 -- REASON_TIMEOUT

       Either like CONNECTREFUSED above but connect() got the ETIMEDOUT
       errno, or the client-side timeout of 120 seconds kicked in and we
       gave up.

       8 -- REASON_NOROUTE

       Like CONNECTREFUSED except the errno at the hidden service when
       trying to connect() to the service was ENETUNREACH, EHOSTUNREACH,
       EACCES, or EPERM.

       10 -- REASON_INTERNAL

       Internal error inside the Tor client -- hopefully you will not see
       this much. Please report if you do!

       12 -- REASON_CONNRESET

       Like CONNECTREFUSED except the errno at the hidden service when
       trying to connect() to the service was ECONNRESET.

2. Authentication and authorization.

   The rendezvous protocol as described in Section 1 provides a few options
   for implementing client-side authorization. There are two steps in the
   rendezvous protocol that can be used for performing client authorization:
   when downloading and decrypting parts of the hidden service descriptor and
   at Bob's Tor client before contacting the rendezvous point. A service
   provider can restrict access to his service at these two points to
   authorized clients only.

   There are currently two authorization protocols specified that are
   described in more detail below:

    1. The first protocol allows a service provider to restrict access
       to clients with a previously received secret key only, but does not
       attempt to hide service activity from others.

    2. The second protocol, albeit being feasible for a limited set of about
       16 clients, performs client authorization and hides service activity
       from everyone but the authorized clients.

2.1. Service with large-scale client authorization

   The first client authorization protocol aims at performing access control
   while consuming as few additional resources as possible. This is the "basic"
   authorization protocol. A service provider should be able to permit access
   to a large number of clients while denying access for everyone else.
   However, the price for scalability is that the service won't be able to hide
   its activity from unauthorized or formerly authorized clients.

   The main idea of this protocol is to encrypt the introduction-point part
   in hidden service descriptors to authorized clients using symmetric keys.
   This ensures that nobody else but authorized clients can learn which
   introduction points a service currently uses, nor can someone send a
   valid INTRODUCE1 message without knowing the introduction key. Therefore,
   a subsequent authorization at the introduction point is not required.

   A service provider generates symmetric "descriptor cookies" for his
   clients and distributes them outside of Tor. The suggested key size is
   128 bits, so that descriptor cookies can be encoded in 22 base64 chars
   (which can hold up to 22 * 6 = 132 bits, leaving 4 bits to encode the
   authorization type (here: "0") and allow a client to distinguish this
   authorization protocol from others like the one proposed below).
   Typically, the contact information for a hidden service using this
   authorization protocol looks like this:

     v2cbb2l4lsnpio4q.onion Ll3X7Xgz9eHGKCCnlFH0uz

   When generating a hidden service descriptor, the service encrypts the
   introduction-point part with a single randomly generated symmetric
   128-bit session key using AES-CTR as described for v2 hidden service
   descriptors in rend-spec. Afterwards, the service encrypts the session
   key to all descriptor cookies using AES. Authorized client should be able
   to efficiently find the session key that is encrypted for him/her, so
   that 4 octet long client ID are generated consisting of descriptor cookie
   and initialization vector. Descriptors always contain a number of
   encrypted session keys that is a multiple of 16 by adding fake entries.
   Encrypted session keys are ordered by client IDs in order to conceal
   addition or removal of authorized clients by the service provider.

     ATYPE  Authorization type: set to 1.                      [1 octet]
     ALEN   Number of clients := 1 + ((clients - 1) div 16)    [1 octet]
   for each symmetric descriptor cookie:
     ID     Client ID: H(descriptor cookie | IV)[:4]          [4 octets]
     SKEY   Session key encrypted with descriptor cookie     [16 octets]
   (end of client-specific part)
     RND    Random data      [(15 - ((clients - 1) mod 16)) * 20 octets]
     IV     AES initialization vector                        [16 octets]
     IPOS   Intro points, encrypted with session key  [remaining octets]

   An authorized client needs to configure Tor to use the descriptor cookie
   when accessing the hidden service. Therefore, a user adds the contact
   information that she received from the service provider to her torrc
   file. Upon downloading a hidden service descriptor, Tor finds the
   encrypted introduction-point part and attempts to decrypt it using the
   configured descriptor cookie. (In the rare event of two or more client
   IDs being equal a client tries to decrypt all of them.)

   Upon sending the introduction, the client includes her descriptor cookie
   as auth type "1" in the INTRODUCE2 cell that she sends to the service.
   The hidden service checks whether the included descriptor cookie is
   authorized to access the service and either responds to the introduction
   request, or not.

2.2. Authorization for limited number of clients

   A second, more sophisticated client authorization protocol goes the extra
   mile of hiding service activity from unauthorized clients. This is the
   "stealth" authorization protocol. With all else being equal to the preceding
   authorization protocol, the second protocol publishes hidden service
   descriptors for each user separately and gets along with encrypting the
   introduction-point part of descriptors to a single client. This allows the
   service to stop publishing descriptors for removed clients. As long as a
   removed client cannot link descriptors issued for other clients to the
   service, it cannot derive service activity any more. The downside of this
   approach is limited scalability. Even though the distributed storage of
   descriptors (cf. proposal 114) tackles the problem of limited scalability to
   a certain extent, this protocol should not be used for services with more
   than 16 clients. (In fact, Tor should refuse to advertise services for more
   than this number of clients.)

   A hidden service generates an asymmetric "client key" and a symmetric
   "descriptor cookie" for each client. The client key is used as
   replacement for the service's permanent key, so that the service uses a
   different identity for each of his clients. The descriptor cookie is used
   to store descriptors at changing directory nodes that are unpredictable
   for anyone but service and client, to encrypt the introduction-point
   part, and to be included in INTRODUCE2 cells. Once the service has
   created client key and descriptor cookie, he tells them to the client
   outside of Tor. The contact information string looks similar to the one
   used by the preceding authorization protocol (with the only difference
   that it has "1" encoded as auth-type in the remaining 4 of 132 bits
   instead of "0" as before).

   When creating a hidden service descriptor for an authorized client, the
   hidden service uses the client key and descriptor cookie to compute
   secret ID part and descriptor ID:

     secret-id-part = H(time-period | descriptor-cookie | replica)

     descriptor-id = H(client-key[:10] | secret-id-part)

   The hidden service also replaces permanent-key in the descriptor with
   client-key and encrypts introduction-points with the descriptor cookie.

     ATYPE  Authorization type: set to 2.                         [1 octet]
     IV     AES initialization vector                           [16 octets]
     IPOS   Intro points, encr. with descriptor cookie   [remaining octets]

   When uploading descriptors, the hidden service needs to make sure that
   descriptors for different clients are not uploaded at the same time (cf.
   Section 1.1) which is also a limiting factor for the number of clients.

   When a client is requested to establish a connection to a hidden service
   it looks up whether it has any authorization data configured for that
   service. If the user has configured authorization data for authorization
   protocol "2", the descriptor ID is determined as described in the last
   paragraph. Upon receiving a descriptor, the client decrypts the
   introduction-point part using its descriptor cookie. Further, the client
   includes its descriptor cookie as auth-type "2" in INTRODUCE2 cells that
   it sends to the service.

2.3. Hidden service configuration

   A hidden service that is meant to perform client authorization adds a
   new option HiddenServiceAuthorizeClient to its hidden service
   configuration. This option contains the authorization type which is
   either "basic" for the protocol described in 2.1 or "stealth" for the
   protocol in 2.2 and a comma-separated list of human-readable client
   names, so that Tor can create authorization data for these clients:

     HiddenServiceAuthorizeClient auth-type client-name,client-name,...

   If this option is configured, HiddenServiceVersion is automatically
   reconfigured to contain only version numbers of 2 or higher. There is
   a maximum of 512 client names for basic auth and a maximum of 16 for
   stealth auth.

   Tor stores all generated authorization data for the authorization
   protocols described in Sections 2.1 and 2.2 in a new file using the
   following file format:

     "client-name" human-readable client identifier NL
     "descriptor-cookie" 128-bit key ^= 22 base64 chars NL

   If the authorization protocol of Section 2.2 is used, Tor also generates
   and stores the following data:

     "client-key" NL a private key in PEM format
     [No extra arguments]

2.4. Client configuration

   To specify the cookie to use to access a given hidden service,
   clients use the following syntax:

    HidServAuth onion-address auth-cookie [service-name]:

     Valid onion addresses contain 16 characters in a-z2-7 plus
     ".onion", and valid auth cookies contain 22 characters in
     A-Za-z0-9+/. The service name is only used for internal purposes,
     e.g., for Tor controllers; nothing in Tor itself requires or uses
     it.


3. Hidden service directory operation

   This section has been introduced with the v2 hidden service descriptor
   format. It describes all operations of the v2 hidden service descriptor
   fetching and propagation mechanism that are required for the protocol
   described in section 1 to succeed with v2 hidden service descriptors.

3.1. Configuring as hidden service directory

   Every onion router that has its directory port open can decide whether it
   wants to store and serve hidden service descriptors. An onion router which
   is configured as such includes the "hidden-service-dir" flag in its router
   descriptors that it sends to directory authorities.

   The directory authorities include a new flag "HSDir" for routers that
   decided to provide storage for hidden service descriptors and that
   have been running for at least 96 hours.

3.2. Accepting publish requests

   Hidden service directory nodes accept publish requests for v2 hidden service
   descriptors and store them to their local memory. (It is not necessary to
   make descriptors persistent, because after restarting, the onion router
   would not be accepted as a storing node anyway, because it has not been
   running for at least 24 hours.) All requests and replies are formatted as
   HTTP messages. Requests are initiated via BEGIN_DIR cells directed to
   the router's directory port, and formatted as HTTP POST requests to the URL
   "/tor/rendezvous2/publish" relative to the hidden service directory's root,
   containing as its body a v2 service descriptor.

   A hidden service directory node parses every received descriptor and only
   stores it when it thinks that it is responsible for storing that descriptor
   based on its own routing table. See section 1.4 for more information on how
   to determine responsibility for a certain descriptor ID.

3.3. Processing fetch requests

   Hidden service directory nodes process fetch requests for hidden service
   descriptors by looking them up in their local memory. (They do not need to
   determine if they are responsible for the passed ID, because it does no harm
   if they deliver a descriptor for which they are not (any more) responsible.)
   All requests and replies are formatted as HTTP messages. Requests are
   initiated via BEGIN_DIR cells directed to the router's directory port,
   and formatted as HTTP GET requests for the document "/tor/rendezvous2/<z>",
   where z is replaced with the encoding of the descriptor ID.
  
