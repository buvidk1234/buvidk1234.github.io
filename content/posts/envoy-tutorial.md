+++
date = '2026-07-19T18:00:00+08:00'
draft = false
title = 'Envoy 请求生命周期与静态代理配置教程'
tags = ['Envoy', 'Proxy', 'Service Mesh']
+++

# Envoy 请求生命周期与静态代理配置教程

## 适用范围

本文基于 Envoy v1.39.0 官方文档的 `Life of a Request`、快速开始、HTTP 路由、Admin、xDS 和访问日志文档，说明 Envoy 作为 HTTP 反向代理时的核心模型、静态配置方式、请求处理流程和排障入口。

示例目标：

- 本地启动一个后端 HTTP 服务。
- Envoy 监听 `10000` 端口。
- Envoy 将所有 HTTP 请求转发到 `backend:80`。
- Envoy Admin 监听 `9901` 端口，用于查看运行时配置和指标。

## 核心术语

`downstream`：连接到 Envoy 的一侧。对于边缘代理，通常是外部客户端；对于 sidecar，可能是本地应用或服务网格内的其他代理。

`upstream`：Envoy 转发请求的目标端。通常是业务服务实例。

`listener`：绑定 IP 和端口，接收 TCP 连接或 UDP 数据报。一个 Envoy 进程可以有多个 listener。

`filter chain`：Envoy 的处理管线。listener filter 处理连接元数据，network filter 处理 L3/L4 字节流，HTTP filter 处理 HTTP 请求/响应流。

`HTTP connection manager`：HTTP 代理场景中的核心 network filter，负责协议编解码、路由匹配、HTTP filter chain 管理、访问日志、统计指标和本地响应。

`route`：HTTP 请求到 upstream cluster 的匹配规则。典型匹配条件包括 `:authority`、path、prefix、header、query parameter。

`cluster`：Envoy 中的上游服务抽象。cluster 内包含一组 endpoint、负载均衡策略、连接池、健康检查、熔断、异常点剔除和上游 TLS 等配置。

`endpoint`：cluster 中的具体网络地址，例如 `10.0.1.10:8080`。

`xDS`：Envoy 动态配置 API 的统称。LDS 管理 listener，RDS 管理 route，CDS 管理 cluster，EDS 管理 endpoint，SDS 管理证书和密钥。

## 请求处理模型

HTTP 请求进入 Envoy 后，数据路径可以拆成两个子系统：

- `listener subsystem`：处理 downstream 连接、下游协议编解码、HTTP filter chain、响应回写。
- `cluster subsystem`：处理 upstream cluster 选择、endpoint 选择、连接池、负载均衡、健康状态和熔断。

两者通过 `envoy.filters.http.router` 连接。router filter 在 HTTP filter chain 末尾执行路由选择，并向 cluster manager 获取上游连接池。

![envoy-1](/images/envoy-1.png)

典型 HTTP 请求路径：

```text
downstream client
  -> listener accept
  -> listener filters
  -> filter chain match
  -> transport socket
  -> network filters
  -> HTTP connection manager
  -> HTTP codec
  -> downstream HTTP filters
  -> router filter
  -> cluster manager
  -> load balancer
  -> upstream connection pool
  -> upstream HTTP codec
  -> upstream endpoint
```

响应路径按 HTTP filter 和 network filter 的反向顺序返回：

```text
upstream endpoint
  -> upstream HTTP codec
  -> router filter
  -> downstream HTTP filters, reverse order
  -> HTTP connection manager
  -> network filters, reverse order
  -> transport socket
  -> downstream client
```

## 线程与连接生命周期

Envoy 主线程负责进程生命周期、配置加载、统计刷新等控制面工作。worker 线程处理请求数据面。一个 downstream TCP 连接在生命周期内固定由一个 worker 线程处理，同一连接上的 HTTP/2 或 HTTP/3 多路复用 stream 也归属该 worker。

每个 worker 维护自己的 listener 实例和 upstream connection pool。worker 之间尽量不共享请求态数据，这使 Envoy 可以按 CPU 核心数扩展。

listener 有三个关键状态：

- `warming`：等待依赖配置就绪，例如 RDS route、SDS secret、cluster 初始化。
- `active`：已绑定监听地址，可以接收新连接。
- `draining`：不再接收新连接，已有连接在 drain 时间内继续处理。

![envoy-2](/images/envoy-2.png)

## Filter Chain 执行顺序

![envoy-3](/images/envoy-3.png)

### Listener Filter

