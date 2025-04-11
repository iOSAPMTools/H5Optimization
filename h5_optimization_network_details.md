# H5 秒开优化：网络层面详解

本篇文档详细阐述在 iOS `WKWebView` 中实现 H5 秒开的网络层面优化策略。网络传输是影响 H5 加载性能的关键环节，优化网络可以显著减少白屏时间。

## 3.1.1 DNS 预解析 (DNS Prefetching)

*   **原理**: 浏览器或 App 在用户实际访问某个域名之前，就提前完成该域名的 DNS 查询工作，并将结果缓存。当用户真正请求该域名下的资源时，可以直接使用缓存的 IP 地址，跳过 DNS 查询步骤，节省时间（通常几十到几百毫秒）。
*   **H5 实现**:
    ```html
    <link rel="dns-prefetch" href="//cdn.example.com">
    <link rel="dns-prefetch" href="//api.example.com">
    ```
    将这些标签添加到 HTML 的 `<head>` 中。通常用于第三方域名、CDN 域名、API 域名等。浏览器会自动处理。
*   **Native (iOS) 实现**:
    *   **手动触发**: 在 App 认为合适的时机（如启动时、进入包含 WebView 的页面前），可以主动发起对目标域名的网络请求（例如使用 `URLSession` 发起一个 HEAD 请求或一个专门的 DNS 查询库），系统会自动缓存 DNS 结果。
    *   **系统级**: iOS 系统本身也有一定的 DNS 缓存机制。主动预热可以提高命中率。
*   **注意事项**:
    *   不要预解析过多域名，以免造成不必要的 DNS 查询开销。仅针对关键、高频访问的域名进行预解析。
    *   `dns-prefetch` 只负责 DNS 解析，不建立连接。

## 3.1.2 TCP/TLS 预连接 (Preconnect)

*   **原理**: 比 DNS 预解析更进一步，不仅解析 DNS，还预先建立 TCP 连接（三次握手），甚至完成 TLS 握手。当实际请求发生时，可以直接复用这个已经建立好的安全连接发送数据，节省更多时间（包括 RTT 和 TLS 协商时间，可能节省几百毫秒到 1 秒以上）。
*   **H5 实现**:
    ```html
    <link rel="preconnect" href="https://api.example.com">
    <!-- 可选：如果跨域资源的请求需要携带凭证（如 Cookie） -->
    <link rel="preconnect" href="https://api.example.com" crossorigin>
    ```
    添加到 HTML 的 `<head>` 中。通常用于最关键的服务域名，如 API 接口、核心资源 CDN 等。
*   **Native (iOS) 实现**:
    *   使用底层网络库（如 `CFNetwork` 或某些第三方网络库）提供的接口尝试预建立 TCP/TLS 连接。这通常比 H5 的 `preconnect` 更可控，但实现也更复杂。
*   **注意事项**:
    *   `preconnect` 比 `dns-prefetch` 消耗更多资源（服务器和客户端都需要维持连接）。因此，仅对 1-2 个最关键的域名使用。
    *   如果页面在 10 秒内未使用预连接的通道，浏览器可能会关闭它。

## 3.1.3 CDN 加速 (CDN Acceleration)

*   **原理**: 内容分发网络 (Content Delivery Network) 将源站的静态资源（HTML, CSS, JS, 图片, 字体, 视频等）缓存到全球各地靠近用户的边缘节点服务器上。用户请求资源时，会被智能 DNS 或其他技术导向到地理位置最近、访问速度最快的节点，从而减少网络延迟，提高下载速度。
*   **实现**:
    *   选择 CDN 服务商（如阿里云、腾讯云、AWS CloudFront, Cloudflare 等）。
    *   将项目的静态资源上传或同步到 CDN 服务商提供的存储空间（如对象存储 OSS）。
    *   配置 CDN 加速域名，并将其指向源站或存储空间。
    *   在 H5 代码中，将静态资源的 URL 替换为 CDN 加速域名。
    *   合理配置 CDN 的缓存策略（缓存时间、刷新机制等）。
*   **优势**:
    *   显著降低全球用户的访问延迟。
    *   分担源站压力，提高可用性。
    *   通常提供额外的安全防护功能。
*   **注意事项**:
    *   确保 CDN 缓存配置正确，避免资源更新不及时。
    *   对于动态内容（如 API 接口），CDN 通常只做路径优化或有限的缓存，效果不如静态资源明显。可以考虑使用全站加速产品。

## 3.1.4 HTTP/2 或 HTTP/3

