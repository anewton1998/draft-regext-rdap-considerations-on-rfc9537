%%%
Title = "Considerations on RFC 9537"
area = "Applications and Real-Time Area (ART)"
workgroup = "Registration Protocols Extensions (regext)"
abbrev = "rfc9537-considered"
ipr= "trust200902"

[seriesInfo]
name = "Internet-Draft"
value = "draft-newton-regext-rdap-considerations-on-rfc9537-00"
stream = "IETF"
status = "informational"
date = 2024-05-29T00:00:00Z

[[author]]
initials="A."
surname="Newton"
fullname="Andy Newton"
organization="ICANN"
[author.address]
email = "andy@hxr.us"

%%%

.# Abstract

This document discusses client implementation issues relating to RFC 9537, “Redacted Fields in
the Registration Data Access Protocol (RDAP) Response”. The considerations in this document
have arisen from problems raised by two separate teams attempting to implement RFC 9537 in
both an RDAP web client and an RDAP command line client. Some of these problems may be
insurmountable, leaving portions of RFC 9537 non-interoperable between clients and servers,
while other problems place a high degree of complexity upon clients.

{mainmatter}

# Background

[@!RFC9537] is an RDAP extension designed to allow the programmatic designation of information
in RDAP responses as redacted, where the policy of the RDAP server designates information to
be purposefully withheld from view. Quoting from Section 1:

> Because an RDAP response may exclude a field due to either the lack of data or
> the lack of RDAP client privileges, this extension is used to explicitly specify which
> RDAP fields are not included in the RDAP response due to redaction.

[@!RFC9537] specifies these redactions with JSONPath ([@!RFC9535]), an expression language
designed to produce one or more JSON values from a JSON document. These JSONPath
expressions are given in three distinct strings per redaction directive: “prePath”, “postPath”, and
“replacementPath”. This RFC also modifies the IANA RDAP JSON Values registry with additional
fields to be used in the process of presenting information about a redaction, however the
registry contains text intended for humans and does not explicitly contain JSONPath expressions.

[@!RFC9537] defines four different methods of redaction: removal, empty value, partial value, and
replacement value. Redaction by removal indicates that the redacted information is not in the
RDAP JSON response. Redaction by empty value uses JSONPath to identify empty JSON values,
and redaction by partial value does the same but for JSON values that are partially empty.
Finally, redaction by replacement value instructs clients that a server has substituted
a redacted value with other data in the RDAP JSON response.

For redaction to be useful, a client must be able to show a user the information being redacted.
This can be done in many ways depending on the client’s environment. For many clients, the
method can be to make visual representations, such as changing a color or showing an icon,
in the place where the information would normally be expected to be found by a user or near that place.
Clients may also use audio cues in a similar fashion, especially those for the visually impaired.
That is, clients merely rendering the values of "prePath", "postPath", or "replacementPath"
are of little use to most humans.

The information in this document was obtained through the experience of attempting to implement
[@!RFC9537] in an RDAP web client, <https://lookup.icann.org>, and in an RDAP command-line
interface client, <https://github.com/icann/icann-rdap>. As part of this effort, the server-side 
aspects of RFC 9537 were implemented in the RDAP server also at <https://github.com/icann/icann-rdap>.
Finally, a corpus of test cases for RDAP responses was created at 
<https://github.com/anewton1998/redacted_examples>.

# Redaction by Removal

This is the default redaction method in [@!RFC9537], and the most difficult to implement.

In this method, the portions of the RDAP JSON response that are redacted are not present but
may be specified in the “prePath” string. Because the information is removed from the response,
the client will yield an empty set when evaluating the “prePath” expression against the
response. In other words, the client has no means to identify the 
information that has been redacted using the value of “prePath”.

Further, [@!RFC9537] makes "prePath" OPTIONAL and does not
require its usage, though every example in the RFC of redaction by removal does use "prePath".

[@!RFC9537] gives the following advice to the client in Section 5.1:

> When the server is using the Redaction by Removal Method or the Redaction by
> Replacement Value Method with an alternative field value, the JSONPath
> expression of the "prePath" member will not resolve successfully with the
> redacted response. The client can key off the "name" member for display logic
> related to the redaction.

In other words, the client may be able to know the information that was removed using the “name”
member of the redaction directive described in Section 4.2 of [@!RFC9537], though the RFC
does not explain how the client is to go about doing this nor does it use normative RFC language
to indicate this is the nature of the mechanism to use. 

The “name” member is a JSON object that takes either a “description” JSON string or a “type” JSON string:

```
“name” : {
  “type”: “Registered Value”
}
```
or
````
“name” : {
  “description”: “Unregistered Value”
}
````

The “type” string contains a value registered with IANA in the RDAP JSON Values registry. This
registry does not contain any formalism to describe the data to be redacted. The “description”
string contains any text and has no definition. Therefore, RFC 9537 does not describe a
mechanism for a client implementer to "explicitly" determine the information that is to be redacted.

Client implementers must therefore devise a method not given by RFC 9537.

