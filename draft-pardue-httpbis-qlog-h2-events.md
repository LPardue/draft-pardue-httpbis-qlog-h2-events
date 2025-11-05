---
title: "TODO - Your title"
abbrev: "TODO - Abbreviation"
category: info

docname: draft-pardue-httpbis-qlog-h2-events-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "HTTP"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "LPardue/draft-pardue-httpbis-qlog-h2-events"
  latest: "https://LPardue.github.io/draft-pardue-httpbis-qlog-h2-events/draft-pardue-httpbis-qlog-h2-events.html"

author:
 -
    fullname: "Lucas Pardue"
    organization: Cloudflare
    email: "lucas@lucaspardue.com"

normative:

  QLOG-MAIN:
    I-D.ietf-quic-qlog-main-schema

  RFC9110:
    display: HTTP

  RFC9113:
    display: HTTP/2

informative:

...

--- abstract

This document defines qlog event definitions for HTTP/2 protocol.


--- middle

# Introduction

This document defines a qlog event schema ({{Section 8 of QLOG-MAIN}})
containing concrete events for HTTP/2{{RFC9113}}.

The event namespace with identifier `http2` is defined; see {{schema-def}}. In
this namespace multiple events derive from the qlog abstract Event class
({{Section 7 of QLOG-MAIN}}), each extending the "data" field and defining their
"name" field values and semantics.

{{events-table}} summarizes the name value of each event type that is defined in
this specification. Some event data fields use complex data types. These are
represented as enums or re-usable definitions, which are grouped together on the
bottom of this document for clarity.

