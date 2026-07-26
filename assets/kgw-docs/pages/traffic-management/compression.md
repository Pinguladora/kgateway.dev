Reduce response payload sizes and decompress incoming request bodies with gzip, Brotli, or Zstandard compression.

## About compression

Compression is an HTTP option that enables your gateway proxy to compress response data or decompress request data on behalf of clients. Compression is useful when large payloads need to be transmitted without compromising response time.

Use the {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}} resource to configure compression and decompression per route. Choose between the following options:

- **Response compression**: When enabled on a route, {{< reuse "kgw-docs/snippets/kgateway.md" >}} compresses HTTP responses when the downstream client advertises support with an `Accept-Encoding` header. You select which codecs to offer with the `libraries` field (`Gzip`, `Brotli`, or `Zstd`), and Envoy negotiates the codec to use from the client's `Accept-Encoding` header. When you omit `libraries`, {{< reuse "kgw-docs/snippets/kgateway.md" >}} uses gzip, so existing configuration keeps working unchanged. The following content types are compressed by default:
  - `application/javascript`
  - `application/json`
  - `application/xhtml+xml`
  - `image/svg+xml`
  - `text/css`
  - `text/html`
  - `text/plain`
  - `text/xml`

- **Request decompression**: When enabled on a route, {{< reuse "kgw-docs/snippets/kgateway.md" >}} decompresses request bodies before forwarding them to the backend service. As with response compression, you choose the codecs with the `libraries` field, and Envoy selects the decompressor from the request's `Content-Encoding` header. This lets the gateway normalize request bodies so backends do not each need to support every codec.

The client chooses the codec in both directions, so by default the order of `libraries` is not a preference that the gateway enforces. Envoy uses the highest quality codec the client accepts based on the `Accept-Encoding` weights, and when several are offered with equal quality the client's own ordering decides. If the client accepts none of the offered codecs, the response is sent uncompressed. To have the gateway prefer certain codecs instead, see [Set a preferred codec order](#compression-preferred-order).

For more information about how Envoy handles compression, see the [Envoy compressor filter docs](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/compressor_filter).

## Before you begin

{{< reuse "kgw-docs/snippets/prereq.md" >}}

## Response compression {#response-compression}

Enable compression on a route so that {{< reuse "kgw-docs/snippets/kgateway.md" >}} compresses responses for clients that support it.

1. Create an HTTPRoute that routes requests from the `compression.example` domain to the httpbin sample app.
   ```yaml
   kubectl apply -f- <<EOF
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
     name: httpbin-compression
     namespace: httpbin
   spec:
     parentRefs:
     - name: http
       namespace: {{< reuse "kgw-docs/snippets/namespace.md" >}}
     hostnames:
       - compression.example
     rules:
       - backendRefs:
           - name: httpbin
             port: 8000
   EOF
   ```

2. Create a {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}} that enables response compression on the route. This example offers all three codecs. To keep the previous gzip-only behavior, set `responseCompression: {}` or omit the `libraries` field.
   ```yaml
   kubectl apply -f- <<EOF
   apiVersion: {{< reuse "kgw-docs/snippets/trafficpolicy-apiversion.md" >}}
   kind: {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}}
   metadata:
     name: response-compression
     namespace: httpbin
   spec:
     targetRefs:
     - group: gateway.networking.k8s.io
       kind: HTTPRoute
       name: httpbin-compression
     compression:
       responseCompression:
         libraries:
         - Gzip
         - Brotli
         - Zstd
   EOF
   ```

   | Setting | Description |
   |--|--|
   | `spec.targetRefs` | The HTTPRoute to apply this policy to. |
   | `spec.compression.responseCompression` | Enables response compression. Responses are compressed only when the client sends an `Accept-Encoding` header that matches an offered codec and the response content type is compressible. |
   | `spec.compression.responseCompression.libraries` | The codecs to offer, from `Gzip`, `Brotli`, and `Zstd`. Envoy picks the codec from the client's `Accept-Encoding` header. Defaults to `[Gzip]` when unset. By default, order is not a preference that the gateway enforces. |