listener filter 在连接刚被 accept 后执行，用于提取连接级元数据。典型例子是 `envoy.filters.listener.tls_inspector`，它可以在 TLS 握手早期提取 SNI 和 ALPN，供 filter chain match 使用。

如果 listener 有多个 `filter_chains`，Envoy 会根据目标地址、SNI、ALPN、源端口等条件选择最匹配的 filter chain。没有匹配项时，如果配置了 `default_filter_chain`，使用默认链；否则关闭连接。

### Transport Socket

transport socket 负责连接上的传输层封装。明文 HTTP 可以不配置 TLS transport socket；HTTPS/mTLS 需要配置 downstream 或 upstream TLS context。

在 downstream TLS 场景中，transport socket 先完成 TLS 握手和解密，再把明文字节交给 network filter chain。upstream TLS 场景中，Envoy 在向上游写出前完成加密。

### Network Filter

![envoy-4](/images/envoy-4.png)

network filter 处理连接级字节流。HTTP 代理中最关键的是：

```text
envoy.filters.network.http_connection_manager
```

HTTP connection manager 创建 HTTP codec，将 HTTP/1.1、HTTP/2、HTTP/3 的协议细节抽象成请求/响应的 headers、body、trailers。

### HTTP Filter

HTTP filter 按 stream 维度执行，不直接处理 TCP 字节。请求路径执行 decoder filter，响应路径执行 encoder filter，decoder/encoder filter 两边都会执行。

![envoy-5](/images/envoy-5.png)

HTTP filter chain 的最后一个 filter 通常是：

```text
envoy.filters.http.router
```

router filter 执行以下操作：

- 固定当前 route 匹配结果。
- 获取 route 指向的 cluster 名称。
- 向 cluster manager 请求 HTTP connection pool。
- 为请求创建 upstream stream。
- 执行 timeout、retry、shadow、hedge 等路由策略。

HTTP filter 可以修改请求头并触发 route cache 重新计算。进入 router filter 后，route 选择被最终确定。

![envoy-6](/images/envoy-6.png)

## 最小静态代理配置

以下配置使用完全静态资源：listener、route、cluster、endpoint 都写在 bootstrap 文件中。

`envoy.yaml`：

```yaml
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901

static_resources:
  listeners:
  - name: listener_http
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          normalize_path: true
          merge_slashes: true
          access_log:
          - name: envoy.access_loggers.stdout
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
              log_format:
                text_format_source:
                  inline_string: "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS% %DURATION% \"%REQ(X-REQUEST-ID)%\" \"%UPSTREAM_HOST%\"\n"
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend_service
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: backend
                  timeout: 5s
                  retry_policy:
                    retry_on: 5xx,connect-failure,refused-stream
                    num_retries: 2
                    per_try_timeout: 2s
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: backend
    type: STRICT_DNS
    connect_timeout: 0.25s
    lb_policy: ROUND_ROBIN
    dns_lookup_family: V4_ONLY
    load_assignment:
      cluster_name: backend
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: backend
                port_value: 80
    circuit_breakers:
      thresholds:
      - priority: DEFAULT
        max_connections: 1024
        max_pending_requests: 1024
        max_requests: 2048
        max_retries: 3
    outlier_detection:
      consecutive_5xx: 5
      interval: 10s
      base_ejection_time: 30s
      max_ejection_percent: 50
```

配置结构：

- `admin`：开启 Admin 监听端口。
- `static_resources.listeners`：定义 downstream 入口。
- `filter_chains.filters`：把 HTTP connection manager 挂到 listener。
- `route_config.virtual_hosts`：按 host 或 `:authority` 选择虚拟主机。
- `routes`：按 path/prefix/header 等条件选择 route。
- `route.cluster`：将请求交给指定 cluster。
- `clusters`：定义 upstream 服务、服务发现方式、连接超时、负载均衡、endpoint、熔断和异常点剔除。

## 使用 Docker Compose 运行

目录结构：

```text
envoy-demo/
  docker-compose.yaml
  envoy.yaml
  html/
    index.html
```

`docker-compose.yaml`：

```yaml
services:
  backend:
    image: nginx:1.27-alpine
    volumes:
    - ./html:/usr/share/nginx/html:ro

  envoy:
    image: envoyproxy/envoy:v1.39.0
    ports:
    - "10000:10000"
    - "9901:9901"
    volumes:
    - ./envoy.yaml:/etc/envoy/envoy.yaml:ro
```

创建测试页面：

```bash
mkdir -p envoy-demo/html
cd envoy-demo
printf 'backend ok\n' > html/index.html
```

