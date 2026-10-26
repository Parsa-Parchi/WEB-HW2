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

## Question 3: Delivering server events to clients

### 3.1 What challenge does the HTTP request-response model create?

In ordinary HTTP, the client initiates a request and the server sends a response. A server cannot independently open a new ordinary HTTP exchange to a browser that has not requested it. This makes timely delivery of new chat messages and alerts difficult: an event can occur while the browser has no outstanding request waiting for it. Repeated requests can detect events, but introduce a trade-off between notification delay and unnecessary traffic. This limitation concerns how exchanges are initiated, rather than HTTP's statelessness; a response can also remain open and carry data over time.

### 3.2 What solutions address this challenge?

**Periodic polling:** The client sends requests at regular intervals, such as asking for new messages every few seconds. Each request receives an immediate response, whether or not anything changed. This is simple and works with ordinary HTTP infrastructure, but frequent polling wastes requests when there are no events, while longer intervals increase notification delay. It suits updates that do not need immediate delivery.

**Long polling:** The client sends a request that the server holds open until an event arrives or a timeout occurs. After receiving the response, the client immediately sends another request. This reduces empty responses and delivers events sooner than infrequent polling, but requires managing pending requests, timeouts, and reconnects. Each delivered response still needs a new request to continue receiving events, making it less efficient for very frequent messages.

**Server-Sent Events (SSE):** The browser opens an HTTP request, commonly through `EventSource`, and the server keeps its `text/event-stream` response open to deliver events as they occur. SSE provides one-way server-to-client updates, with browser support for reconnection and event IDs that can help resume delivery when the server supports replay. Client-to-server actions use separate requests. It is well suited to alerts, activity feeds, or receiving chat updates, although intermediaries must avoid buffering the stream and allow suitable timeouts.

**WebSocket:** The client establishes a connection through a handshake, traditionally an HTTP/1.1 upgrade. After establishment, both endpoints can send messages independently over a persistent, full-duplex channel without a new HTTP request for each message. This suits interactive chat with frequent traffic in both directions. WebSocket is a separate protocol rather than an ordinary HTTP response stream; applications must manage authentication, connection health, reconnection, and any required message replay.

**Webhooks:** A receiving application registers a reachable callback URL with an event provider. When an event occurs, the provider sends a new HTTP request, usually `POST`, to that URL. This preserves HTTP's request-response model: the event provider acts as the HTTP client, while the callback endpoint acts as the server. Webhooks suit server-to-server notifications, such as payment alerts or repository events; a normal browser tab is not a publicly reachable callback server. A backend can receive a webhook and then notify browsers through SSE or WebSocket. Receivers should authenticate deliveries and handle possible retries and duplicates.

## Question 4: Identifying URL components

### 4.1 What are the components of the given URL?

```text
ss://asghar:1234!!@ss.myproxy.com:1234#shadowSocks1
```

Using the generic URI structure `scheme://[userinfo@]host[:port]/path?query#fragment`:

| Component | Value | Explanation |
| --- | --- | --- |
| Scheme | `ss` | Identifies the scheme associated with Shadowsocks links; this is not an HTTP URL. |
| Authority | `asghar:1234!!@ss.myproxy.com:1234` | Contains the user information, host, and port. |
| User information | `asghar:1234!!` | The part before `@`. Under the conventional `username:password` interpretation, the username is `asghar` and the password is `1234!!`. |
| Host | `ss.myproxy.com` | The complete server hostname; `myproxy.com` is the domain and `ss` is its subdomain. |
| Port | `1234` | The explicit server port after the hostname's colon. |
| Path | Empty | No path appears after the authority. |
| Query | Absent | There is no `?` component. |
| Fragment | `shadowSocks1` | The value after `#`, excluding the delimiter itself; a client can use it as a display label. |

### 4.2 How do the delimiters separate these parts?

The colon after `ss` ends the scheme, and `//` introduces the authority. The `@` separates user information from the host. The colon after the hostname introduces the port, while `#` starts the fragment. The two exclamation marks belong to the user-information value; they do not separate URL components. The fragment is interpreted by the client rather than forming part of the server address.

This is a structural decomposition of the supplied example. The generic URI syntax treats user information as one component and does not itself require a username/password meaning. Actual Shadowsocks share-link formats have scheme-specific rules, commonly involving an encryption method and password, so this decomposition alone does not establish that the example is a valid working Shadowsocks configuration.

## Question 5: Choosing HTTP status codes for REST API scenarios