3. Send a request to the `/html` httpbin path with an `Accept-Encoding` header that lists the codecs the client accepts. Verify that the response includes a `content-encoding` header set to one of the offered codecs, indicating that the response body is compressed.
   {{< tabs >}}
   {{% tab name="Cloud Provider LoadBalancer" %}}
   ```sh
   curl -vik http://$INGRESS_GW_ADDRESS:8080/html \
     -H "host: compression.example:8080" \
     -H "Accept-Encoding: br, gzip, zstd"
   ```
   {{% /tab %}}
   {{% tab name="Port-forward for local testing" %}}
   ```sh
   curl -vik http://localhost:8080/html \
     -H "host: compression.example" \
     -H "Accept-Encoding: br, gzip, zstd"
   ```
   {{% /tab %}}
   {{< /tabs >}}

   Example output. The client lists `br` first, so Envoy uses Brotli. Reorder the header (for example `zstd, gzip, br`) to select a different codec:
   ```console {hl_lines=[5]}
   < HTTP/1.1 200 OK
   HTTP/1.1 200 OK
   < content-type: text/html; charset=utf-8
   content-type: text/html; charset=utf-8
   < content-encoding: br
   content-encoding: br
   < transfer-encoding: chunked
   transfer-encoding: chunked
   < server: envoy
   server: envoy
   ```

4. Send the same request without an `Accept-Encoding` header. Verify that the response does **not** include a `content-encoding` header, confirming that compression is only applied when the client requests it.
   {{< tabs >}}
   {{% tab name="Cloud Provider LoadBalancer" %}}
   ```sh
   curl -vik http://$INGRESS_GW_ADDRESS:8080/html \
     -H "host: compression.example:8080"
   ```
   {{% /tab %}}
   {{% tab name="Port-forward for local testing" %}}
   ```sh
   curl -vik http://localhost:8080/html \
     -H "host: compression.example"
   ```
   {{% /tab %}}
   {{< /tabs >}}

   Example output:
   ```console
   < HTTP/1.1 200 OK
   HTTP/1.1 200 OK
   < content-type: text/html; charset=utf-8
   content-type: text/html; charset=utf-8
   < content-length: 3741
   content-length: 3741
   < server: envoy
   server: envoy
   ```

### Limit compression by size and content type {#compression-settings}

By default, {{< reuse "kgw-docs/snippets/kgateway.md" >}} relies on Envoy's defaults for which responses to compress: any response at least 30 bytes long whose content type is in the default list shown in [About compression](#about-compression). You can override both:

- `minContentLengthBytes`: the smallest response `Content-Length`, in bytes, that triggers compression. Responses smaller than this are sent uncompressed. Envoy enforces this only when the response has a `Content-Length` header, so chunked responses are always eligible.
- `contentTypes`: the set of content types to compress. When set, it replaces the default list rather than adding to it.

```yaml
apiVersion: {{< reuse "kgw-docs/snippets/trafficpolicy-apiversion.md" >}}
kind: {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}}
metadata:
  name: response-compression
  namespace: httpbin
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: httpbin-compression
  compression:
    responseCompression:
      libraries:
      - Gzip
      - Brotli
      - Zstd
      minContentLengthBytes: 1024
      contentTypes:
      - application/json
      - text/html
```

| Setting | Description |
|--|--|
| `spec.compression.responseCompression.minContentLengthBytes` | The minimum response size, in bytes, required to trigger compression. Defaults to 30 when unset. |
| `spec.compression.responseCompression.contentTypes` | The content types to compress. Replaces the default set when specified. |

### Handle ETag headers {#compression-etag}

Compressing a response changes its body, so by default Envoy removes a strong `ETag` header when it compresses a response, because the tag no longer matches the bytes the client receives. Removing the `ETag` can defeat downstream caching and conditional requests that rely on it. Two fields on `responseCompression` let you control this behavior:

- `disableOnEtag`: skip compression for any response that carries an `ETag`, so the tag stays intact as a strong validator.
- `weakenEtagOnCompress`: still compress the response, but weaken the `ETag` by prepending `W/` instead of removing it, so caches and conditional requests keep working while signaling that compression changed the body.

Both default to false, which keeps Envoy's default handling. When you set both to true, `weakenEtagOnCompress` takes precedence, so the response is compressed and its `ETag` is weakened.

```yaml
apiVersion: {{< reuse "kgw-docs/snippets/trafficpolicy-apiversion.md" >}}
kind: {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}}
metadata:
  name: response-compression
  namespace: httpbin
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: httpbin-compression
  compression:
    responseCompression:
      libraries:
      - Gzip
      weakenEtagOnCompress: true
```

| Setting | Description |
|--|--|
| `spec.compression.responseCompression.disableOnEtag` | Skips compression for responses that carry an `ETag` header, keeping the tag as a strong validator. Defaults to false. |
| `spec.compression.responseCompression.weakenEtagOnCompress` | Weakens the `ETag` with a `W/` prefix instead of removing it when a response is compressed. Defaults to false. When set together with `disableOnEtag`, this takes precedence. |

### Set a preferred codec order {#compression-preferred-order}