把上面的 `envoy.yaml` 和 `docker-compose.yaml` 放入 `envoy-demo/`。

校验配置：

```bash
docker compose run --rm envoy --mode validate -c /etc/envoy/envoy.yaml
```

启动：

```bash
docker compose up -d
```

请求代理入口：

```bash
curl -v http://localhost:10000/
```

预期返回：

```text
backend ok
```

查看 Envoy 访问日志：

```bash
docker compose logs -f envoy
```

成功请求的 `RESPONSE_FLAGS` 通常为 `-`，`RESPONSE_CODE_DETAILS` 通常为 `via_upstream`，`UPSTREAM_HOST` 为被选中的 backend endpoint。

## 请求如何映射到配置

一次 `curl http://localhost:10000/` 的配置映射如下：

1. TCP 连接进入 `listener_http`，监听地址是 `0.0.0.0:10000`。
2. 当前 listener 只有一个 filter chain，因此直接选择该链。
3. 当前 filter chain 只有 `envoy.filters.network.http_connection_manager`。
4. HTTP connection manager 解码 HTTP/1.1 请求。
5. HCM 使用 `route_config.name = local_route`。
6. `domains: ["*"]` 匹配任意 Host 或 `:authority`。
7. `prefix: "/"` 匹配所有路径。
8. route 指向 `cluster: backend`。
9. cluster manager 查找名为 `backend` 的 cluster。
10. `STRICT_DNS` 解析 Docker Compose 服务名 `backend`。
11. `ROUND_ROBIN` 在健康 endpoint 中选择一个 upstream host。
12. router filter 通过 upstream connection pool 创建或复用连接。
13. 响应按 filter reverse order 返回 downstream。
14. 请求结束后写 access log、更新 stats、结束 trace span。

## 路由配置要点

HTTP route 先匹配 virtual host，再在 virtual host 内匹配 route。

virtual host 匹配：

```yaml
virtual_hosts:
- name: api
  domains:
  - "api.example.com"
  - "api.example.com:443"
```

route 顺序匹配：

```yaml
routes:
- match:
    prefix: "/api/v1/admin"
  route:
    cluster: admin_api
- match:
    prefix: "/api/v1"
  route:
    cluster: user_api
```

更具体的规则应放在前面。`prefix: "/api/v1/admin"` 如果放在 `prefix: "/api/v1"` 后面，将不会被命中。

常见 route 动作：

```yaml
route:
  cluster: user_api
  timeout: 3s
  prefix_rewrite: "/"
  retry_policy:
    retry_on: 5xx,connect-failure,refused-stream
    num_retries: 2
    per_try_timeout: 1s
```

多 cluster 权重转发：

```yaml
route:
  weighted_clusters:
    clusters:
    - name: user_api_v1
      weight: 90
    - name: user_api_v2
      weight: 10
```

直接返回：

```yaml
direct_response:
  status: 204
```

重定向：

```yaml
redirect:
  https_redirect: true
```

## Cluster 配置要点

静态 endpoint：

```yaml
clusters:
- name: static_api
  type: STATIC
  connect_timeout: 0.25s
  lb_policy: ROUND_ROBIN
  load_assignment:
    cluster_name: static_api
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: 10.0.1.10
              port_value: 8080
      - endpoint:
          address:
            socket_address:
              address: 10.0.1.11
              port_value: 8080
```

DNS endpoint：

```yaml
clusters:
- name: dns_api
  type: STRICT_DNS
  connect_timeout: 0.25s
  lb_policy: ROUND_ROBIN
  dns_lookup_family: V4_ONLY
  load_assignment:
    cluster_name: dns_api
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: api.internal
              port_value: 8080
```

常用负载均衡策略：

- `ROUND_ROBIN`：轮询。
- `LEAST_REQUEST`：优先选择活跃请求更少的 host。
- `RANDOM`：随机选择 host。
- `RING_HASH`：一致性哈希。
- `MAGLEV`：Maglev 哈希负载均衡。

主动健康检查示例：

```yaml
health_checks:
- timeout: 1s
  interval: 5s
  unhealthy_threshold: 2
  healthy_threshold: 2
  http_health_check:
    path: /healthz
```

异常点剔除示例：

```yaml
outlier_detection:
  consecutive_5xx: 5
  interval: 10s
  base_ejection_time: 30s
  max_ejection_percent: 50
```

熔断示例：

```yaml
circuit_breakers:
  thresholds:
  - priority: DEFAULT
    max_connections: 1024
    max_pending_requests: 1024
    max_requests: 2048
    max_retries: 3
```