*   **原理**:
    *   **HTTP/2**:
        *   **多路复用 (Multiplexing)**: 在单个 TCP 连接上并行处理多个请求和响应，避免了 HTTP/1.1 的队头阻塞问题。
        *   **头部压缩 (Header Compression)**: 使用 HPACK 算法压缩请求和响应头部，减少传输数据量。
        *   **服务器推送 (Server Push)**: 服务器可以在客户端请求 HTML 时，主动将页面可能需要的 CSS、JS 等资源推送到客户端缓存中（需谨慎使用）。
    *   **HTTP/3**:
        *   基于 **QUIC** 协议（运行在 UDP 之上），进一步解决了 TCP 层的队头阻塞问题。
        *   更快的连接建立（0-RTT 或 1-RTT）。
        *   连接迁移（网络切换时保持连接）。
*   **实现**:
    *   在 Web 服务器（Nginx, Apache, Caddy 等）或负载均衡器上启用 HTTP/2 或 HTTP/3 支持。需要 HTTPS 环境。
    *   确保 CDN 服务商也支持并开启了相应协议。
    *   客户端（现代浏览器和 `WKWebView`）通常自动支持。
*   **优势**:
    *   在加载大量小资源（图片、CSS、JS 模块）时效果显著。
    *   减少了 TCP 连接数，降低服务器压力。
    *   在网络状况不稳定时表现更好（尤其是 HTTP/3）。
*   **注意事项**:
    *   服务器推送（Server Push）配置不当可能推送过多无用资源，反而降低性能，目前使用较少。
    *   HTTP/3 对服务器和网络设备要求更高。

## 3.1.5 网络请求合并 (Request Bundling)

*   **原理**: 减少页面加载所需的 HTTP 请求总数。每个 HTTP 请求都有额外的开销（DNS 查询、连接建立、TLS 协商、请求/响应头等）。合并请求可以减少这些固定开销。
*   **实现**:
    *   **CSS/JS 文件合并**: 使用构建工具（Webpack, Rollup, Parcel 等）将多个 CSS 或 JS 文件打包成一个或少数几个文件。
    *   **CSS Sprites**: 将多个小图标、背景图合并到一张大图中，通过 CSS 的 `background-image` 和 `background-position` 来显示需要的部分。有在线工具或构建插件可以自动生成。
    *   **Icon Fonts**: 将图标制作成字体文件，通过 CSS 引入并使用特定字符来显示图标。优点是矢量、可通过 CSS 控制颜色大小，缺点是通常只支持单色，加载字体文件也有开销。
    *   **SVG Sprites**: 将多个 SVG 图标合并到一个 SVG 文件中，通过 `<symbol>` 和 `<use>` 标签引用。矢量、可通过 CSS 控制样式。
    *   **接口合并**: 后端提供聚合接口，将前端原本需要多次请求的数据一次性返回（需要后端支持）。
*   **注意事项**:
    *   过度合并可能导致单个文件过大，反而影响加载和缓存效率。需要权衡，可以按需加载或分块打包。
    *   HTTP/2 的多路复用在一定程度上降低了合并请求的必要性，但并非完全替代，适度合并仍有益。

## 3.1.6 传输压缩 (Gzip/Brotli Compression)

*   **原理**: 服务器在发送 HTTP 响应之前，对文本类资源（HTML, CSS, JS, JSON, XML, SVG 等）使用 Gzip 或 Brotli 算法进行压缩。浏览器在接收到响应后自动解压。这可以大幅减小传输文件的大小（通常压缩率 50%-80%），从而加快下载速度。Brotli 通常提供比 Gzip 更高的压缩率（约 15-25%），但压缩时消耗稍多 CPU。
*   **实现**:
    *   **Web 服务器配置**:
        *   **Nginx**: 在 `nginx.conf` 的 `http`, `server` 或 `location` 块中启用 `gzip on;` 或 `brotli on;`，并配置 `gzip_types` / `brotli_types` 指定需要压缩的文件 MIME 类型。
        *   **Apache**: 使用 `mod_deflate` (Gzip) 或 `mod_brotli` 模块，并在 `.htaccess` 或配置文件中启用。
    *   **CDN 配置**: 大部分 CDN 服务商提供 Gzip/Brotli 压缩的开关选项。
    *   **构建时预压缩**: 对于静态资源，可以在构建阶段就生成 `.gz` 和 `.br` 格式的预压缩文件，服务器可以直接发送这些文件（需要服务器支持，如 Nginx 的 `gzip_static on;` / `brotli_static on;`），避免运行时压缩的 CPU 开销。
*   **验证**: 使用浏览器开发者工具的网络(Network)面板，查看响应头中的 `Content-Encoding: gzip` 或 `Content-Encoding: br`，并比较 `Content-Length` (压缩后) 和原始大小。
*   **注意事项**:
    *   不要对已经压缩过的文件（如 JPG, PNG, MP4, ZIP）再次压缩，效果甚微且浪费 CPU。
    *   确保服务器和客户端都支持相应的压缩算法。现代浏览器和 `WKWebView` 都支持 Gzip 和 Brotli。 