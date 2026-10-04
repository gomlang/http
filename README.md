# HTTP protocol utilities

`ecosystem::http` contains pure, transport-independent HTTP helpers shared by
ecosystem clients and servers. It does not open sockets or parse live streams.

`strip_hop_headers` validates a sequence of name/value pairs, removes standard
hop-by-hop fields and every field nominated by a `Connection` header, lowercases
retained names and preserves duplicate value order. An invalid header or
`Connection` token is a recoverable error. Reverse proxies should apply it on
both sides of the exchange and set trusted forwarding headers only afterward.
Connection nominations use a private sorted index, so each retained field does
not scan the entire token list. Validation still follows input order, including
fields that will be removed.

`content_length(headers, limit)` validates header syntax and returns the optional
unsigned declared length within a caller-supplied byte limit. Repeated fields
and comma lists are accepted only when every decimal value agrees; leading
zeroes are allowed. Empty members, signs, non-ASCII whitespace, overflow,
conflicting lengths and simultaneous Transfer-Encoding are errors. This follows
[RFC 9110 section 8.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.6)
and the strict framing policy in
[RFC 9112 section 6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.3).
It does not decide whether a status/method permits a body, validate transfer
codings, or enforce that the body actually contains the declared bytes.
Connection token lists accept only ASCII space and tab as surrounding whitespace.
Empty list members, including an entirely empty Connection field, are ignored
as required by [RFC 9110 sections 5.6.1.2 and 7.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.6.1.2).
Nonempty members must still be valid tokens; all nominated fields are removed.

`response_framing(method, status, headers, limit)` determines HTTP/1.1 response
framing as `BodyFraming::{Empty, Tunnel, Fixed(u64), Chunked, UntilClose}`.
It follows the ordered rules of [RFC 9112 section 6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.3):
HEAD and 1xx/204/304 take precedence, followed by successful CONNECT. Those cases
ignore length/coding semantics and limits after header syntax validation; notably
CONNECT 204 follows the no-body rule. Methods are case-sensitive. Other responses
reject simultaneous TE/CL and inconsistent or excessive lengths. Transfer coding
names are case-insensitive tokens; chunked must occur once and last. A non-chunked
final coding or absent framing means EOF termination. Empty list members are
ignored; an entirely empty TE list and coding parameters are unsupported errors.
The helper does not decode transfer codings, cap bytes actually read, handle 101
protocol switching, or make connection-reuse decisions.

`request_framing(headers, limit)` returns `Empty` when neither length nor transfer
coding is present, `Fixed(n)` for an agreed Content-Length (including zero), or
`Chunked` when the final transfer coding is chunked. Request framing is independent
of the method. Requests never use EOF termination: a transfer coding list ending
in another coding is rejected. Header syntax, TE/CL ambiguity, length limits and
transfer coding syntax use the same strict policy as `response_framing`, including
rejection of coding parameters. These rules follow
[RFC 9112 section 6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.3).
The helper classifies framing only; callers decode supported codings, enforce
stream limits and handle connection closure after invalid framing.

`dump_request` and `dump_response` construct bounded HTTP/1.1-style byte views
from explicit start-line parts, headers and a copied binary body. They reject
start-line/header injection and return a limit error rather than truncate. They
append headers to a single byte buffer, without repeatedly copying the preceding
headers as the field count grows. Validation and limit checks follow wire order.
They are diagnostics, not exact round trips: parsed requests may have lost original
header case/order, and HTTP/2 is represented as HTTP/1.1 text. Callers provide
their own headers and framing fields; these functions do not add or reconcile
`Content-Length`, chunked encoding, trailers or connection state.

`chunk_size(line, limit)` parses a chunk-size line without CRLF and returns its
hexadecimal size within a caller-supplied `u64` limit. It validates every chunk
extension, including token names, token or quoted values, quoted-pair escapes
and the space/tab positions allowed by [RFC 9112 section 7.1.1](https://www.rfc-editor.org/rfc/rfc9112.html#section-7.1.1).
Extension values are ignored after validation. Callers separately bound the
line length, read the payload and CRLF, and parse trailers after a zero chunk.

```gom
use ecosystem::http::{Error, dump_request, strip_hop_headers};
use std::bytes::{Bytes};

fn debug_request() -> Result[Bytes, Error] {
    let headers = strip_hop_headers(
        Vec::from_array([("Connection", "x-private"), ("X-Private", "secret")]),
    )?;
    dump_request("GET", "/", headers, Bytes::new(), 1024)
}
```

## Development and examples

Requires GoML 0.1.56 or newer. The `examples/basic/` example shares the root manifest and its dependencies. From the library root, run:

```sh
goml run --example basic
goml test
goml verify --timeout 300s
```

`goml test` builds the example and runs its tests. `goml verify` repeats the example checks as an independent module against an isolated registry snapshot. `(cd ../verification && just ecosystem-test http)` also retains the library-specific smoke and compatibility checks.