**Note:** This behavior is still under design in [issue #14455](https://github.com/kgateway-dev/kgateway/issues/14455). The configuration surface is not finalized, so the exact field may change.

By default the client chooses the codec, so when a client accepts several codecs with equal weight the client's ordering decides which one Envoy uses. Most browsers send `Accept-Encoding: gzip, deflate, br, zstd` with no weights, so gzip almost always wins even though brotli and zstd usually compress better.

You can instead let the gateway set the order Envoy uses to break those ties, so a stronger codec such as brotli or zstd is chosen instead of gzip. A {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}} still decides which codecs are offered for a route, and the gateway's order only applies to clients that do not express a preference.

<!-- Config example intentionally omitted until the placement (Settings vs HTTPListenerPolicy) is decided in issue #14455. -->

## Request decompression {#request-decompression}

Enable decompression on a route so that {{< reuse "kgw-docs/snippets/kgateway.md" >}} decompresses request bodies before forwarding them to the backend service. This is useful when clients send compressed request bodies, so that backends receive plain text regardless of the codec the client used.

1. Create an HTTPRoute that routes requests from the `decompression.example` domain to the httpbin sample app.
   ```yaml
   kubectl apply -f- <<EOF
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
     name: httpbin-decompression
     namespace: httpbin
   spec:
     parentRefs:
     - name: http
       namespace: {{< reuse "kgw-docs/snippets/namespace.md" >}}
     hostnames:
       - decompression.example
     rules:
       - backendRefs:
           - name: httpbin
             port: 8000
   EOF
   ```

2. Create a {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}} that enables request decompression on the route. This example accepts all three codecs. To keep the previous gzip-only behavior, set `requestDecompression: {}` or omit the `libraries` field.
   ```yaml
   kubectl apply -f- <<EOF
   apiVersion: {{< reuse "kgw-docs/snippets/trafficpolicy-apiversion.md" >}}
   kind: {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}}
   metadata:
     name: request-decompression
     namespace: httpbin
   spec:
     targetRefs:
     - group: gateway.networking.k8s.io
       kind: HTTPRoute
       name: httpbin-decompression
     compression:
       requestDecompression:
         libraries:
         - Gzip
         - Brotli
         - Zstd
   EOF
   ```

   | Setting | Description |
   |--|--|
   | `spec.targetRefs` | The HTTPRoute to apply this policy to. |
   | `spec.compression.requestDecompression` | Enables request decompression. When a request arrives with a `Content-Encoding` header that matches an offered codec, {{< reuse "kgw-docs/snippets/kgateway.md" >}} decompresses the body before forwarding the request to the backend. |
   | `spec.compression.requestDecompression.libraries` | The codecs to decompress, from `Gzip`, `Brotli`, and `Zstd`. Envoy picks the decompressor from the request's `Content-Encoding` header. Defaults to `[Gzip]` when unset. |

3. Create a gzip-compressed payload to use in the test.
   ```sh
   echo -n 'Hello, world!' | gzip > /tmp/payload.gz
   ```

4. Send the compressed payload to the `/post` httpbin path with the `Content-Encoding: gzip` header. Verify that the `data` field in the response shows the decompressed string `Hello, world!`, confirming that {{< reuse "kgw-docs/snippets/kgateway.md" >}} decompressed the request body before forwarding it to httpbin.
   {{< tabs >}}
   {{% tab name="Cloud Provider LoadBalancer" %}}
   ```sh
   curl -vik -X POST http://$INGRESS_GW_ADDRESS:8080/post \
     -H "host: decompression.example:8080" \
     -H "Content-Type: text/plain" \
     -H "Content-Encoding: gzip" \
     --data-binary @/tmp/payload.gz
   ```
   {{% /tab %}}
   {{% tab name="Port-forward for local testing" %}}
   ```sh
   curl -vik -X POST http://localhost:8080/post \
     -H "host: decompression.example" \
     -H "Content-Type: text/plain" \
     -H "Content-Encoding: gzip" \
     --data-binary @/tmp/payload.gz
   ```
   {{% /tab %}}
   {{< /tabs >}}

   Example output:
   ```json {hl_lines=[3]}
   {
     ...
     "data": "Hello, world!",
     "headers": {
       "Content-Type": [
         "text/plain"
       ],
       ...
     }
   }
   ```

## Cleanup

{{< reuse "kgw-docs/snippets/cleanup.md" >}}

```sh
kubectl delete {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}} response-compression -n httpbin
kubectl delete httproute httpbin-compression -n httpbin
kubectl delete {{< reuse "kgw-docs/snippets/trafficpolicy.md" >}} request-decompression -n httpbin
kubectl delete httproute httpbin-decompression -n httpbin
```
