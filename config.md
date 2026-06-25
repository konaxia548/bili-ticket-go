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