One possible solution is to have the client dynamically generate a JSONPath expression for each
value it would normally present and then compare that value against the “prePath” string.
However, this would require the “prePath” string to contain a JSONPath expression that only
evaluates to one JSON value, would require an algorithm to canonicalize JSONPath expressions
for comparison of the dynamically generated expression to the “prePath” string, and would
require all clients and servers to formally agree to the position of all items in all RDAP arrays.

Another possible solution would be to have the client dynamically construct an RDAP JSON
response based on the “prePath” string. However, as JSONPath contains wildcards (e.g. “[*]”),
recursion (e.g. “[..]”) and unbounded array slices (e.g. “[0:]”), this could lead to infinite
length JSON objects and arrays and is practically unworkable.

Both these solutions are theoretical, and each would be highly complex to implement.
Therefore, for all practical purposes a client has no known method to "explicitly" identify
the values of an RDAP response that have been redacted.

Finally, the implication of the "prePath" expression is that it should always evaluate
to an empty set. Left undefined is behavior when that is not the case nor any specific
guidance to server implementers to verify it always evaluates to an empty set.

# Redaction by Replacement Value

The redaction by replacement value method has two modes. In the first mode, it signals that
values that would have been in the RDAP JSON response have been replaced with other values
that are in the RDAP JSON response. Like redaction by removal, a client has no formal means to
determine the information that would have been in the response as this method relies on
the "prePath" expression. Nor does RFC 9537 provide guidance to client implementers as to 
the purpose of signaling the replacement (i.e. what are clients to do with this information).

In the second mode, the “replacementPath” identifies values in an RDAP JSON response that
are present but have been replaced with unidentified values. The example given in the RFC is
that of an email address that has been anonymized. There is no guidance given to client
implementers as to the purpose of this information. That is, the information is present in the
response but there is no formal definition of what has been replaced.

It is possible a client could present to a user the description of the redaction directive. However,
the structure of the redaction JSON does not allow for multiple descriptions with differing
“lang” tags and therefore this mechanism has some internationalization problems.

# Types and Empty Value

The empty value redaction method signals information has been redacted by identifying, with
JSONPath, the value or values which have been redacted. The RFC states the following:

> The Redaction by Empty Value Method is when a redacted field is not removed
> but its value is set to an empty value, such as "" for a jCard text ("text") property
> or null for a non-text property.

This text does not fully describe the nature of an empty value for all JSON types, instead
equating an empty string as an empty value for JSON strings and null as the empty value for all
other JSON types. Left to interpretation is the nature of an empty JSON object, which can be
represented as “{}”, and an empty JSON array, which can be represented as “[]”. In other
words, is an empty object “{}” or null?

In addition to this, null is not the same type as other JSON types and therefore cannot be used
anywhere a specific type, such as a boolean or number, has been specified unless explicitly
allowed by the JSON-using specification. That is, if RFC 9083 designates a value as a specific
type a server cannot change that value to a null in an RDAP response and be compliant with 
[@!RFC9083].

Within the core RDAP responses, null is not allowed in any place. Therefore, using it would
create RDAP invalid JSON, which is also not allowed by this RFC as stated in Section 3:

> The resulting redacted RDAP response MUST comply with the format defined in
> the RDAP RFCs, such as RFC 9083 and updates.

For all practical purposes, redaction by empty value can only applied to a JSON string.

# Types and Partial Value

Like the above, redaction by partial value is not restricted to non-string types. However, the RFC
does not describe the semantics of partial value redaction for any but the string type.
Therefore, clients are given no guidance regarding the nature of a partial boolean, number, or
null.

Furthermore, when applying this redaction method to a string, object, or array a client has no
way of determining which parts were redacted. The example given in the RFC is of the string
“Vancouver\nBC\n1239\n" but the client has no means of determining if the redacted portion of
the string comes before, after, or in the middle of this string.

The same issue applies to JSON objects and arrays. Though some readers may deduce that
redaction by partial value only applies to strings, the RFC does not state this.

Therefore, for all practical purposes there is no distinction between redaction by partial value
and redaction by empty value. Furthermore, if a client cannot determine which parts of a string
have been redacted, then it cannot adequately present this information to a user.

# Overlapping and Nesting Redactions

Another issue that is unaddressed by RFC 9537 is that of overlapping JSONPath expressions.
Consider the following example of a domain with one entity in which two redaction directives
are given:

````
{
  "rdapConformance": [
    "rdap_level_0",
    "redacted"
  ],
  "objectClassName": "domain",
  "ldhName": "example-11.net",
  "entities": [
    {
      "objectClassName": "entity",
      "handle": "123",
      "roles": [ "registrar" ],
      "vcardArray": [
        "vcard",
        [
          [
            "version",
            {},
            "text",
            "4.0"
          ],
          [
            "fn",
            {},
            "text",
            "Example Registrar Inc."
          ],
          [
            "email",
            {},
            "text",
            ""
          ]
        ]
      ]
    }
  ],
  "redacted": [
    {
      "name": {
      "description": "Registrar Email"
      },
      "prePath": "$.entities[?(@.roles[0]=='registrar')].vcardArray[1][?(@[0]=='email')]",
      "method": "removal",
      "reason": {
        "description": "Server policy"
      }
    },
    {
      "name": {
        "description": "Registrar Email"
      },
      "prePath": "$.entities[?(@.roles[0]=='registrar')].vcardArray[1][?(@[0]=='email')][3]",
      "method": "emptyValue",
      "reason": {
        "description": "Server policy"
      }
    }
  ]
}
````
Here the client is told that the email address is an empty value and that it does not exist in the
response.

Similar to the issue above, RFC 9537 does not limit redaction directives to the top level of an
RDAP lookup response, therefore it is possible to have redactions specified by both the target of
the lookup (e.g. domain) and its subordinate objects (e.g. entities).

# JSONPath Implementation Conformance

During the development of these RDAP clients, issues of varying conformance to [@!RFC9535] have
arisen with numerous JSONPath libraries. The JSONPath Comparison web page at
<https://cburgmer.github.io/json-path-comparison/> highlights many of the challenges of
compatibility between implementations of JSONPath. 

While JSONPath is very usable in constrained environments, the broad nature in which it is 
used in [@!RFC9537] is problematic as it relates to interoperability between clients and servers.

To help with testing, a corpus of RDAP responses using [@!RFC9537] was created: <https://github.com/anewton1998/redacted_examples>.
As a consequence, shortcomings were discovered in the JSONPath libraries being used and 
developers on both teams attempted various workarounds to no avail. No discernible pattern
was detected with regard to compatibility issues, but every JSONPath library utilized by the teams had
issues.

# Other Considerations

## Tel URIs

There also exist many "corner cases" that server operators need to consider.
Take for example the "tel" property of jCard, which maybe rendered as:

```
[ 
  "tel",
  {
    "type": ["voice"]
  },
  "uri",
  "tel:+1-555-555-1234;ext=1234"
]  
```

Should a server redact the phone extension, how does the tel URI get
expressed: `tel:+1-555-555-1234;ext=` or `tel:+1-555-555-1234`? Is this redaction
by removal or partial value? In both cases, JSONPath does not have the capability
to specify which parts of the string are to be redacted.

Additionally, the "tel" property is not required to be a tel URI but maybe unstructured
text. For example:

```
[ 
  "tel",
  {
    "type": ["voice"]
  },
  "text",
  "1-555-555-1234 x1234"
]  
```

As the string to be redacted by partial value may take many forms, clients have no
formal means of determine which part is to be redacted.

## Structured and Unstructured Addresses

Server implementers should also take care regarding the redaction used for jCard
postal addresses. Within jCard, postal addresses may be expressed as either structured
or unstructured. Here is an example of structured address from [@!RFC9083]:

```
["adr",
  { "type":"work" },
  "text",
  [
    "",
    "Suite 1234",
    "4321 Rue Somewhere",
    "Quebec",
    "QC",
    "G1V 2M2",
    "Canada"
  ]
]
```

And here is an example of an unstructured postal address from [@!RFC9083]:

```
["adr",
  {
    "type":"home",
    "label":"123 Maple Ave\nSuite 90001\nVancouver\nBC\n1239\n"
  },
  "text",
  [
    "", "", "", "", "", "", ""
  ]
],
```

As both the JSONPath and the redaction method are different for each of these, a
server implementer should take care to use the proper values. Additionally,
if the server references an IANA registered redaction, the registration
should cover both cases.

# Complexity

Overall, both teams spent a considerable amount of time in attempts to implement clients
for [@!RFC9537]. Both teams considered [@!RFC9537] highly complex and difficult to implement.
Given this, it is unlikely that any general purpose RDAP client would be implementing
[@!RFC9537] without explicit, directed funding for that purpose. And it is likely that
implementation among clients will differ substantially in behavior.

# Path Forward

To summarize the findings:

* Redaction by removal cannot be accomplished by "explicitly identifying" the parts of an RDAP JSON response that have been removed.
* Redaction by empty value is only applicable to JSON strings.
* Redaction by partial value cannot specify the parts of the value redacted, is only applicable to strings, and therefore no different
than redaction by empty value.
* Redaction by replacement value cannot identify the values that were replaced.
* The usage of JSONPath is too broad, rendering the mechanisms of [@!RFC9537] that do work to be non-interoperable.
* Overlaps in JSONPath expressions can present challenges to clients attempting to display textual explanations.

One potential fix would be to instruct clients to always ignore the JSONPath expressions and instead strictly key
off the redacted registrations in the IANA registry. The implications of this approach is that each registration
would need to clearly specify the parts of RDAP to be redacted, and client implementers would be required to update
their client software as frequently as necessary to accommodate new registrations. In other words, this approach
would not "explicitly identify" redacted RDAP data, nor would it give servers the ability to express redaction reasons
in multiple languages.

# Security Considerations

TBD.

{backmatter}

