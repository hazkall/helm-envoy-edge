# Envoy Helm Chart

Helm chart to deploy a Envoy Proxy Edge

## Install Helm repo

```bash

helm repo add envoy-edge https://ativy-digital.github.io/helm-proxy-envoy-edge/

helm repo update

helm install RELEASE fluentd-loki/fluentd-loki -n envoy

```

## Envoy configuration

Oficial reference: <https://www.envoyproxy.io/docs/envoy/v1.21.0/configuration/best_practices/edge.html?highlight=lb_endpoints>


This default helm chart has a sample of proxy LoadBalance GRPC -> HTTP 2/0 REST

Just change this fields on sample

 <<<<<< INSERT_CLUSTER_NAME >>>>>> -> <https://www.envoyproxy.io/docs/envoy/v1.21.0/api-v3/config/cluster/v3/cluster.proto.html?highlight=cluster>

 <<<<<< URL_DOMAIN_UPSTREAM >>>>>> -> <https://www.envoyproxy.io/docs/envoy/v1.21.0/api-v3/config/core/v3/address.proto#envoy-v3-api-msg-config-core-v3-socketaddress>

```yaml

  envoy.yaml: |-

    overload_manager:
      refresh_interval: 0.25s
      resource_monitors:
      - name: "envoy.resource_monitors.fixed_heap"
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.resource_monitors.fixed_heap.v3.FixedHeapConfig
          max_heap_size_bytes: 2147483648  # 2 GiB
      actions:
      - name: "envoy.overload_actions.shrink_heap"
        triggers:
        - name: "envoy.resource_monitors.fixed_heap"
          threshold:
            value: 0.95
      - name: "envoy.overload_actions.stop_accepting_requests"
        triggers:
        - name: "envoy.resource_monitors.fixed_heap"
          threshold:
            value: 0.98

    admin:
      access_log_path: /tmp/admin_access.log
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 9901

    static_resources:
      listeners:
      - name: listener_01
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 10000
        per_connection_buffer_limit_bytes: 32768

        filter_chains:
        - filters:
          - name: envoy.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              codec_type: auto
              access_log:
              - name: envoy.access_loggers.file
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
                  path: /dev/stdout
                  log_format:
                    text_format: |
                      [%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%"
                      %RESPONSE_CODE% %RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION%
                      %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-FORWARDED-FOR)%" "%REQ(USER-AGENT)%"
                      "%REQ(X-REQUEST-ID)%" "%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%" "%UPSTREAM_TRANSPORT_FAILURE_REASON%"\n
              - name: envoy.access_loggers.file
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
                  path: /dev/stdout
              route_config:
                name: local_route
                virtual_hosts:
                - name: local_service
                  domains: ["*"]
                  routes:
                  - match: { prefix: "/" }
                    route: { cluster: <<<<<< INSERT_CLUSTER_NAME >>>>>>, host_rewrite_literal: <<<<<< URL_DOMAIN_UPSTREAM >>>>>> }
              http_filters:
              - name: envoy.filters.http.router

      clusters:
      - name: <<<<<< INSERT_CLUSTER_NAME >>>>>>
        per_connection_buffer_limit_bytes: 32768
        type: logical_dns
        http2_protocol_options: {}
        dns_lookup_family: V4_ONLY
        lb_policy: round_robin
        load_assignment:
          cluster_name: <<<<<< INSERT_CLUSTER_NAME >>>>>>
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: <<<<<< URL_DOMAIN_UPSTREAM >>>>>>
                    port_value: <<<<<< PORT_DOMAIN_UPSTREAM >>>>>>
        transport_socket:
          name: envoy.transport_sockets.tls
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
            sni: <<<<<< URL_DOMAIN_UPSTREAM >>>>>>
            common_tls_context:
              alpn_protocols: ["h2"]

```

## Configuration Examples

<https://www.envoyproxy.io/docs/envoy/v1.21.0/configuration/overview/examples>
