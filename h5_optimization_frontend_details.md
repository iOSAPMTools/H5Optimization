# H5 秒开优化：前端层面详解

本篇文档详细阐述在 iOS `WKWebView` 中实现 H5 秒开的前端层面优化策略。前端代码的质量、资源加载方式和渲染策略直接决定了用户看到内容的速度和交互的流畅性。

## 3.2.1 关键渲染路径优化 (Critical Rendering Path)

关键渲染路径是指浏览器将 HTML、CSS 和 JavaScript 转换为屏幕上像素所经历的步骤序列。优化此路径是缩短白屏时间的核心。

*   **阻塞资源处理**:
    *   **CSS 阻塞**: 浏览器需要构建完整的 CSSOM 才能进行渲染树构建和布局。外部 CSS 文件会阻塞渲染。
        *   **优化**:
            *   **减少 CSS 文件数量**: 通过合并减少请求。
            *   **媒体查询分离**: 使用 `<link rel="stylesheet" href="print.css" media="print">` 将非屏幕显示的 CSS 分离，使其不阻塞初始渲染。
            *   **异步加载 CSS**: 使用 `<link rel="preload" href="style.css" as="style" onload="this.rel='stylesheet'">` 技巧，让 CSS 文件异步加载，加载完成后再生效（注意可能导致 FOUC - Flash of Unstyled Content）。
            *   **内联关键 CSS** (见下文)。
    *   **JavaScript 阻塞**: 默认情况下，`<script>` 标签（无论内联还是外部）会阻塞 HTML 解析。浏览器需要下载（如果是外部）、解析并执行完 JS 才能继续处理后面的 HTML。如果 JS 操作了 DOM 或 CSSOM，阻塞会更严重。
        *   **优化**:
            *   **将脚本移至 `<body>` 底部**: 确保脚本执行前 HTML 已基本解析完毕。
            *   **使用 `async` 属性**: `<script async src="script.js"></script>`。脚本会异步下载，下载完成后立即执行，执行时仍可能阻塞 HTML 解析。多个 `async` 脚本执行顺序不定。适合无依赖关系的独立脚本。
            *   **使用 `defer` 属性**: `<script defer src="script.js"></script>`。脚本会异步下载，但在 HTML 解析完成、`DOMContentLoaded` 事件触发之前按顺序执行。适合需要操作 DOM 或有依赖关系的脚本。
            *   **模块化 (ES Modules)**: `<script type="module">` 默认行为类似 `defer`。
            *   **减少首次加载的 JS 体积** (见下文 Tree Shaking, Code Splitting)。

*   **内联关键 CSS (Inline Critical CSS)**:
    *   **原理**: 将渲染首屏内容所必需的最小化 CSS 代码直接嵌入到 HTML 的 `<head>` 内的 `<style>` 标签中。这样浏览器无需等待外部 CSS 文件下载，拿到 HTML 后就能立即开始渲染首屏。非关键 CSS 则通过异步方式加载。
    *   **实现**:
        *   **手动提取**: 分析首屏元素，找出相关的 CSS 规则。非常繁琐且易出错。
        *   **自动化工具**: 使用如 `Critical` (Node.js 库) 或类似工具。它们可以启动一个无头浏览器加载页面，分析首屏视口内的元素及其样式，生成关键 CSS。
        ```javascript
        // 示例：使用 Critical 库 (需要在 Node.js 环境构建时运行)
        const critical = require('critical');

        critical.generate({
          base: 'dist/',
          src: 'index.html',
          target: 'index-critical.html', // 输出内联了关键 CSS 的 HTML
          inline: true,
          width: 1300, // 视口宽度
          height: 900, // 视口高度
          // 可选：将剩余 CSS 异步加载
          // extract: true,
          // assetPaths: ['dist/css']
        });
        ```
    *   **注意事项**:
        *   内联的 CSS 不应过大，否则会增加 HTML 文件体积，影响 TTFB 和初始下载。通常建议关键 CSS 压缩后 < 15KB。
        *   需要有机制异步加载剩余的完整 CSS。
        *   页面结构或样式变化时需要重新生成关键 CSS。

## 3.2.2 资源优化 (Resource Optimization)

减小资源体积，优化加载方式。

