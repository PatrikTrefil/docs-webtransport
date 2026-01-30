---
title: WebTransport support in .NET
description: Learn about the support for WebTransport protocol in .NET.
ms.date: 09/10/2025
helpviewer_keywords:
    - "protocols, WebTransport"
    - "sending data, WebTransport"
    - "WebTransport"
    - "receiving data, WebTransport"
    - "application protocols, WebTransport"
    - "Internet, WebTransport"
---

# WebTransport protocol

WebTransport protocol enables communication with a remote server using a secure multiplexed transport.
Currently, we support WebTransport over HTTP/3.
The <xref:System.Net.WebTransport.ClientWebTransportSession?displayProperty=fullName> exposes the ability to establish a WebTransport session over HTTP/3. The session can be established using the `ConnectAsync` method.

## Platform dependencies

WebTransport over HTTP/3 requires support for the transport protocol QUIC. For information on the platform requirements of QUIC in .NET, see [QUIC Platform dependencies](../quic/quic-overview.md#platform-dependencies).

## API overview

<xref:System.Net.WebTransport> brings three major classes that enable the usage of the WebTransport protocol:

-   <xref:System.Net.WebTransport.ClientWebTransportSession> - client side class for establishing WebTransport session over HTTP/3, corresponding to [RFC XXXX Section 3](https://datatracker.ietf.org/doc/html/draft-ietf-webtrans-http3-12#name-session-establishment).
-   <xref:System.Net.WebTransport.WebTransportSession> - WebTransport session, corresponding to [RFC XXXX Section 4.1](https://datatracker.ietf.org/doc/html/draft-ietf-webtrans-overview-10#name-session-wide-features).
-   <xref:System.Net.WebTransport.WebTransportStream> - WebTransport stream, corresponding to [RFC 9000 Section 4.3](https://datatracker.ietf.org/doc/html/draft-ietf-webtrans-overview-10#name-streams).

To use the WebTransport protocol in a server-scenario, see [WebTransport support in ASP.NET Core]().

### `WebTransportSession`

<xref:System.Net.WebTransport.WebTransportSession> represents a WebTransport session. Client side sessions are created using a static method <xref:System.Net.WebTransport.ClientWebTransportSesssion.ConnectAsync(System.Uri,System.Net.Http.HttpMessageInvoker,System.Net.WebTransport.WebTransportSessionCreationOptions,System.Threading.CancellationToken)> that establishes the session. The session creation is configured using the <xref:System.Net.WebTransport.WebTransportSessionCreationOptions>. You always need to provide a <xref:System.Uri> of the endpoint you want to establish the session with, an instance of <xref:System.Net.HttpVersion> that describes the version of the HTTP protocol to use (currently only supported version is <xref:System.Net.HttpVersion.Version30>) and a <xref:System.Net.Http.HttpMessageInvoker> instance that is used for the initial handshake. The <xref::System.Net.Http.HttpMessageInvoker> instance must support HTTP/3. Note, that it may be necessary for you to setup a keep-alive mechanism for the <xref::System.Net.Http.HttpMessageInvoker> to prevent an idle timeout of the underlying QUIC connection. If you are using <xref:System.Net.Http.SocketsHttpHandler>, you can use <xref:System.Net.Http.SocketsHttpHandler.KeepAlivePingDelay> and <xref:System.Net.Http.SocketsHttpHandler.KeepAlivePingTimeout>. The details of the configuration depend on your specific requirements and the configuration of the underlying QUIC connection. It is also possible to set various limits for the WebTransport session such as a limit for the number of bytes sent over the session (<xref:System.Net.WebTransport.ClientWebTransportSessionCreationOptions.InitialDataSentLimitForPeer>). If you do not set these limits, their default values will be used.

Once the session is established, it can be used to open and accept unidirectional/bidirectional streams using <xref:System.Net.WebTransport.WebTransportSession.OpenOutboundStreamAsync(System.Net.WebTransport.WebTransportStreamType,System.Threading.CancellationToken)> and <xref:System.Net.WebTransport.WebTransportSession.AcceptInboundStreamAsync(System.Threading.CancellationToken)>.
You can change the settings of the session during its lifetime using the `WebTransportSession.Set*LimitForPeerAsync()` methods.
The values of `WebTransportSession.*LimitProvidedByPeer` are updated automatically when the peer changes its limits.

When the work with the session is done, it needs to be closed and disposed. Users can only close the session gracefully. If you want to provide a status code and status description, you shall use <xref:System.Net.WebTransport.WebTransportSession.CloseAsync(System.Int64,System.String,System.Threading.CancellationToken)>. Note that the delivery of the status code and the status description is best-effort. If you do not want to provide any details, you can use <xref:System.Net.WebTransport.WebTransportSession.CloseAsync()>. You can also request the peer to close the session using <xref:System.Net.WebTransport.WebTransportSession.RequestCloseAsync(System.Threading.CancellationToken)> instead of closing it yourself. In some cases, the session may be closed abortively automatically. That happens for example if the peer violates the protocol or there is a network error. Finally, <xref:System.Net.WebTransport.WebTransportSession.DisposeAsync> must be called at the end of the work with the session to fully release all the associated resources.

Consider the following example code:

```csharp
using System.Net.WebTransport;

var sessionCreationOptions = new WebTransportSessionCreationOptions
{
    TargetUri = new Uri("https://example.com"),
    // Optional limits, if not set default values will be used.
    InitialUnidirectionalStreamCountLimitForPeer = 10,
    InitialBidirectionalStreamCountLimitForPeer = 100,
    HttpVersion = HttpVersion.Version30,
    HttpVersionPolicy = HttpVersionPolicy.RequestVersionExact
};

await using WebTransportSession session = await WebTransportSession.ConnectAsync(sessionCreationOptions);

// Update the unidirectional stream limit for the peer.
await session.SetUnidirectionalStreamCountLimitForPeerAsync(20);

// Open/accept streams.
await using (outgoingStream = await session.OpenOutboundStreamAsync(WebTransportStreamType.Bidirectional))
await using (incomingStream = await session.AcceptInboundStreamAsync(WebTransportStreamType.Unidirectional)
{
    // Work with the streams...
}

// Close the connection with a custom code.
await session.CloseAsync(42, "Status description");

// DisposeAsync will be called by await using at the top.
```

### `WebTransportStream`

<xref:System.Net.WebTransport.WebTransportStream> is the actual type that is used to send and receive data in the WebTransport protocol. It derives from ordinary <xref:System.IO.Stream> and can be used as such, but it also offers several features that are specific to the WebTransport protocol. Firstly, a WebTransport stream can either be unidirectional or bidirectional, see [RFC XXXX Section 4.3](https://datatracker.ietf.org/doc/html/draft-ietf-webtrans-overview-10#name-streams). A bidirectional stream is able to send and receive data on both sides, whereas unidirectional stream can only write from the initiating side and read on the accepting one. 

Another particularity of a WebTransport stream is the ability to explicitly gracefully close the writing side in the middle of work with the stream, see <xref:System.Net.WebTransport.WebTransportStream.CompleteWrites> or <xref:System.Net.WebTransport.WebTransportStream.WriteAsync(System.ReadOnlyMemory{System.Byte},System.Boolean,System.Threading.CancellationToken)> overload with `completeWrites` argument. Closing of the writing side lets the peer know that no more data will arrive, yet the peer still can continue sending (in case of a bidirectional stream). And for erroneous cases, either writing or reading side of the stream can be aborted, see <xref:System.Net.WebTransport.WebTransportStream.Abort(System.Net.WebTransport.WebTransportAbortDirection,System.Int64)>. When a read operation is cancelled using a <xref:System.Threading.CancellationToken>, the reading side of the stream is aborted. When a write operation is cancelled using a <xref:System.Threading.CancellationToken>, the writing side of the stream is aborted.

The behavior of the individual methods for each stream type is summarized in the following table (note that both client and server can open and accept streams):

| Method                            | Peer opening stream                                                                                                                                                                | Peer accepting stream                                                                                                                                                             |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CanRead`                         | _bidirectional_: `true`<br/> _unidirectional_: `false`                                                                                                                             | `true`                                                                                                                                                                            |
| `CanWrite`                        | `true`                                                                                                                                                                             | _bidirectional_: `true`<br/> _unidirectional_: `false`                                                                                                                            |
| `ReadAsync`                       | _bidirectional_: reads data<br/> _unidirectional_: `InvalidOperationException`                                                                                                     | reads data                                                                                                                                                                        |
| `WriteAsync`                      | sends data => peer read returns the data                                                                                                                                           | _bidirectional_: sends data => peer read returns the data<br/> _unidirectional_: `InvalidOperationException`                                                                      |
| `CompleteWrites`                  | closes writing side => peer read returns 0                                                                                                                                         | _bidirectional_: closes writing side => peer read returns 0<br/> _unidirectional_: no-op                                                                                          |
| `Abort(WebTransportAbortDirection.Read)`  | _bidirectional_: peer write throws `WebTransportException(WebTransportError.OperationAborted)`<br/> _unidirectional_: no-op | peer write throws `WebTransportException(WebTransportError.OperationAborted)`                                              |
| `Abort(WebTransportAbortDirection.Write)` | peer read throws `WebTransportException(WebTransportError.OperationAborted)`                                                | _bidirectional_: peer read throws `WebTransportException(WebTransportError.OperationAborted)`<br/> _unidirectional_: no-op |

On top of these methods, `WebTransportStream` offers two specialized properties to get notified whenever either reading or writing side of the stream has been closed: <xref:System.Net.WebTransport.WebTransportStream.ReadsClosed> and <xref:System.Net.WebTransport.WebTransportStream.WritesClosed>. Both return a `Task` that completes with its corresponding side getting closed, whether it be success or abort, in which case the `Task` will contain appropriate exception. These properties are useful when the user code needs to know about stream side getting closed without issuing call to `ReadAsync` or `WriteAsync`.

Finally, when the work with the stream is done, it needs to be disposed with <xref:System.Net.WebTransport.WebTransportStream.DisposeAsync>. The dispose will make sure that both reading and/or writing side - depending on the stream type - is closed. If stream hasn't been properly read till the end, dispose will issue an equivalent of `Abort(WebTransportAbortDirection.Read)`. However, if stream writing side hasn't been closed, it will be gracefully closed as it would be with `CompleteWrites`. The reason for this difference is to make sure that scenarios working with an ordinary `Stream` behave as expected and lead to a successful path. Consider the following example:

```csharp
// Work done with all different types of streams.
async Task WorkWithStreamAsync(Stream stream)
{
    // This will dispose the stream at the end of the scope.
    await using (stream)
    {
        // Simple echo, read data and send them back.
        byte[] buffer = new byte[1024];
        int count = 0;
        // The loop stops when read returns 0 bytes as is common for all streams.
        while ((count = await stream.ReadAsync(buffer)) > 0)
        {
            await stream.WriteAsync(buffer.AsMemory(0, count));
        }
    }
}

// Open a WebTransportStream and pass to the common method.
var webtransportStream = await session.OpenOutboundStreamAsync(WebTransportStreamType.Bidirectional);
await WorkWithStreamAsync(webtransportStream);
```

The sample usage of `WebTransportStream` in client scenario:

```csharp
// Consider session from the session example, open a bidirectional stream.
await using var stream = await session.OpenOutboundStreamAsync(WebTransportStreamType.Bidirectional, cancellationToken);

// Send some data.
await stream.WriteAsync(data, cancellationToken);
await stream.WriteAsync(data, cancellationToken);

// End the writing-side together with the last data.
await stream.WriteAsync(data, completeWrites: true, cancellationToken);
// Or separately.
stream.CompleteWrites();

// Read data until the end of stream.
while (await stream.ReadAsync(buffer, cancellationToken) > 0)
{
    // Handle buffer data...
}

// DisposeAsync called by await using at the top.
```

## Session closing handshake



## See also

-   [Networking in .NET](../overview.md)
-   [HTTP/3 with HttpClient](../../../core/extensions/httpclient-http3.md)
-   [WebTransport support in ASP.NET Core]()
-   <xref:System.Net.WebTransport>
-   <xref:System.Net.WebTransport.WebTransportSesssion>
-   <xref:System.Net.WebTransport.ClientWebTransportSesssion>
-   <xref:System.Net.WebTransport.WebTransportStream>
-   <xref:System.Net.Quic>
