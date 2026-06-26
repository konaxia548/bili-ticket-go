# 配置文件使用文档

所有放在配置文件中的，修改都可能造成不确定的结果，请谨慎修改。

如需监听0.0.0.0，请阅读本文档的Web部分。

## 加载规则

- 程序只读取当前工作目录下的 `config.yml`。
- 所有字段都是可选字段。程序先使用内置默认值。


## 运行时开关

```yaml
force_hot_project: false
```

- `force_hot_project`: 是否强制所有项目按 hot project 流程处理。默认 `false`。为 `false` 时跟随接口返回的 `hotProject`。



## 网络

```yaml
network:
  proxy_url: ""
  proxy_deduplicate_egress: true
  proxy_isolate_on_412: true
  proxy_412_isolation_ms: 60000
  proxy_timeout_ms: 3000
  proxy_timeout_retries: 2
  proxy_timeout_isolation_ms: 60000
```

- `proxy_url`: 下单接口使用的代理轮询列表。支持 `local`、`http`、`https`、`socks5`，多个入口用英文逗号分隔。
- `local` 表示本机直连。
- `proxy_url` 为空时，下单接口默认使用 `local`。
- `proxy_url` 非空时，只使用配置中明确列出的入口。未配置 `local` 时不会自动添加或回退到 `local`。
- 显式配置的入口全部检查失败时，会返回无可用代理错误，不会静默改成本机直连。
- 配置了 `proxy_url` 时，终端会输出 `发现了 N 个代理配置，使用 M 个。`。`N` 为解析去重后的配置数，`M` 为健康检查和出口去重后保留的入口数。
- `proxy_deduplicate_egress`: 是否剔除公网出口 IP 相同的后续入口，默认 `true`。设为 `false` 时，即使多个入口出口相同，也全部保留参与轮询。
- `proxy_isolate_on_412`: HTTP 412 时是否隔离当前代理，默认 `true`。
- `proxy_412_isolation_ms`: 412 隔离时间，默认 60000 毫秒。
- `proxy_timeout_ms`: 单次代理请求超时，默认 3000 毫秒。
- `proxy_timeout_retries`: 超时后的额外重试次数，默认 2 次。
- `proxy_timeout_isolation_ms`: 所有超时重试耗尽后的隔离时间，默认 60000 毫秒。

## DNS

```yaml
dns:
  mode: httpdns
  ip: ""
```

- `mode`:
  - `system`: 使用系统 DNS。
  - `httpdns`: 使用程序内置HttpDns探测返回 IP。
  - `ip`: 使用 `dns.ip` 指定的 IP。
- `mode: ip` 时 `ip` 必填，可用逗号分隔多个 IP。

## 定时

```yaml
timing:
  use_ntp: false
  ntp_servers:
    - time1.cloud.tencent.com
    - time2.cloud.tencent.com
    - time3.cloud.tencent.com
    - time4.cloud.tencent.com
    - time5.cloud.tencent.com
  ntp_timeout_ms: 1500
  timed_request_offset_ms: 0
  timed_project_refresh_interval_ms: 30000
```

- `use_ntp`: 定时 rush / waiting 是否使用 NTP 校准时间。
- `ntp_servers`: 可以是 YAML 数组，也可以是逗号分隔字符串。
- `timed_request_offset_ms`: 仅定时模式生效。负数表示提前发起首个请求，正数表示延后。
- `timed_project_refresh_interval_ms`: 定时开始前项目信息刷新间隔。小于 0 时回退默认值。


## Web

```yaml
web:
  listen_host: 127.0.0.1
  listen_port: 18080
  auth_token: ""
```

- `web.listen_port`: 范围为 1..65535。
- 当 `web.listen_host` 不是 loopback 地址时，`web.auth_token` 必填。

## 错误码处理和延迟

`error_policy` 控制抢票循环的请求节奏、单次模式重试、HTTP 状态码退避、业务错误码退避和业务错误码分类。所有时间字段单位都是毫秒。

```yaml
error_policy:
  loop:
    rush_interval_ms: 300
    max_create_attempts: 20
    max_rounds: 0

  http:
    default_backoff_ms: 300
    status_backoff_ms:
      "429": 1200

  code_backoff_ms:
    "100009": 1500

  code_class_override:
    "101006": reprepare
```

### loop 节奏

- `rush_interval_ms`: rush/waiting 下单轮次之间的等待时间。每轮失败且允许进入下一轮时使用。
- `max_create_attempts`: 每轮最多执行多少次下单接口。
- `max_rounds`: 最大轮数；`0` 表示不限轮数，直到成功、失败、取消或上下文结束。

### HTTP 状态码延迟

- `http.default_backoff_ms`: 可重试 HTTP 错误的默认退避。网络瞬时错误也使用这个值。
- `http.status_backoff_ms`: 覆盖指定 HTTP 状态码的退避时间。key 必须加引号，状态码范围为 100..599。
- HTTP 412 默认最小退避是 10 秒；如果在 `status_backoff_ms` 中显式配置 `"412"`，则使用配置值。
- 如果响应头里有 `Retry-After` 或 `X-Bili-Retry-After`，程序会取它和配置退避中的较大值。

示例：让 429 慢一点，412 保持更保守：

```yaml
error_policy:
  http:
    default_backoff_ms: 300
    status_backoff_ms:
      "429": 1200
      "412": 10000
```

### 业务错误码延迟

`code_backoff_ms` 用于指定业务错误码的退避时间。key 必须加引号，值不能为负数。

```yaml
error_policy:
  code_backoff_ms:
    "100009": 1500
    "211": 1200
    "3": 5000
```

对普通可重试业务码，显式 `code_backoff_ms` 会覆盖接口返回的 retry-after。对库存退避类错误码，未显式配置时会取接口 retry-after 和默认库存退避中的较大值。

### 业务错误码分类

`code_class_override` 用于改变业务错误码的处理方式。支持的分类：

- `limit`: 限流或临时限制，按可重试错误处理。
- `reprepare`: 当前 prepare token 或订单上下文不可继续，结束本轮并重新 prepare。
- `fatal`: 直接视为致命错误。配置为 `fatal` 的错误码不能同时配置 `code_backoff_ms`。
- `stock_backoff`: 库存瞬时不足。waiting 模式会返回监控；rush 模式会按退避继续后续尝试/轮次。
- `buyer_error`: 购票人、实名、地址等用户侧错误，直接失败并提示。
- `confirm_rescue`: createV2/confirm/status 阶段需要一次 confirmInfo 补救。