*   **图片优化**:
    *   **选择合适格式**:
        *   **WebP**: 通常比 JPG, PNG 提供更好的压缩率和质量，支持有损、无损、透明度和动画。iOS 14 及以上 `WKWebView` 支持。需要后端根据 `Accept` 请求头判断或前端 JS 判断后提供兼容格式。
        *   **AVIF**: 压缩率更高，但支持性更晚 (iOS 16)。
        *   **JPG**: 适合照片等复杂色彩图像。
        *   **PNG**: 适合需要透明背景的图像，或简单色彩图像（如图标、Logo）。
        *   **SVG**: 矢量格式，无限缩放不失真，适合图标、简单图形。文件体积通常较小。
    *   **压缩图片**: 使用工具（如 ImageOptim, TinyPNG/JPG, Squoosh.app）或构建插件（imagemin）在保证可接受质量的前提下尽可能压缩图片文件大小。
    *   **响应式图片**: 使用 `<picture>` 元素或 `srcset`、`sizes` 属性，让浏览器根据设备屏幕密度、尺寸和网络状况加载最合适的图片版本。
    *   **懒加载 (Lazy Loading)**: 对非首屏（视口之外）的图片，延迟加载。当图片滚动到视口附近时才开始加载。
        *   **实现**:
            *   **原生懒加载**: `<img src="placeholder.jpg" loading="lazy" data-src="real-image.jpg">`。`loading="lazy"` 属性浏览器原生支持（iOS 15.4 Safari/WKWebView 支持）。简单方便，但兼容性和触发时机控制有限。
            *   **JavaScript + Intersection Observer**: 使用 `IntersectionObserver` API 监测图片是否进入视口，进入后再将 `data-src` 的真实 URL 赋值给 `src`。兼容性好，控制精确。有许多现成的 JS 库（如 lazysizes）。

*   **代码压缩与混淆 (Minification & Uglification)**:
    *   **原理**:
        *   **压缩 (Minification)**: 从代码中移除不必要的字符，如空格、换行、注释，而不改变其功能。适用于 CSS, JS, HTML。
        *   **混淆 (Uglification/Mangling)**: 重命名变量、函数名等为更短的名称（如 `a`, `b`），进一步减小 JS 文件体积。
    *   **实现**: 使用构建工具的插件：
        *   **JavaScript**: `TerserWebpackPlugin` (Webpack), `rollup-plugin-terser` (Rollup)。
        *   **CSS**: `CssMinimizerWebpackPlugin` (Webpack), `cssnano` (PostCSS 插件)。
        *   **HTML**: `HtmlWebpackPlugin` (Webpack) 通常有压缩选项，或使用 `html-minifier`。

*   **移除无用代码 (Tree Shaking)**:
    *   **原理**: 在 JavaScript 打包过程中，静态分析代码中的 `import` 和 `export` 语句，找出项目中未被任何地方引用的"死代码"（dead code），并在最终打包结果中将其移除。
    *   **实现**: 依赖于 ES6 模块语法 (`import`/`export`)。构建工具如 Webpack 和 Rollup 默认支持 Tree Shaking。
    *   **要求**:
        *   代码必须使用 ES6 模块。
        *   确保构建工具配置正确开启了优化（如 Webpack 的 `mode: 'production'`）。
        *   避免副作用（side effects）。如果一个模块导入了但未使用其导出，但该模块本身执行时有副作用（如修改全局变量、自动运行代码），Tree Shaking 可能无法安全移除它。可以通过在 `package.json` 中设置 `"sideEffects": false` 或列出有副作用的文件来帮助构建工具判断。

*   **代码分割 (Code Splitting)**:
    *   **原理**: 将巨大的 JS 包拆分成多个小块（chunks），然后按需加载或并行加载。这样初始加载时只需下载核心代码，其他功能模块的代码在需要时再加载，从而加快首屏渲染速度。
    *   **实现**:
        *   **入口点分割**: 配置多个入口文件（Webpack `entry`）。
        *   **动态导入 (Dynamic `import()` )**: 在代码中使用 `import('module-path')` 语法。这会返回一个 Promise，构建工具会自动将 `module-path` 及其依赖打包成一个独立的 chunk，在代码执行到 `import()` 时才异步加载。适用于按路由加载组件、用户触发某个功能时加载相关代码等场景。
        *   **提取公共依赖**: 使用构建工具配置（如 Webpack 的 `SplitChunksPlugin`）将多个入口或动态块共享的公共模块提取到单独的文件中，利用浏览器缓存。

## 3.2.3 服务端渲染 (SSR) 与预渲染 (Prerendering)

*   **服务端渲染 (Server-Side Rendering, SSR)**:
    *   **原理**: 客户端（浏览器/WKWebView）请求页面时，服务器（通常是 Node.js 环境）执行前端框架（如 React, Vue, Angular）的代码，将组件渲染成完整的 HTML 字符串，连同所需的数据一起发送给客户端。客户端接收到 HTML 后可以立即显示内容，然后再进行 "hydration"（水合，即绑定事件监听器，使页面可交互）。
    *   **优势**:
        *   **极快的首屏加载速度 (FCP)**: 用户能非常快地看到内容，极大改善白屏。
        *   **利于 SEO**: 搜索引擎爬虫可以直接抓取到完整内容的 HTML。
    *   **劣势**:
        *   **服务器压力增大**: 需要在服务器端执行渲染逻辑。
        *   **实现复杂度高**: 需要 Node.js 环境，对项目架构有要求（代码需同构）。
        *   **TTI (Time to Interactive) 可能延迟**: 用户虽然看到内容，但需要等待 JS 下载执行完毕并完成 hydration 后才能交互。
    *   **适用场景**: 对首屏性能要求极高、需要 SEO 的动态内容网站（如新闻、电商）。
    *   **框架支持**: Next.js (React), Nuxt.js (Vue), Angular Universal。

