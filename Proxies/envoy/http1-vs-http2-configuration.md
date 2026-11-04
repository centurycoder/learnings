# Envoy: which configuration selects HTTP/2?

Based on [Life of a Request — Envoy v1.40.2](https://www.envoyproxy.io/docs/envoy/v1.40.2/intro/life_of_a_request).

**The example explicitly selects HTTP/2 upstream, but does not force HTTP/2 downstream.** The client-to-Envoy and Envoy-to-backend connections have independent HTTP protocol choices.

## Downstream: client → Envoy

The HTTP connection manager contains:

```yaml
http2_protocol_options:
  max_concurrent_streams: 100
```

This configures the HTTP/2 codec’s settings **when HTTP/2 is used**. It does not mean “accept only HTTP/2.”

The downstream protocol-selection field is `codec_type`, inside the HTTP connection manager’s `typed_config`. The example omits it, so the effective default is:

```yaml
codec_type: AUTO
```

With `AUTO`, Envoy prefers TLS ALPN when available; otherwise it infers the protocol from incoming bytes.

To require an HTTP/2 codec instead, set:

```yaml
# Inside the HTTP connection manager's typed_config:
codec_type: HTTP2
```

This assumes the client speaks HTTP/2. It does not upgrade an HTTP/1.1 client. `codec_type: HTTP1` instead assumes HTTP/1.1.

### TLS ALPN: an omission in the example

For ordinary HTTPS clients to negotiate HTTP/2, advertise the supported application protocols in the selected filter chain’s **downstream TLS transport configuration**:

```yaml
# Inside transport_socket.typed_config.common_tls_context:
alpn_protocols: ["h2", "http/1.1"]
```

This advertises both HTTP/2 (`h2`) and HTTP/1.1. Together with `codec_type: AUTO`, it allows either version to be negotiated.

The documentation’s example omits `alpn_protocols`. This field has no default protocol list: when empty, Envoy does not advertise ALPN. Therefore, the page’s downstream HTTP/2 request is an **assumed scenario**, not behavior guaranteed for a normal HTTPS client by the shown configuration. Envoy can still process HTTP/2 without ALPN if the client sends HTTP/2 using prior knowledge.

## Upstream: Envoy → backend

Cluster `some_service` contains:

```yaml
typed_extension_protocol_options:
  envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
    "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
    explicit_http_config:
      http2_protocol_options:
        max_concurrent_streams: 100
```

Here, **`explicit_http_config.http2_protocol_options` selects HTTP/2 for upstream connections**, as well as configuring its settings. There is no automatic HTTP/1.1 fallback in this mode.

The same field name has different significance depending on its location:

- HCM `http2_protocol_options`: configure HTTP/2 settings; `codec_type` controls protocol selection.
- Upstream `explicit_http_config.http2_protocol_options`: explicitly select HTTP/2 and configure its settings.

The backend must support HTTP/2 for this upstream configuration to work. The downstream version need not match: Envoy can accept HTTP/1.1 from a client and forward the request using HTTP/2 upstream.

## References

- [HTTP connection manager: CodecType](https://www.envoyproxy.io/docs/envoy/v1.40.2/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto.html#envoy-v3-api-enum-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-codectype)
- [TLS CommonTlsContext: alpn_protocols](https://www.envoyproxy.io/docs/envoy/v1.40.2/api-v3/extensions/transport_sockets/tls/v3/tls.proto.html#envoy-v3-api-field-extensions-transport-sockets-tls-v3-commontlscontext-alpn-protocols)
- [Upstream ExplicitHttpConfig](https://www.envoyproxy.io/docs/envoy/v1.40.2/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto.html#envoy-v3-api-msg-extensions-upstreams-http-v3-httpprotocoloptions-explicithttpconfig)