| Name value                     | Importance |  Definition |
|:-------------------------------|:-----------|:------------|
| http2:parameters_set           | Core       | {{parametersset}} |
| http2:parameters_restored      | Core       | {{parametersrestored}} |
| http2:frame_created            | Core       | {{framecreated}} |
| http2:frame_parsed             | Core       | {{frameparsed}} |
{: #events-table title="HTTP/2 Events"}


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The event and data structure definitions in ths document are expressed in the
Concise Data Definition Language {{!CDDL=RFC8610}} and its extensions described
in {{QLOG-MAIN}}.

The following fields from {{QLOG-MAIN}} are imported and used: name, namespace,
type, data, group_id, RawInfo, and time-related fields.

Events are defined with an importance level as described in {{Section 8.3 of
QLOG-MAIN}}.

As is the case for {{QLOG-MAIN}}, the qlog schema definitions in this document
are intentionally agnostic to serialization formats. The choice of format is an
implementation decision.

# Event Schema Definition {#schema-def}

This document describes how HTTP/2 is expressed in qlog with an event schema.
Per the requirements in {{Section 8 of QLOG-MAIN}}, this document registers the
`http2` namespace. The event schema URI is `urn:ietf:params:qlog:events:http2`.

## Draft Event Schema Identification
{:removeinrfc="true"}

Only implementations of the final, published RFC can use the events belonging to
the event schema with the URI `urn:ietf:params:qlog:events:http2`. Until such an
RFC exists, implementations MUST NOT identify themselves using this URI.

Implementations of draft versions of the event schema MUST append the string
"-" and the corresponding draft number to the URI. For example, draft 99 of this
document is identified using the URI `urn:ietf:params:qlog:events:http2-99`.

The namespace identifier itself is not affected by this requirement.

# HTTP/2 Events

HTTP/2 events extend the `$ProtocolEventData` extension point defined in
{{QLOG-MAIN}}. Additionally, they allow for direct extensibility by their use of
per-event extension points via the `$$` CDDL "group socket" syntax, as also
described in {{QLOG-MAIN}}.

## parameters_set {#parametersset}

The `parameters_set` event contains HTTP/2 settings, mostly those received from
the SETTINGS frame. It has Core importance level.

Parameters are typically set once but they can be set at different times during
the connection, therefore a qlog can have multiple instances of `parameters_set`
with different fields set.

The "owner" field reflects how Settings are exchanged on a connection. Sent
settings have the value "local" and received settings have the value
"received".

~~~ cddl
HTTP2ParametersSet = {
    ? owner: Owner

    ? header_table_size: uint32
    ? enable_push: uint32
    ? max_concurrent_streams: uint32
    ? initial_window_size: uint32
    ? max_frame_size: uint32
    ? max_header_list_size: uin32

    ? extended_connect: uint32
    ? no_rfc7540_priorities: uint32

    * $$http2-parametersset-extension
}
~~~
{: #parametersset-def title="HTTP2ParametersSet definition"}

The `parameters_set` event can contain any number of unspecified fields. This
allows for representation of reserved settings (aka GREASE) or ad-hoc support
for extension settings that do not have a related qlog schema definition.

## parameters_restored {#parametersrestored}

When using TLS 0-RTT, HTTP/2 clients are expected to remember and reuse the
server's SETTINGs from the previous connection. The `parameters_restored` event
is used to indicate which HTTP/2 settings were restored and to which values when
utilizing 0-RTT. It has Core importance level.

~~~ cddl
HTTP2ParametersRestored = {
    ? owner: Owner

    ? header_table_size: uint32
    ? enable_push: uint32
    ? max_concurrent_streams: uint32
    ? initial_window_size: uint32
    ? max_frame_size: uint32
    ? max_header_list_size: uin32

    ? extended_connect: uint32
    ? no_rfc7540_priorities: uint32

    * $$http2-parametersrestored-extension
}
~~~
{: #parametersrestored-def title="HTTP2ParametersRestored definition"}

## frame_created {#framecreated}

The `frame_created` event is emitted when the HTTP/2 framing actually happens.
It has Core importance level.

~~~ cddl
HTTP2FrameCreated = {
    stream_id: uint32
    ? length: uint32
    frame: $HTTP2Frame
    ? raw: RawInfo

    * $$http2-framecreated-extension
}
~~~
{: #framecreated-def title="HTTP2FrameCreated definition"}

## frame_parsed {#frameparsed}

The `frame_parsed` event is emitted when the HTTP/2 frame is parsed. It has Core
importance level.

~~~ cddl
HTTP2FrameParsed = {
    stream_id: uint64
    ? length: uint64
    frame: $HTTP2Frame
    ? raw: RawInfo

    * $$http2-frameparsed-extension
}
~~~
{: #frameparsed-def title="HTTP2FrameParsed definition"}

# HTTP/2 Data Type Definitions

The following data type definitions can be used in HTTP/2 events.

## Owner

~~~ cddl
Owner = "local" /
        "remote"
~~~
{: #owner-def title="Owner definition"}

## HTTP2Frame

The generic `$HTTP2Frame` is defined here as a CDDL "type socket" extension point.
It can be extended to support additional HTTP/2 frame types.

~~~~~~
; The HTTP2Frame is any key-value map (e.g., JSON object)
$HTTP2Frame /= {
    * text => any
}
~~~~~~
{: #h2-frame-def title="HTTP2Frame type socket definition"}

The HTTP/2 frame types defined in this document are as follows:

~~~ cddl
HTTP2BaseFrames = HTTP2DataFrame /
                  HTTP2HeadersFrame /
                  HTTP2PriorityFrame /
                  HTTP2RstStreamFrame /
                  HTTP2SettingsFrame /
                  HTTP2PushPromiseFrame /
                  HTTP2PingFrame /
                  HTTP2GoawayFrame /
                  HTTP2WindowUpdateFrame /
                  HTTP2ContinuationFrame /
                  HTTP2ReservedFrame /
                  HTTP2UnknownFrame

$HTTP2Frame /= HTTP2BaseFrames
~~~
{: #baseframe-def title="HTTP2BaseFrames definition"}

### HTTP2DataFrame

~~~ cddl
HTTP2DataFrame = {
    frame_type: "data"
    ? raw: RawInfo
}
~~~
{: #dataframe-def title="HTTP2DataFrame definition"}

### HTTP2HeadersFrame

The payload of an HTTP/2 HEADERS frame is the HPACK-encoding of an HTTP field
section; see {{?RFC7541}}. `HTTP2HeaderFrame`, in contrast, contains the HTTP
field section without encoding.

~~~ cddl
HTTP2HTTPField = {
    ? name: text
    ? name_bytes: hexstring
    ? value: text
    ? value_bytes: hexstring
}
~~~
{: #h2field-def title="HTTP2HTTPField definition"}

~~~ cddl
HTTP2HeadersFrame = {
    frame_type: "headers"
    headers: [* HTTP2HTTPField]
    ? raw: RawInfo
}
~~~
{: #headersframe-def title="HTTP2HeadersFrame definition"}

For example, the HTTP field section

~~~
:path: /index.html
:method: GET
:authority: example.org
:scheme: https
~~~

would be represented in a JSON serialization as:

~~~
headers: [
  {
    "name": ":path",
    "value": "/"
  },
  {
    "name": ":method",
    "value": "GET"
  },
  {
    "name": ":authority",
    "value": "example.org"
  },
  {
    "name": ":scheme",
    "value": "https"
  }
]
~~~
{: #headersframe-ex title="HTTP2HeadersFrame example"}

{{RFC9113}} and {{Section 5.1 of RFC9110}} define rules for the
characters used in HTTP field sections names and values. Characters outside the
range are invalid and result in the message being treated as malformed. It can
however be useful to also log these invalid HTTP fields. Characters in the
allowed range can be safely logged by the text type used in the `name` and
`value` fields of `HTTP2HTTPField`. Characters outside the range are unsafe for the
text type and need to be logged using the `name_bytes` and `value_bytes` field.
An instance of `HTTP2HTTPField` MUST include either the `name` or `name_bytes`
field and MAY include both. An `HTTP2HTTPField` MAY include a `value` or
`value_bytes` field or neither.

### HTTP2SettingsFrame

The settings field can contain zero or more entries. Each setting has a name
field, which corresponds to Setting Name as defined (or as would be defined if
registered) in the "HTTP/3 Settings" registry maintained at
<https://www.iana.org/assignments/HTTP2-parameters>.

An endpoint that receives unknown settings is not able to log a specific name.
Instead, the name value of "unknown" can be used and the value captured in the
`name_bytes` field; a numerical value without variable-length integer encoding.

~~~ cddl
HTTP2SettingsFrame = {
    frame_type: "settings"
    settings: [* HTTP2Setting]
    ? raw: RawInfo
}

HTTP2Setting = {
    ? name: $HTTP2SettingsName
    ; only when name === "unknown"
    ? name_bytes: uint64

    value: uint64
}

$HTTP2SettingsName /= "settings_header_table_size" /
                   "settings_enable_push" /
                   "settings_max_concurrent_streams" /
                   "settings_initial_window_size" /
                   "settings_max_frame_size" /
                   "settings_max_header_list_size" /
                   "settings_extended_connect" /
                   "settings_no_rfc7540_priorities" /
                   "unknown"
~~~
{: #settingsframe-def title="HTTP2SettingsFrame definition"}

### HTTP2PushPromiseFrame

~~~ cddl
HTTP2PushPromiseFrame = {
    frame_type: "push_promise"
    push_id: uint64
    headers: [* HTTP2HTTPField]
    ? raw: RawInfo
}
~~~
{: #pushpromiseframe-def title="HTTP2PushPromiseFrame definition"}

### HTTP2GoAwayFrame

~~~ cddl
HTTP2GoawayFrame = {
    frame_type: "goaway"

    stream_id: uint32
    error_code: uint32
    ? additional_debug_data: hexstring
    ? raw: RawInfo
}
~~~
{: #goawayframe-def title="HTTP2GoawayFrame definition"}

### HTTP2UnknownFrame

The frame_type_bytes field is the numerical value without variable-length
integer encoding.

~~~ cddl
HTTP2UnknownFrame = {
    frame_type: "unknown"
    frame_type_bytes: uint32
    ? raw: RawInfo
}
~~~
{: #unknownframe-def title="HTTP2UnknownFrame definition"}

# Security Considerations

The security and privacy considerations discussed in {{Section 14 of QLOG-MAIN}}
apply to this document as well.

# IANA Considerations

This document will have IANA actions if adopted.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
