**Important:** Read CONTRIBUTING.md before submitting feedback or contributing
```




DNSOP Working Group                                         J. Woodworth
Internet-Draft                                                 D. Ballew
Obsoletes: 222 (if approved)                           CenturyLink, Inc.
Updates: 2308, 4033, 4034, 4035 (if           S. Bindinganaveli Raghavan
         approved)                                Hughes Network Systems
Intended status: Standards Track                             D. Lawrence
Expires: January 4, 2018                             Akamai Technologies
                                                            July 3, 2017


                       BULK DNS Resource Records
                       draft-woodworth-bulk-rr-06

Abstract

   The BULK DNS resource record type defines a method of pattern-based
   creation of DNS resource records based on numeric substrings of query
   names.  The intent of BULK is to simplify generic assignments in a
   memory-efficient way that can be easily shared between the primary
   and secondary nameservers for a zone.

Ed note

   Text inside square brackets ([]) is additional background
   information, answers to frequently asked questions, general musings,
   etc.  They will be removed before publication.  This document is
   being collaborated on in GitHub at <https://github.com/vttale/bulk-
   rr>.  The most recent version of the document, open issues, etc
   should all be available here.  The authors gratefully accept pull
   requests.

Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

   Internet-Drafts are working documents of the Internet Engineering
   Task Force (IETF).  Note that other groups may also distribute
   working documents as Internet-Drafts.  The list of current Internet-
   Drafts is at http://datatracker.ietf.org/drafts/current/.

   Internet-Drafts are draft documents valid for a maximum of six months
   and may be updated, replaced, or obsoleted by other documents at any
   time.  It is inappropriate to use Internet-Drafts as reference
   material or to cite them other than as "work in progress."

   This Internet-Draft will expire on January 4, 2018.

Copyright Notice

   Copyright (c) 2017 IETF Trust and the persons identified as the
   document authors.  All rights reserved.

   This document is subject to BCP 78 and the IETF Trust's Legal
   Provisions Relating to IETF Documents
   (http://trustee.ietf.org/license-info) in effect on the date of
   publication of this document.  Please review these documents
   carefully, as they describe your rights and restrictions with respect
   to this document.  Code Components extracted from this document must
   include Simplified BSD License text as described in Section 4.e of
   the Trust Legal Provisions and are provided without warranty as
   described in the Simplified BSD License.

Table of Contents

   1.  Introduction
     1.1.  Background and Terminology
   2.  The BULK Resource Record
     2.1.  BULK RDATA Wire Format
     2.2.  The BULK RR Presentation Format
   3.  BULK Replacement
     3.1.  Matching the Domain Name Pattern
     3.2.  Record Generation using Replacement Pattern
       3.2.1.  Delimiters
       3.2.2.  Delimiter intervals
       3.2.3.  Padding length
       3.2.4.  Final processing
   4.  The NPN Resource Record
     4.1.  NPN RDATA Wire Format
     4.2.  The NPN RR Presentation Format
     4.3.  Use and Normalization Processing of NPN RRs
       4.3.1.  Pseudocode for NPN Normalization Processing
     4.4.  Pattern Based DNSSEC support
   5.  Known Limitations
     5.1.  Unsupported Nameservers
   6.  Security Considerations
     6.1.  DNSSEC Signature Strategies
       6.1.1.  On-the-fly Signatures
       6.1.2.  Normalized (NPN-Based) Signatures
       6.1.3.  Non-DNSSEC Zone Support Only
     6.2.  DNSSEC Validator Details
     6.3.  DDOS Attack Vectors and Mitigation
     6.4.  Implications of Large-Scale DNS Records
   7.  Privacy Considerations
   8.  IANA Considerations
   9.  Acknowledgments
   10. References
     10.1.  Normative References
     10.2.  Informative References
   Appendix A.  BULK Examples
     A.1.  Example 1
     A.2.  Example 2
     A.3.  Example 3
   Appendix B.  NPN Examples
     B.1.  EXAMPLE 1
     B.2.  EXAMPLE 2
     B.3.  EXAMPLE 3
     B.4.  EXAMPLE 4
   Authors' Addresses

1.  Introduction

   The BULK DNS resource record defines a pattern-based method for on-
   the-fly resource record generation.  It is essentially an enhanced
   wildcard mechanism, constraining generated resource record owner
   names to those that match a pattern of variable numeric substrings.
   It is also akin to the $GENERATE master file directive [bind-arm]
   without being limited to numeric values and without creating all
   possible records in the zone data.

   For example, consider the following record:

   example.com. 86400 IN BULK A (
                         pool-A-[0-255]-[0-255].example.com.
                         10.55.${1}.${2}
                      )

   It will answer requests for pool-A-0-0.example.com through pool-
   A-255-255.example.com with the IPv4 addresses 10.55.0.0 through
   10.55.255.255.

   Much larger record sets can be defined while minimizing the
   associated requirements for server memory and zone transfer network
   bandwidth.

   DNSSEC support is also described.  The Numeric Pattern Normalization
   (NPN) resource record provides a way of generating pattern-based
   DNSSEC signatures, and securely performing DNSSEC validation on such
   signatures.

1.1.  Background and Terminology

   ppThe reader is assumed to be familiar with the basic DNS and DNSSEC
   concepts described in [RFC1034], [RFC1035], [RFC4033], [RFC4034], and
   [RFC4035]; subsequent RFCs that update them in [RFC2181] and
   [RFC2308]; and DNS terms in [RFC7719].

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
   document are to be interpreted as described in [RFC2119].

2.  The BULK Resource Record

   The BULK resource record enables an authoritative nameserver to
   generate RRs for other types based upon the query received.

   The Type value for the BULK RR type is TBD.

   The BULK RR is class-independent.

2.1.  BULK RDATA Wire Format

   The RDATA for a BULK RR is as follows:

                        1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Match Type          |                               /
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+       Domain Name Pattern     /
   /                                                               /
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   /                                                               /
   /                      Replacement Pattern                      /
   /                                                               /
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Match Type identifies the type of the RRset to be generated by this
   BULK record.  It is two octets corresponding to an RR TYPE code as
   specified in [RFC1035], Section 3.2.1.

   Domain Name Pattern consists of a pattern encoded as a wire-format
   fully qualified domain name.  The full name is used so that numeric
   substrings above the zone cut can be captured in addition to those in
   the zone.  It needs no length indicator for the entire field because
   the root label marks its end.

   Special characters are interpreted as per the following Augmented
   Backus-Naur Form (ABNF) notation from [RFC5234].

   match         =  1*(range / string)

   range         =  "[" decnum "-" decnum "]" /
                     "<" hexnum "-" hexnum ">"
                         ; create references for substitution

   string        =  1*(ctext / quoted-char)

   decnum        =  1*decdigit
                         ; constrained to 65535 maximum.

   hexnum        =  1*hexdigit
                         ; constrained to ffff maximum.

   octet         =  %x00-FF

   decdigit      =  %x30-39
                         ; 0-9
   hexdigit      =  DIGIT / 0x41-0x46 / 0x61-66
                         ; 0-9, A-F, a-f

   ctext         =  <any octet excepting "\">

   quoted-char   = "\" octet
                          ; to allow special characters as literals

   [ Should [] and <> be allowed as short for [0-255] and <00-ff>? ]

   Interpretation of the Domain Name Pattern is described in detail in
   the "BULK Replacement" section.

   Replacement Pattern describes how the answer RRset MUST be generated
   for the matching query.  It needs no length indicator because its end
   can be derived from the RDATA length minus Match Type and Domain Name
   Pattern lengths.  It uses the following additional ABNF elements:

   replace       =   1*(reference / string)

   reference     =   "$" "{" (positions / "*") [options] "}"

   positions     =   (position / posrange) 0*("," (position / posrange))

   posrange      =   position "-" position

   position      =   1*decnum

   options       =   delimiter [interval [padding]]

   delimiter     =   "|" *1(ctext | quoted-char)
                           ; "\|" to use "|" as delimiter
                           ; "\\" to use "\" as delimiter

   interval      =   "|" *2DIGIT

   padding       =   "|" *2DIGIT


   [ Is this complexity beyond simple ${1}, ${2}, etc, really worth it?
   I definitely see how it could make for shorter replacement patterns,
   but does it enhance their clarity and usability? ]

   The Replacement Pattern MUST end in a period if it is intended to
   represent a fully qualified domain name.

   [ Should there be a defined limit on the number of references?  Even
   every hex character of an IPv6 address is only 32 references, and
   knowing a reasonable maximum can possibly simplify implementations. ]

2.2.  The BULK RR Presentation Format

   Match Type is represented as an RR type mnemonic or with [RFC3597]'s
   generic TYPE mechanism.

   Domain Name Pattern is represented as a fully qualified domain name
   as per [RFC1035] Section 5.1 rules for encoding whitespace and other
   special characters.

   Replacement Pattern is represented by the standard <character-string>
   text rules for master files as per [RFC1035] section 5.1.

   It is suggested that lines longer than 80 characters be wrapped with
   parenthetical line continuation, per [RFC1035] Section 5.1, starting
   after Match Type and ending after Replacement Pattern.

3.  BULK Replacement

   When an authoritative nameserver receives a query for which it does
   not have a matching name or a covering wildcard, it MUST then look
   for BULK RRs at the zone apex, selecting all BULK RRs with a Match
   Type that matches the query type and a Domain Name Pattern that
   matches the query name.  Note that query type ANY will select all
   Match Types, and all query types match a CNAME Match Type [ and
   DNAME? ].  One or more answer RRs will be generated per the
   replacement rules below.  Examples are provided in an appendix.

   By only triggering the BULK algorithm when the query name does not
   exist, administrators are given the flexibility to explicitly
   override the behaviour of specific names that would otherwise match
   the BULK record's Domain Name Pattern.  This is unlike BIND's
   $GENERATE directive, which adds the generated RRs to any existing
   names.

3.1.  Matching the Domain Name Pattern

   A query name matches the Domain Name Pattern if the characters that
   appear outside the numeric ranges match exactly and those within
   numeric ranges have values that fall within the range.  Numeric
   matches MUST be of the appropriate decimal or hexadecimal type as
   specified by the delimiters in the pattern.  For example, if a range
   is given as [0-255], then FF does not match even though its value as
   a hexadecimal number is within the range.

   When a query name matches a Domain Name Pattern, the value in each
   numeric range is stored for use by the Replacement Pattern, with
   reference numbers starting at 1 and counting from the left.  For
   example, matching the query name host-24-156 against
   host-[0-255]-[0-255] assigns 24 to ${1} and 156 to ${2}.

3.2.  Record Generation using Replacement Pattern

   The Replacement Pattern generates the record data by replacing the
   ${...} references with data captured from the query name, and copying
   all other characters literally.

   The simplest form of reference uses only the reference number between
   the braces, "{" and "}".  The value of the reference is simply copied
   directly from the matching position of the query name.

   The next form of reference notation uses the asterisk, "_".  With
   ${_}, all captured values in order of ascending position, delimited
   by its default delimiter (described below), are placed in the answer.

   Numeric range references, such as ${1-4}, replaces all values
   captured by those references, in order, delimited by the default
   delimiter described below.  To reverse the order in which they are
   copied, reverse the upper and lower values, such as ${4-1}.  This is
   useful for generating PTR records from query names in which the
   address is encoded in network order.

   Similar to range references, separating positions by commas creates
   sets for replacement.  For example, ${1,4} would be replaced by the
   first and fourth captured values, delimited its default delimiter.
   This notation may be combined with the numeric range form, such as
   ${3,2,1,8-4}.

3.2.1.  Delimiters

   A reference can specify a delimiter to use between copied position
   values by following a vertical bar, "|", with either zero or one
   characters.  [ Why not any length? ]?  Zero characters, such as in
   ${1-3|}, means no delimiter is used.  The default delimiter is the
   hyphen, "-". [ Earlier drafts attempted to make the default delimiter
   context-dependent, such as by using a period for A requests and a
   colon for AAAA requests.  This increases implementation complexity as
   an attempt to make things more intuitive for zone administrators.
   However it isn't clear that this is a net gain for ease of use. ]

3.2.2.  Delimiter intervals

   A second vertical bar in the reference options introduces a delimiter
   interval.  The default behavior of a multi-position reference is to
   combine each captured value specified with a delimiter between each.
   With a delimiter interval the delimiters are only added between every
   Nth value.  For example, ${*|-|4} adds a hyphen between every group
   of four captured positions.  This can be a handy feature in the IPv6
   reverse namespace where every nibble is captured as a separate value
   and generated hostnames include sets of 4 nibbles.  An empty or 0
   value for the delimiter interval MUST be interpreted as the default
   value of 1.

3.2.3.  Padding length

   The fourth and final reference option determines the field width of
   the copied value.  Shorter values MUST be padded with leading zeroes
   ("0") and longer values MUST be truncated to the width.

   The default behavior, and that of an explicit empty padding length,
   is that the captured query name substring is copied exactly.  A width
   of zero "0" is a signal to "unpad", and any leading zeros MUST be
   removed. [ Unnecessary complexity? ]

   If a delimiter interval greater than 1 is used, captured values
   between the intervals will be concatenated and the padding or
   unpadding applied as a unit and not individually.  An example of this
   would be ${*||4|4} which would combine each range of 4 captured
   values and pad or truncate them to a width of 4 characters.

   [ If this is kept, the element/feature should probably be renamed
   from "padding" since it is just as likely to truncate. ]

3.2.4.  Final processing

   The string that results from all replacements is converted to the
   appropriate RDATA format for the record type.  If the conversion
   fails, the SERVFAIL rcode MUST be set on the response.

   The TTL of each RR generated by a BULK RR is the TTL of the
   corresponding BULK record itself.  [ BULK should probably have its
   own TTL field because using that of the record itself feels like bad
   design.  On the other hand, if BULK is never meant to be queried for
   directly and only appears in authoritative data, its own TTL is
   pretty useless normally. ]

   If the generated record type is one that uses domain names in its
   resource record data, such as CNAME, a relative domain names MUST be
   fully qualified with the origin domain of the BULK RR.

4.  The NPN Resource Record

   The Numeric Pattern Normalization (NPN) resource record provides pre-
   processing information to reduce the number of possible variants that
   can be generated by a BULK RR into one signable record.  By
   identifying parts of the dynamic resource record which should be
   ignored or represented as a static value, one exemplar record and
   signature is used to validate all other records that match the
   pattern.

   For example, a pattern replacement like pool-A-${1}-${2}.example.com
   that generates PTR records for pool-A-0-0.example.com through pool-
   A-255-255.example.com would have an NPN record that signals a
   validating resolver to verify all pool-A-#-#.example.com names
   against a record for pool-A-9-9.example.com.

   Though it is imperfect in that forged records could be validated as
   legitimate, it is nevertheless an improvement over the security
   afforded by non-DNSSEC DNS.

   The Type value for the NPN RR type is TBD.

   The NPN RR is class independent and has no special TTL requirements.

4.1.  NPN RDATA Wire Format

   The RDATA for an NPN RR consists of a 2 octet Match Type field, a 1
   octet Flags field, a 1 octet Owner Ignore field, a 1 octet Left
   Ignore field and a 1 octet Right Ignore field.

                        1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Match Type          |     Flags     |  Owner Ignore |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Left Ignore  |  Right Ignore |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Match Type indicates the type of the RRset with which this record is
   associated.

   Flags defines additional processing parameters for data
   normalization.  This document defines only the Period-As-Number flag
   "." (position 5), the Hyphen-As-Number "-" (position 6) and the
   hexadecimal flag "X" (position 7).  All other flags are reserved for
   future use.

    0 1 2 3 4 5 6 7
   +-+-+-+-+-+-+-+-+
   |Reserved |.|-|X|
   +-+-+-+-+-+-+-+-+

   Bits 0-4: Reserved for future

   Bit    5: Period As Number (.) Flag
      If 0, periods are treated as non-digits.
      If 1, periods will be processed as digits.

   Bit    6: Hyphen As Number (-) Flag
      If 0, hyphens are treated as non-digits.
      If 1, hyphens will be processed as digits.

   Bit    7: Hexadecimal (X) Flag
      If 0, numeric digits include only 0-9.
      If 1, numeric digits include a-f in addition to 0-9.

   Owner Ignore defines the number of octets in the owner name, as
   counted from the left, which MUST be ignored by the normalization
   process.  This field offers additional security to pattern based
   signatures which may not be immediately apparent.  By restricting the
   leftmost characters defined by this value, ultimately the length of
   the generated portion of the accompanying BULK RR will be confined
   accordingly.

   Left Ignore defines the number of octets of the generated RDATA, as
   counted from the left, which MUST be ignored by the normalization
   process.

   Right Ignore defines the number of octets of the generated RDATA, as
   counted from the right, which MUST be ignored by the normalization
   process.

4.2.  The NPN RR Presentation Format

   Match Type is represented as an RR type mnemonic or with [RFC3597]'s
   generic TYPE mechanism.

   Flags is a string of characters indicating the status of each bit as
   per the following table.  The characters can appear in any order.

   +------------------+-----------+-----------+
   |       Flag       |   Unset   |    Set    |
   +------------------+-----------+-----------+
   | Period As Number |           |     .     |
   +------------------+-----------+-----------+
   | Hyphen As Number |           |     -     |
   +------------------+-----------+-----------+
   |   Hexadecimal    |     9     |     f     |
   +------------------+-----------+-----------+

   Owner Ignore, Left Ignore, and Right Ignore are displayed as unsigned
   decimal integers, ranging from 0 through 255.

4.3.  Use and Normalization Processing of NPN RRs

   [ This section needs reworking still, and should perhaps be pulled
   out into a separate document.  Notably one of issues that is not
   really described well is that, as designed so far, at signing time
   the NPN record has to be associated with the matching BULK record,
   which is slightly problematic with regard to the idea that NPNs are
   suggested to be extended to be used in the future with other
   patterns-based record generation.  Once the appropriate BULK record
   is selected, the signer would then have to understand its semantics
   to fake up the exemplar to sign - raising the question as to why it
   doesn't also know the appropriate values for the Ignore fields, since
   it will have to understand what the static and variable parts are.

   One way around all this is to just sign the BULK record itself and
   return it in the additional section along with the answer, so that
   the resolver could validate not only a signature but the resulting
   record based on the substitution algorithm.  It'd still be
   problematic for older DNSSEC validators that don't grok BULK, but no
   more so than not grokking NPN.  Unfortunately to them in both cases
   the type-appropriate answer itself will be unsigned and thus fail
   validation. ]

   This document provides a minor yet significant modification to DNSSEC
   regarding how RRsets will be signed or verified.  Specifically the
   Signature Field of [RFC4034], Section 3.1.8.  Prior to processing
   into canonical form, signed zones may contain associated RRs where;
   owner, class and type of a non NPN RR directly corresponds with an
   NPN RR matching owner, class and Match Type.  If this condition
   exists the NPN RR's RDATA defines details for processing the
   associated RDATA into a "Normalized" format.  Normalized data is
   based on pre-canonical formatting and zero padded for "A" and "AAAA"
   RR types for acceptable precision during the process.  This concept
   will become clearer in the NPN pseudocode and examples provided in
   the sections to follow.

   The rules for this transformation are simple:

   o  For RR's Owner field, characters from the beginning to the index
      of the Owner Ignore value or the final string of characters
      belonging to the zone's ORIGIN MUST NOT be modified by this
      algorithm.  While the Owner Ignore value is not used for BULK
      records but is included with the expectation other pattern-based
      resource records may emerge and leverage NPN records for their
      DNSSEC support requirements.

   o  For RR's RDATA field, character from beginning to the index of
      Left Ignore value or characters with index of Right Ignore value
      to the end MUST NOT be modified by this algorithm.

   o  In the remaining portion of both Owner and RDATA strings of
      numeric data, defined as character "0" through "f" or "0" through
      "9" depending on whether or not the Hexadecimal flag is set or
      not, MUST be consolidated to a single character and set to the
      highest value defined by the Hexadecimal flag.  Examples may be
      found in the following section.  If period-as-number or hyphen-as-
      number flags are set whichever are used ("." or "-") would be
      treated as part of the number and consolidated where appropriate.

   Once the normalization has been performed the signature will continue
   processing into canonical form using the normalized RRs in the place
   of original ones.

   NPN RRs MAY be included in the "Additional" section to provide a hint
   of the NPN processing required for the verification path.

   It is important to note, properly sizing the Ignore fields is
   critical to minimizing the risk of spoofed signatures.  Never
   intentionally set all Ignore values to zero in order to make
   validation easier as it places the validity of zone data at risk.
   Only accompany RRs which are pattern derived (such as BULK) with NPN
   records as doing so may unnecessarily reduce the confidence level of
   generated signatures.

4.3.1.  Pseudocode for NPN Normalization Processing

   This section provides a simple demonstration of process flow for NPN
   rdata normalization and DNSSEC signatures.

   The pseudocode provided below assumes all associated RRs are valid
   members of a DNSSEC-compatible RRset, including BULK generated ones.

      for rr in rrset
          if (has_NPN<rr.owner, rr.class, rr.type>)
              rr.rdata_normal = NPN_normalize<rr.rdata>
              add_to_sigrrset<NPN.owner, rr.class, rr.type,
                  rr.rdata_normal>
              next
          else
              add_to_sigrrset<rr.owner, rr.class, rr.type, rr.rdata>
              next

      process_canonical_form<sigrrset>

      dnssec_sign<sigrrset>

   Similar logic MUST be used for determining DNSSEC validity of RRsets
   in validating nameservers for signatures generated based on NPN
   normalization.

4.4.  Pattern Based DNSSEC support

   The NPN resource record could be used to support other dynamic RR
   types which do not currently exist.

5.  Known Limitations

   This section defines known limitations of the BULK resource type.

5.1.  Unsupported Nameservers

   Authoritative nameservers that do not understand the semantics of the
   new record type will not be able to deliver the intended answers even
   when the type appears in their zone data This significantly affects
   the interoperability of primary versus secondary authorities that are
   not all running the same software.  Adding new RRs which affect
   handling by authoritative servers, or being unable to add them, is an
   issue that needs to be explored more thoroughly within dnsop.

   On the resolver side, rolling out a new semantics in DNSSEC has also
   proven to be difficult in the past.  Lacking NPN support effectively
   means that operators using BULK will have to forego DNSSEC signing of
   the affected zones, or do online signing of the dynamically generated
   records.

6.  Security Considerations

   Two known security considerations exist for the BULK resource record,
   DNSSEC and DDOS attack vectors.

6.1.  DNSSEC Signature Strategies

   DNSSEC was designed to provide validation for DNS resource records.
   In a nutshell this requires each (owner, class, type) tuple to have
   its own signature.  This essentially defeats the purpose of providing
   large generated blocks of RRs in a single RR as each generated RR
   would require its own legitimate RRSIG record.

   In the following sections several options are discussed to address
   this issue.  Of the options, on-the-fly provides the most secure
   solution and NPN provides the most flexible.

6.1.1.  On-the-fly Signatures

   This solution requires authoritative nameservers to sign generated
   records as they are created.  Not all authoritative nameserver
   implementations offer on-the-fly signatures, and even with those that
   do not all operators will want to keep signing keys online, so this
   solution would either require all implementations to support on-the-
   fly signing or be ignored by implementations which can not or will
   not comply.

   No changes to validating resolvers is required to support this
   solution.

6.1.2.  Normalized (NPN-Based) Signatures

   This solution provides the most flexible solution as nameservers
   without on-the-fly signing capabilities can still support signatures
   for BULK records.  The down side to this solution is it requires
   additional DNSSEC-aware resolver support.

   It has been pointed out that due to this limitation, creation of
   DNSSEC-signed BULK RRs requiring NPN support SHOULD be formally
   discouraged until such time a respectable percentage (>80%) of
   validating resolvers in-the-wild possess NPN processing capabilities.
   Until that time, on-the-fly signing and unsigned records offer the
   intended capabilities of the BULK specification, while requiring zero
   new features to support RR resolution.  The authors would like to
   encourage opening this door for pattern based technologies such as
   NPN records as a solution to BULK RRs as well as other pattern based
   RRs to come.  Given enough time, enough nameservers will be patched
   and upgraded for unrelated reasons and by means of simple attrition
   can supply a level of inertia and eventually widespread adoption can
   be assumed.

   NPN records are likely to be a topic of great debate as to their own
   security limitations.  It is, however, the authors' belief that while
   any logic which limits the input of digital signatures lessens the
   validity of those signatures, the limitation is minimal and the gain
   is significant.  The main reason for this is as a general rule, RRs
   used in a generic manner such as conventional $GENERATE RRs or
   scripted mass pattern generated RRs have a lesser importance than
   other RRs in managed zones.  These therefore inherently pose less
   risk by means of attack and have a much less reward by defeating
   security measures.

   This being said, care must still be taken to set the Ignore fields
   appropriately to minimize exposure and only use NPN RRs to secure
   pattern-based records such as BULK.

6.1.3.  Non-DNSSEC Zone Support Only

   As a final option zones which wish to remain entirely without DNSSEC
   support may serve such zones without either of the above solutions
   and records generated based on BULK RRs will require zero support
   from recursive (resolving) nameservers.

6.2.  DNSSEC Validator Details

   Verification of DNSSEC signed BULK generated RRs may be performed
   against on-the-fly signatures with zero modification to their
   behavior.  However, verification using NPN records would require
   changes to the logic to incorporate processing RDATA generated by
   BULK logic as described above so the results will be compatible.

6.3.  DDOS Attack Vectors and Mitigation

   As an additional defense against Distributed Denial Of Service (DDOS)
   attacks against recursive (resolving) nameservers it is highly
   recommended shorter TTLs be used for BULK RRs than others.  While
   disabling caching with a zero TTL is not recommended, as this would
   only result in a shift of the attack target, a balance will need to
   be found.  While this document uses 24 hours (86400 seconds) in its
   examples, values between 300 to 900 seconds are likely more
   appropriate and is RECOMMENDED.  What is ultimately deemed
   appropriate may differ from zone to zone and administrator to
   administrator.

   [ I am unclear how this helps DDOS mitigation against anyone at all.
   ]

6.4.  Implications of Large-Scale DNS Records

   The production of such large-scale records in the wild may have some
   unintended side-effects.  These side-effects could be of concern or
   add unexpected complications to DNS based security offerings or
   forensic and anti-spam measures.  While outside the scope of this
   document, implementers of technology relying on DNS resource records
   for critical decision making must take into consideration how the
   existence of such a volume of records might impact their technology.

   Solutions to the magnitude problem for BULK generated RRs are
   expected be similar if not identical to that of existing wildcard
   records, the core difference being the resultant RDATA will be unique
   for each requested Domain Name within its scope.

   The authors of this document are confident that by careful
   consideration, negative_side-effects produced by implementing the
   features described in this document can be eliminated from any such
   service or product.

7.  Privacy Considerations

   Neither the BULK nor NPN records introduce any new privacy concerns
   to DNS data.

8.  IANA Considerations

   IANA is requested to assign numbers for two DNS resource record types
   identified in this document: BULK and NPN.

9.  Acknowledgments

   This document was created as an extension to the DNS infrastructure.
   As such, many people over the years have contributed to its creation
   and the authors are appreciative to each of them even if not thanked
   or identified individually.

   A special thanks is extended for the kindness, wisdom and technical
   advice of Robert Whelton (CenturyLink, Inc.) and Gary O'Brien
   (Secure64).

10.  References

10.1.  Normative References

   [RFC1034]  Mockapetris, P., "Domain names - concepts and facilities",
              STD 13, RFC 1034, DOI 10.17487/RFC1034, November 1987,
              <http://www.rfc-editor.org/info/rfc1034>.

   [RFC1035]  Mockapetris, P., "Domain names - implementation and
              specification", STD 13, RFC 1035, DOI 10.17487/RFC1035,
              November 1987, <http://www.rfc-editor.org/info/rfc1035>.

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119,
              DOI 10.17487/RFC2119, March 1997,
              <http://www.rfc-editor.org/info/rfc2119>.

   [RFC2181]  Elz, R. and R. Bush, "Clarifications to the DNS
              Specification", RFC 2181, DOI 10.17487/RFC2181, July 1997,
              <http://www.rfc-editor.org/info/rfc2181>.

   [RFC2308]  Andrews, M., "Negative Caching of DNS Queries (DNS
              NCACHE)", RFC 2308, DOI 10.17487/RFC2308, March 1998,
              <http://www.rfc-editor.org/info/rfc2308>.

   [RFC2317]  Eidnes, H., de Groot, G., and P. Vixie, "Classless IN-
              ADDR.ARPA delegation", BCP 20, RFC 2317,
              DOI 10.17487/RFC2317, March 1998,
              <http://www.rfc-editor.org/info/rfc2317>.

   [RFC3597]  Gustafsson, A., "Handling of Unknown DNS Resource Record
              (RR) Types", RFC 3597, DOI 10.17487/RFC3597, September
              2003, <http://www.rfc-editor.org/info/rfc3597>.

   [RFC4033]  Arends, R., Austein, R., Larson, M., Massey, D., and S.
              Rose, "DNS Security Introduction and Requirements",
              RFC 4033, DOI 10.17487/RFC4033, March 2005,
              <http://www.rfc-editor.org/info/rfc4033>.

   [RFC4034]  Arends, R., Austein, R., Larson, M., Massey, D., and S.
              Rose, "Resource Records for the DNS Security Extensions",
              RFC 4034, DOI 10.17487/RFC4034, March 2005,
              <http://www.rfc-editor.org/info/rfc4034>.

   [RFC4035]  Arends, R., Austein, R., Larson, M., Massey, D., and S.
              Rose, "Protocol Modifications for the DNS Security
              Extensions", RFC 4035, DOI 10.17487/RFC4035, March 2005,
              <http://www.rfc-editor.org/info/rfc4035>.

   [RFC5234]  Crocker, D., Ed. and P. Overell, "Augmented BNF for Syntax
              Specifications: ABNF", STD 68, RFC 5234,
              DOI 10.17487/RFC5234, January 2008,
              <http://www.rfc-editor.org/info/rfc5234>.

10.2.  Informative References

   [bind-arm]
              Internet Systems Consortium, "BIND 9 Configuration
              Reference", 2016,
              <https://ftp.isc.org/isc/bind9/cur/9.9/doc/arm/
              Bv9ARM.html>.

   [RFC7719]  Hoffman, P., Sullivan, A., and K. Fujiwara, "DNS
              Terminology", RFC 7719, DOI 10.17487/RFC7719, December
              2015, <http://www.rfc-editor.org/info/rfc7719>.

Appendix A.  BULK Examples

A.1.  Example 1

   $ORIGIN 2.10.in-addr.arpa.
   @ 86400 IN BULK PTR (
             [0-255].[0-255].[0-255].[0-255].in-addr.arpa.
             pool-${4-1}.example.com.
           )

   A query received for the PTR of 4.3.2.10.in-addr.arpa will create the
   references ${1} through ${4} with the first four labels of the query
   name.  The ${4-1} reference in the replacement pattern will then
   substitute them in reverse with the default delimiter of hyphen
   between every character and no special field width modifications.
   The TTL of the BULK RR is used for the generated record, making the
   response:

   4.3.2.10.in-addr.arpa 86400 IN PTR pool-10-2-3-4.example.com.

A.2.  Example 2

   $ORIGIN 2.10.in-addr.arpa.
   @ 86400 IN BULK PTR (
             [0-255].[0-255].[0-255].[0-255].in-addr.arpa.
             pool-${2,1|||3}.example.com.
           )

   Example 2 is similar to Example 1, except that it modifies the
   replacement pattern.  The empty option after the first vertical bar
   causes no delimiters to be inserted, while the second empty option
   that would keep the delimiter interval as 1.  The latter is relevant
   because the final value, padding of 3, is applied over each delimiter
   interval even when no delimiter is used.  Not all captures from the
   substring are required to be used in the response.

   The result is that a query for the PTR of 4.3.2.10.in-addr.arpa
   generates this response:

   4.3.2.10.in-addr.arpa 86400 IN PTR pool-003004.example.com.

   [ Admittedly you can't do this very effectively without the field
   width complexity.  Is this sort of name common?  Does it need
   support?  Admittedly $GENERATE had the feature, but is that reason
   enough? ]

   [ Change this to a hex matching example? ]

A.3.  Example 3

   This example contains a classless IPv4 delegation on the /22 CIDR
   boundary as defined by [RFC2317].  The network for this example is
   "10.2.0/22" delegated to a nameserver "ns1.sub.example.com.".  RRs
   for this example are defined as:

   $ORIGIN 2.10.in-addr.arpa.
   @    7200 IN BULK CNAME [0-255].[0-3] ${*|.}.0-3
   0-3 86400 IN NS ns1.sub.example.com.

   A query for the PTR of 25.2.2.10.in-addr.arpa is received and the
   BULK record with the CNAME Match Type matches all query types.  25
   and 2 are captured as references, and joined in the answer by the
   period (".") character as a delimiter, with ".0-3" then appended
   literally and fully qualified by the origin domain.  The final
   synthesized record is:

   25.2.2.10.in-addr.arpa 7200 IN CNAME 25.2.0-3.2.10.in-addr.arpa.

   [ Without $* and options complexity, the pattern to get the same
   result is just ${1}.{$2}.0-3 which is not really significantly
   onerous to enter, and slightly less arcane looking to comprehend. ]

Appendix B.  NPN Examples

B.1.  EXAMPLE 1

   2.10.in-addr.arpa. 86400 IN BULK PTR (
                                    [0-255].[0-10].2.10.in-addr.arpa.
                                    pool-A-${1}-${2}.example.com.
                               )
   2.10.in-addr.arpa. 86400 IN NPN  PTR 9 0 7 13

   A query for the PTR of address 10.2.3.44 would match As shown
   previously in BULK RR examples the query would match the BULK record
   for the query name 44.3.2.10.in-addr.arpa, generating a PTR to pool-
   A-3-44.example.com as the answer.

   By protecting the "Ignore" characters from the generated RR's RDATA
   the focus for normalization becomes "3-44" as illustrated below.

     0 1 2 3 4 5 6 7
                   v---------
       p o o l - A - 3 - 4 4 . e x a m p l e . c o m .
                    ---------^
                             1 1 1 1
                             3 2 1 0 9 8 7 6 5 4 3 2 1 0

   Everything to the left of "3-44" will remain intact, as will
   everything to its right.  The remaining characters will be processed
   for digits between 0 and 9 as indicated by the NPN record's
   hexadecimal flag 9, and each run of digits replaced by the single
   character "9".  The final Normalized RDATA for *.2.10.in-addr.arpa
   would therefore become pool-A-9-9.example.com., and its signature
   would be based on this value to cover all possible permutations of
   the pool-A-${1}-${2}.example.com replacement pattern.

   Since the validating nameserver would use the identical NPN record
   for processing and comparison, all RRs generated by the BULK record
   can now be verified with a single signature.

B.2.  EXAMPLE 2

   This example contains a classless IPv4 delegation on the /22 CIDR
   boundary as defined by [RFC2317].  The network for this example is
   "10.2.0/22" delegated to a nameserver "ns1.sub.example.com.".  RRs
   for this example are defined as:

   $ORIGIN 2.10.in-addr.arpa.
   0-3 86400 IN      NS    ns1.sub.example.com.
       86400 IN BULK CNAME [0-255].[0-3] ${*|.}.0-3
       86400 IN NPN  CNAME 9 0 0 23

   For this example, a query of "10.2.2.65" would enter the nameserver
   as "65.2.2.10.in-addr.arpa." and a "CNAME" RR with the RDATA of
   "65.2.0-3.2.10.in-addr.arpa." would be generated.

   By protecting the "Ignore" characters from the generated RR's RDATA
   the focus for normalization becomes "65.2" as illustrated below.

          0
          v---------
            6 5 . 2 . 0 - 3 . 2 . 1 0 . i n - a d d r . a r p a .
           ---------^
                    2 2 2 2 1 1 1 1 1 1 1 1 1 1
                    3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0

   Everything to the left of "65.2" will remain intact as will
   everything to its right.  The remaining characters will be processed
   for digits from 0 through 9 as indicated by the NPN record's
   hexadecimal flag "9", with each run replaced by the single character
   "9".  The final normalized RDATA would therefore become 9.9.0-
   3.2.10.in-addr.arpa and its signature would be based on this
   normalized RDATA field.  This new normalized string would be used as
   an RDATA for the wildcard label of 2.10.in-addr.arpa now encompassing
   all possible permutations of the ${*|.}.0-3.2.10.in-addr.arpa.
   pattern.

   As in example 1, the validatating resolver would use the same NPN
   record for comparison.

B.3.  EXAMPLE 3

   This example provides reverse logic for example 1 by providing an
   IPv4 address record for a requested hostname.  For this example the
   query is defined as an A record for pool-A-3-44.example.com, with an
   origin of example.com.  RRs for this example are defined as:

   example.com. 86400 IN BULK A (
                                      pool-A-[0-10]-[0-255]
                                      10.2.${*}
                                     )
   example.com. 86400 IN NPN  A 9 0 8 0

   By protecting the "Ignore" characters from the generated RR's RDATA
   the focus for normalization becomes "003.044" as illustrated below.

                   0 1 2 3 4 5 6 7 8
                                   v--------------
                     0 1 0 . 0 0 2 . 0 0 3 . 0 4 4
                                    ---------------^
                                                   0

   This example illustrates a key point about NPN records; since they
   are pre-canonical they MUST operate on a strict subset of WIRE
   formatted data.  For A and AAAA records this means the "Ignore"
   fields are based on zero padded data.  In this example our generated
   record MUST be converted into "010.002.003.044" (shown above) prior
   to processing.  After processing, wire format would become
   "0x0A02032C" (shown in hexadecimal).  This format would be too
   imprecise for normalization so padded decimal is used.

   Everything to the left of "003.044" will remain intact as will
   everything to its right.  The remaining characters will be processed
   for digits between 0 and 9 as indicated by the NPN record's
   hexadecimal flag "9" and each run replaced by the single character
   "9".  The final Normalized RDATA would therefore become 10.2.9.9 and
   its signature would be based on this normalized RDATA field.  This
   new normalized A RR would be used as an RDATA for the wildcard label
   of "_.example.com." now encompassing all possible permutations of the
   10.2.${_} pattern.

B.4.  EXAMPLE 4

   This example provides similar logic for an IPv6 AAAA record.  For
   this example the query is defined as an AAAA record for pool-A-ff-
   aa.example.com with an origin of example.com..  RRs for this example
   are defined as:

   example.com. 86400 IN BULK AAAA (
                                      pool-A-[0-ffff]-[0-ffff]
                                      fc00::${1}:${2}
                                   )
   example.com. 86400 IN NPN  AAAA X 0 30 0

   By protecting the "Ignore" characters from the generated RR's RDATA
   the focus for normalization becomes "00ff:00aa" as illustrated below.

                         1 1 1 1 1 1 1 1 1 1 2 2
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1

       f c 0 0 : 0 0 0 0 : 0 0 0 0 : 0 0 0 0 : -/-/

     4 3 3 3 3 3 3 3 3 3 3 2 2 2 2 2 2 2 2 2 2 1
     0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9
      /-/-/- . . . . . . . . . . . . . . . . . . . . . . . . -/-/-/
                             2 2 2 2 2 2 2 2 2 3 3 3 3 3 3 3 3 3 3 4
                             1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0
                                               v------------------
                        /-/- 0 0 0 0 : 0 0 0 0 : 0 0 f f : 0 0 a a
                                                -------------------^
                           2 1 1 1 1 1 1 1 1 1 1
                           0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0

   This example reinforces the point on pre-canonical processing of NPN
   records; they MUST operate on a strict subset of WIRE formatted data.
   For A and AAAA records this means the "Ignore" fields are based on
   zero padded data.  In this example our generated record MUST be
   converted into "fc00:0000:0000:0000:0000:0000:00ff:00aa" (shown
   above) prior to processing.  After processing, wire format would
   become "0xFC000000000000000000000000FF00AA" (shown in hexadecimal).
   This format is slightly misleading as it is truly only 16 bytes of
   WIRE data and would be too imprecise for normalization so padded
   hexadecimal is used.

   Everything to the left of "00ff:00aa" will remain intact as will
   everything to its right.  The remaining characters will be processed
   for numbers between "0" and "f" as indicated by the NPN record's
   hexadecimal flag "X" and each run replaced by the single character
   "f".  The final Normalized RDATA would therefore become "fc00::f:f"
   and its signature would be based on this "normalized" RDATA field.
   This new "normalized" "AAAA" RR would be used as an RDATA for the
   wildcard label of *.example.com now encompassing all possible
   permutations of the "fc00::${1}:${2}" pattern.

Authors' Addresses

   John Woodworth
   CenturyLink, Inc.
   4250 N Fairfax Dr
   Arlington  VA 22203
   USA

   Email: John.Woodworth@CenturyLink.com


   Dean Ballew
   CenturyLink, Inc.
   2355 Dulles Corner Blvd, Ste 200 300
   Herndon  VA 20171
   USA

   Email: Dean.Ballew@CenturyLink.com


   Shashwath Bindinganaveli Raghavan
   Hughes Network Systems
   11717 Exploration Lane
   Germantown  MD 20876
   USA

   Email: shashwath.bindinganaveliraghavan@hughes.com


   David C Lawrence
   Akamai Technologies
   150 Broadway
   Cambridge  MA 02142-1054
   USA

   Email: tale@akamai.com

```