`circuit_breakers` 在 cluster 维度限制连接数、等待请求数、并发请求数和重试数。触发后，访问日志中的响应标记可用于定位资源溢出。

## Admin 接口

Admin 端口用于查看运行时状态。生产环境不要把 Admin 暴露给不可信网络。推荐绑定 `127.0.0.1`、通过 sidecar 本地访问，或使用 `allow_paths` 限制可访问路径。

健康状态：

```bash
curl -s http://localhost:9901/ready
```

配置快照：

```bash
curl -s http://localhost:9901/config_dump
```

只看 listener：

```bash
curl -s 'http://localhost:9901/config_dump?resource=dynamic_listeners'
```

查看 cluster：

```bash
curl -s http://localhost:9901/clusters
```

查看 HTTP 指标：

```bash
curl -s 'http://localhost:9901/stats?filter=^http\.ingress_http'
```

查看 cluster 指标：

```bash
curl -s 'http://localhost:9901/stats?filter=^cluster\.backend'
```

常用指标：

- `http.<stat_prefix>.downstream_rq_total`：HCM 接收的 downstream 请求总数。
- `http.<stat_prefix>.downstream_rq_time`：downstream 请求耗时分布。
- `cluster.<cluster>.upstream_rq_total`：发往某个 cluster 的请求总数。
- `cluster.<cluster>.upstream_cx_active`：活跃 upstream 连接数。
- `cluster.<cluster>.upstream_rq_pending_active`：等待连接池分配的请求数。
- `listener.<address>.downstream_cx_active`：listener 当前活跃 downstream 连接数。

## 访问日志字段

示例配置输出以下字段：

```text
[%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%" %RESPONSE_CODE% %RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS% %DURATION% "%REQ(X-REQUEST-ID)%" "%UPSTREAM_HOST%"
```

字段含义：

- `%START_TIME%`：请求开始时间。
- `%REQ(:METHOD)%`：HTTP method。
- `%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%`：优先打印原始路径，否则打印当前 `:path`。
- `%PROTOCOL%`：HTTP 协议版本。
- `%RESPONSE_CODE%`：响应状态码。
- `%RESPONSE_FLAGS%`：Envoy 处理异常标记，成功时通常为 `-`。
- `%RESPONSE_CODE_DETAILS%`：Envoy 内部响应原因，例如 `via_upstream`、`route_not_found`、`cluster_not_found`、`no_healthy_upstream`、`upstream_per_try_timeout`。
- `%DURATION%`：请求总耗时，单位毫秒。
- `%REQ(X-REQUEST-ID)%`：请求 ID。
- `%UPSTREAM_HOST%`：被选中的 upstream host。

常见 `RESPONSE_FLAGS`：

| 标记 | 含义 |
| --- | --- |
| `NR` | 没有匹配 route，通常对应 `404` |
| `NC` | route 指向的 cluster 不存在 |
| `UH` | cluster 中没有健康 upstream host |
| `UF` | upstream 连接失败 |
| `UO` | upstream 熔断或资源溢出 |
| `UT` | upstream 请求超时，通常对应 `504` |
| `URX` | 重试次数耗尽 |
| `DC` | downstream 连接终止 |

排障时同时看 `%RESPONSE_FLAGS%` 和 `%RESPONSE_CODE_DETAILS%`。状态码只能说明结果类型，无法准确区分路由未命中、cluster 不存在、endpoint 不健康、连接失败、超时或熔断。

## xDS 配置演进

完全静态配置适合单机实验、固定拓扑和低频变更。动态环境通常按以下顺序拆分：

- 只变 endpoint：静态 listener、route、cluster，endpoint 通过 EDS 下发。
- 变 cluster：通过 CDS 下发 cluster，通常与 EDS 配合。
- 变 route：通过 RDS 下发 route configuration。
- 变 listener：通过 LDS 下发 listener 和 filter chain。
- 变证书：通过 SDS 下发 TLS secret。
- 多资源有依赖顺序：通过 ADS 在单个 gRPC stream 上聚合下发。

避免流量中断的更新原则是 `make before break`：

1. 先下发新 cluster。
2. 再下发新 endpoint。
3. 再下发引用新 cluster 的 listener 或 route。
4. 旧 route 不再引用旧 cluster 后，再移除旧 cluster 和 endpoint。

RDS route 更新不会等待 cluster warming。控制面必须保证 route 引用的 cluster 已经存在并完成初始化，否则会出现 `NC`、`UH` 或短暂 503。

## TLS 与 mTLS 配置位置

