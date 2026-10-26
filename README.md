# Web Programming - Homework 2: HTTP

## Question 1: HTTP versions and the use of UDP in HTTP/3

### 1.1 What are the key differences between HTTP versions?

| Version | Key features and differences |
| --- | --- |
| **HTTP/0.9** | Minimal, text-based protocol; only `GET`; response body only, without response headers or status codes; mainly HTML retrieval. |
| **HTTP/1.0** | Request and response headers; status codes; additional methods such as `HEAD` and `POST`; content-type metadata for different representations; normally a separate TCP connection for each exchange. |
| **HTTP/1.1** | Persistent TCP connections by default; mandatory `Host` header for requests; chunked transfer coding; improved caching; pipelining support, but pipelined responses must remain in request order. |
| **HTTP/2** | Binary framing; multiple concurrent streams multiplexed over one TCP connection; HPACK header compression; no requirement for different streams' responses to finish in request order; TCP-level head-of-line blocking still possible. |
| **HTTP/3** | HTTP over QUIC, which uses UDP; independent reliable streams; QPACK header compression; TLS 1.3 integrated into QUIC; reduced connection setup latency and support for connection migration. |

In HTTP/1.1, a slow response can delay later pipelined responses on the same connection. HTTP/2 multiplexes streams to address that ordering limitation, but TCP still delivers one ordered byte stream: a lost TCP segment can delay delivery of data belonging to several HTTP/2 streams. HTTP/3 moves stream handling into QUIC to address this transport-level limitation.

### 1.2 Why does HTTP/3 use UDP?

Strictly speaking, HTTP/3 runs over **QUIC**, and QUIC runs over **UDP**. It does not send ordinary HTTP requests as unreliable UDP messages. UDP provides the datagram substrate, while QUIC implements reliable delivery, loss recovery, congestion control, and ordered delivery within each stream.

The main reasons for this design are:

- **Independent stream delivery:** Losing data for one QUIC stream does not impose TCP's connection-wide in-order delivery barrier on other streams. Their available data can continue to be delivered. Loss can still affect shared congestion control, and application-level dependencies can still cause blocking.
- **Faster connection establishment:** QUIC combines transport setup with the TLS 1.3 handshake. A new connection typically needs one round trip before ordinary application data can be sent; eligible resumed connections can use 0-RTT data, subject to replay-safety restrictions.
- **Connection migration:** QUIC connection IDs can allow a connection to survive an IP address or port change, such as switching from Wi-Fi to a mobile network, subject to path validation.
- **Easier transport evolution:** Implementing QUIC over widely supported UDP allows transport improvements in applications without requiring changes to TCP implementations in operating systems. UDP is not guaranteed to be allowed on every network, so clients may need to fall back to TCP-based HTTP versions.

Using UDP alone would not provide these features. The benefit comes from QUIC's design on top of UDP, rather than from removing reliability or security.

## Question 2: Stateful and stateless communication

### 2.1 What is the difference between stateful and stateless communication?

In **stateful** communication, the server retains conversational or session context between requests, and later requests can depend on that context. In **stateless** communication, each request can be understood independently using the information it carries, without relying on earlier requests in the conversation. Statelessness does not mean that the server cannot store resources or use a database; it concerns the interaction's dependence on remembered client context.

### 2.2 Which category does HTTP belong to, and how do web applications manage state?

HTTP is **stateless**: the protocol does not automatically remember that separate requests belong to a previously authenticated user or shopping session. Applications that need this continuity add state-management mechanisms, commonly **cookies and server-side sessions**.

After a successful login, the application can create a session record on the server and send its unpredictable identifier in a `Set-Cookie` response header. The browser stores the cookie and includes it in later matching requests through the `Cookie` header. The server uses that identifier to look up the session and recognize the user, allowing features such as remaining logged in or maintaining a shopping cart. The cookie can hold just the identifier rather than the full session data.

Another approach is for the client to send an authentication token with each request, commonly in the `Authorization` header. A self-contained signed token can carry claims the server verifies without looking up a traditional session record, although applications may still keep state for revocation or other purposes. Both approaches supply application-level context explicitly; neither changes HTTP's stateless semantics. Reusing a persistent TCP or QUIC connection also does not, by itself, make HTTP stateful.