### 5.1 The database is unavailable and login requests cannot be processed

**`503 Service Unavailable`** is appropriate if the database outage is temporary and prevents the login service from operating. The failure is on the service side, rather than evidence of invalid credentials. A `Retry-After` header can indicate when a retry is appropriate if that time is known. **`500 Internal Server Error`** is another reasonable choice for an unexpected internal database failure when the application cannot characterize it as temporary unavailability. I would choose `503` for a known temporary outage.

### 5.2 The supplied username does not exist and login fails

**`401 Unauthorized`** is appropriate when login fails because valid authentication credentials have not been supplied. Return a generic message such as "Invalid username or password" rather than revealing whether the account exists. A `401` response must include a `WWW-Authenticate` challenge appropriate to the API's authentication scheme. **`400 Bad Request`** is an alternative for an API that explicitly models rejected credentials as invalid input to a custom login operation; under that contract, the status and message should be consistent for both unknown usernames and incorrect passwords. I would use `401` when the API follows HTTP authentication semantics, rather than `404`, since the missing entity is an account supplied in the login data, not the login endpoint itself.

### 5.3 The website has permanently moved to another domain

**`308 Permanent Redirect`**, with a `Location` header containing the new URL, is a suitable choice for a REST API because it preserves the request method and body during automatic redirection. **`301 Moved Permanently`** also expresses a permanent move and is common for ordinary website pages, but clients may change a redirected `POST` into a `GET`. I would choose `308` for API routes where preserving the method matters, and `301` can be suitable for normal `GET` pages. The redirect should point to the corresponding resource on the new domain.

### 5.4 The user has exceeded the allowed request rate

**`429 Too Many Requests`** specifically indicates that the client has sent too many requests within the applicable rate-limit period. The response should explain the limit and can include `Retry-After` to indicate how long to wait before retrying. This is more precise than a generic `503`, because the stated problem is this user's request rate rather than general service unavailability.

### 5.5 JWT renewal succeeds and the renewed token is returned in a header

**`204 No Content`** is suitable if the renewed token is returned only in a response header and there is no response body. It indicates successful processing; headers can still carry response metadata. **`200 OK`** is also reasonable, especially when the endpoint returns a JSON body containing token information. I would choose `204` under the stated header-only assumption and use the API's documented header for the token. A `204` response must not include a response body.

### 5.6 The user's IP range is blocked from accessing the service

**`403 Forbidden`** is appropriate when an access policy denies requests from the user's IP range, even if the request is otherwise valid. Authenticating again does not by itself remove this restriction, so `401` is not the best fit. **`451 Unavailable For Legal Reasons`** would be appropriate only if a legal restriction is actually the reason for blocking access; a geographic or IP-based block alone does not establish that. I would use `403` for the stated IP-range policy without additional evidence of a legal requirement.

## Question 6: Identifying proxy types

### 6.1 University servers used to obtain packages instead of contacting central package servers directly

**Type: Forward proxy, specifically a caching forward proxy if downloaded packages are cached.** It acts on behalf of university clients: their package requests go through the university intermediary to external package servers. Cached packages can be reused for later requests, reducing external bandwidth use and download time. The defining feature is that the proxy represents the requesting clients when accessing outside servers.

If the university server instead hosts a synchronized copy of a package repository and students configure that repository as their package source, it is more precisely a **repository mirror**, rather than necessarily a forward proxy. Under the question's proxy interpretation, the intended classification is a caching forward proxy.

### 6.2 Distributing requests according to their geographic origin

**Type: Reverse proxy performing geographic routing or load balancing.** It sits in front of the service's backend servers, accepts incoming client requests, and selects a backend or region based on the client's geographic origin. This can reduce latency and direct traffic to suitable regional infrastructure. It represents the service's servers rather than the clients, which makes it a reverse proxy. If the routing decision is made only through geographic DNS responses, that is DNS-based routing rather than a proxy itself; the reverse-proxy classification assumes the intermediary actually receives and forwards the requests.

### 6.3 Adding authentication to a very old service

**Type: Authenticating reverse proxy.** It is placed in front of the legacy service and checks authentication before forwarding allowed requests to it. This adds an access-control layer without requiring authentication support to be implemented inside the old application. The legacy service should accept traffic only through the trusted proxy so users cannot bypass the authentication check by contacting it directly. Any identity headers forwarded to the backend must be supplied by the trusted proxy rather than accepted unchecked from clients.