downstream TLS 配置在 listener filter chain 的 `transport_socket`：

```yaml
filter_chains:
- transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
      common_tls_context:
        tls_certificates:
        - certificate_chain:
            filename: /etc/envoy/certs/server.crt
          private_key:
            filename: /etc/envoy/certs/server.key
  filters:
  - name: envoy.filters.network.http_connection_manager
```

upstream TLS 配置在 cluster 的 `transport_socket`：

```yaml
clusters:
- name: https_backend
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      sni: api.example.com
```

mTLS 比单向 TLS 多出客户端证书、私钥和对端证书校验配置。证书频繁轮换时应使用 SDS，而不是频繁重启 Envoy。

## 排障流程

### 配置无法启动

先运行 validate mode：

```bash
docker compose run --rm envoy --mode validate -c /etc/envoy/envoy.yaml
```

检查项：

- `typed_config["@type"]` 是否使用 v3 类型 URL。
- filter 名称是否与 extension 注册名一致。
- HCM 是否配置 `stat_prefix`。
- HCM 是否且只是否配置了 `route_config`、`rds`、`scoped_routes` 之一。
- route 指向的 cluster 名称是否存在。
- cluster 的 `load_assignment.cluster_name` 是否与 cluster `name` 一致。

### 请求返回 404

优先看 access log：

```text
response_code=404 response_flags=NR response_code_details=route_not_found
```

检查项：

- Host 或 `:authority` 是否匹配 `virtual_hosts.domains`。
- route 顺序是否错误。
- `prefix`、`path`、`safe_regex` 是否符合预期。
- HCM path normalization 是否改变了匹配路径。

### 请求返回 503

按 `RESPONSE_FLAGS` 分流：

- `NC`：route 指向的 cluster 不存在或尚未 warmed。
- `UH`：cluster 存在，但没有健康 host。
- `UF`：连接 upstream 失败，检查地址、端口、网络策略、TLS。
- `UO`：触发 cluster circuit breaker。
- `URX`：重试达到上限。

配合 Admin 查询：

```bash
curl -s http://localhost:9901/clusters
curl -s 'http://localhost:9901/stats?filter=^cluster\.backend'
```

### 请求返回 504

典型标记：

```text
response_flags=UT
response_code_details=upstream_per_try_timeout
```

检查项：

- route `timeout` 是否过小。
- retry `per_try_timeout` 是否小于后端 P99 延迟。
- 后端是否排队、慢查询、线程池耗尽。
- upstream connection pool 是否存在 pending。

### 路由命中但后端收到错误 Host

Envoy 默认转发原始 Host 或 `:authority`。如果上游要求固定 Host，需要在 route 中配置 host rewrite：

```yaml
route:
  cluster: backend
  host_rewrite_literal: backend.internal
```

如果需要改 path，使用：

```yaml
route:
  cluster: backend
  prefix_rewrite: "/"
```

不要通过普通 header append 机制修改 `:path`、`:authority` 或 `Host`。这些字段需要使用 `prefix_rewrite`、`regex_rewrite`、`host_rewrite_literal` 等专用配置。

## 生产配置基线

推荐基线：

- Admin 只绑定本地地址或受控管理网络。
- 所有 listener 显式配置 access log。
- access log 包含 `%RESPONSE_FLAGS%`、`%RESPONSE_CODE_DETAILS%`、`%UPSTREAM_HOST%`、`%UPSTREAM_CLUSTER%`、`%DURATION%`。
- route 显式配置 `timeout`，不要依赖默认值。
- retry 必须配置 `per_try_timeout` 和最大重试次数。
- cluster 配置 `connect_timeout`、`circuit_breakers`、`outlier_detection`。
- 对关键 upstream 配置主动健康检查。
- DNS cluster 在非 IPv6 环境中配置 `dns_lookup_family: V4_ONLY`。
- 边缘代理开启 downstream TLS，服务间敏感流量使用 upstream TLS 或 mTLS。
- 动态配置使用 xDS 时按 CDS/EDS/LDS/RDS 依赖顺序发布。
- 控制面发布 route 前确认 cluster 和 endpoint 已经存在。

## 参考资料

- Envoy v1.39.0 `Life of a Request`: https://www.envoyproxy.io/docs/envoy/v1.39.0/intro/life_of_a_request
- Envoy Getting Started / Quick start: https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/
- Envoy Admin interface: https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/admin.html
- Envoy HTTP route matching: https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/route_matching
- Envoy xDS configuration API overview: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/dynamic_configuration
- Envoy access logging and substitution formatter: https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage.html