*   **预渲染 (Prerendering)**:
    *   **原理**: 在**构建阶段**，使用无头浏览器（如 Puppeteer）启动应用，访问指定的路由，将其渲染结果保存为静态的 HTML 文件。部署时将这些 HTML 文件连同静态资源一起部署。客户端请求时直接返回这个静态 HTML。
    *   **优势**:
        *   **首屏加载速度快**: 类似 SSR，直接返回带内容的 HTML。
        *   **利于 SEO**: 同 SSR。
        *   **无需服务端运行环境**: 最终产物是纯静态文件，可部署在任何静态文件服务器或 CDN 上。服务器压力小。
    *   **劣势**:
        *   **只适用于静态内容或有限的动态路由**: 无法为每个用户动态生成内容。如果页面内容频繁变化或路由无限多，预渲染不适用。
        *   **构建时间增加**: 需要在构建时渲染每个目标页面。
    *   **适用场景**: 静态网站、文档、博客、营销页面等内容相对固定的场景。
    *   **实现**: 使用构建工具插件，如 `prerender-spa-plugin` (Webpack), 或框架自带支持 (如 Nuxt.js 的 `nuxt generate`)。

## 3.2.4 骨架屏 (Skeleton Screen)

*   **原理**: 在等待页面真实内容加载和渲染完成之前，先向用户展示一个页面的大致布局框架的占位符（通常是灰色或有微弱动画的形状），模拟内容的轮廓。它并不能缩短实际的白屏时间（FCP 可能不变甚至略微增加），但能显著改善用户等待时的感知体验，让用户觉得"加载正在进行"。
*   **实现**:
    *   **手动编写**: 根据页面布局，用简单的 HTML 和 CSS 绘制出骨架结构。在数据加载完成后，隐藏骨架屏，显示真实内容。
    *   **自动化生成**:
        *   **工具/库**: 如 `vue-content-loader`, `react-content-loader` 提供预设或可定制的 SVG 骨架屏组件。
        *   **页面结构生成**: 一些工具（如 `page-skeleton-webpack-plugin`）尝试在构建时分析页面 DOM 结构和样式，自动生成骨架屏代码。可能需要调整。
        *   **服务端渲染骨架屏**: SSR 应用可以在服务器端就渲染出骨架屏的 HTML。
    *   **注入方式**:
        *   **前端注入**: 在 JS 开始请求数据时显示骨架屏。
        *   **Native 注入**: Native 在加载 `WKWebView` 时，先展示一个 Native 实现的骨架屏视图覆盖在 WebView 上，待 H5 内容加载差不多（通过 JSBridge 通知）再移除。体验可能更优。
        *   **SSR/预渲染输出**: 直接在 HTML 中包含骨架屏结构。
*   **注意事项**:
    *   骨架屏的样式应尽可能贴近真实内容的布局，否则切换时会很突兀。
    *   设计不宜过于复杂，避免骨架屏本身加载慢。

## 3.2.5 PWA Service Worker 缓存

*   **原理**: Service Worker (SW) 是一个运行在浏览器后台的独立脚本，充当浏览器与网络之间的代理。它可以拦截网络请求，并根据预设的缓存策略来响应请求：从缓存读取、从网络获取、或两者结合。利用 SW 可以实现强大的资源缓存，使得二次访问 H5 页面时，核心资源（HTML, CSS, JS, 图片等）可以直接从缓存加载，速度极快，甚至可以离线访问。
*   **实现**:
    *   **创建 SW 脚本**: 编写 `sw.js` 文件，定义 `install`, `activate`, `fetch` 等事件的处理逻辑。`fetch` 事件是核心，用于拦截请求并决定如何响应。
    *   **注册 SW**: 在主页面 JS 中注册 SW 文件: `navigator.serviceWorker.register('/sw.js')`.
    *   **缓存策略**: 在 `install` 事件中缓存核心静态资源（App Shell），在 `fetch` 事件中定义请求的处理逻辑（如 Cache First, Network First, Stale While Revalidate 等）。
    *   **工具**: 使用 Workbox (Google 出品) 可以极大简化 SW 的编写和缓存管理。它提供了多种预设策略和构建工具集成。
*   **优势**:
    *   **极快的二次加载速度**: 核心资源来自缓存。
    *   **离线访问能力**: 即使网络断开也能访问已缓存的页面。
    *   **推送通知、后台同步**等 PWA 特性（虽然在 `WKWebView` 中支持可能有限或需要额外配置）。
*   **注意事项**:
    *   **HTTPS**: Service Worker 必须运行在 HTTPS 环境下（localhost 除外）。
    *   **缓存更新**: 需要设计好 SW 更新和旧缓存清理的机制，避免用户一直使用旧版本资源。Workbox 提供了解决方案。
    *   **`WKWebView` 支持**: `WKWebView` 从 iOS 11.3 开始支持 Service Worker，但可能有一些限制或行为差异，需要测试验证。确保 `WKWebViewConfiguration` 中的相关设置开启。
    *   **首次加载**: SW 对首次加载没有优化效果，反而可能因为需要注册和安装 SW 而略微增加开销。其优势体现在后续访问。 