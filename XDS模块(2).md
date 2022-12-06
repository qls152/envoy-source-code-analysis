# XDS模块2

本部分主要介绍XDS模块相关的配置文件。

## 配置文件说明

**全静态(无xDS)**

```c++

static_resources:
  listeners:
  - name: redis_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 3999
    filter_chains:
    - filters:
      - name: envoy.redis_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.redis_proxy.v3.RedisProxy
          stat_prefix: egress_redis
          settings:
            op_timeout: 5s
          prefix_routes:
            catch_all_route:
              cluster: redis_cluster
  clusters:
  - name: redis_cluster
    connect_timeout: 1s
    type: strict_dns # static
    lb_policy: MAGLEV
    load_assignment:
      cluster_name: redis_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: redis_server
                port_value: 6399
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9001



```

**EDS + 静态**

```c++
static_resources:
  listeners:
  - name: redis_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 3999
    filter_chains:
    - filters:
      - name: envoy.filters.network.redis_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.redis_proxy.v3.RedisProxy
          stat_prefix: egress_redis
          settings:
            op_timeout: 5s
          prefix_routes:
            routes:
              - prefix: "qa"
                cluster: "cluster_q"
  clusters:
  - name: cluster_q
    connect_timeout: 1s
    type: EDS # static
    lb_policy: MAGLEV
    eds_cluster_config:
      service_name: cluster_qq
      eds_config:
        api_config_source:
          api_type: GRPC
          grpc_services:
            - envoy_grpc:
                cluster_name: xds_cluster
  - name: xds_cluster
    connect_timeout: 0.25s
    type: STATIC_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: xds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 38000
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9001
```

**全动态**

```c++
dynamic_resources:
  cds_config:
    resource_api_version: V3
    api_config_source:
      api_type: GRPC
      transport_api_version: V3
      grpc_services:
      - envoy_grpc:
          cluster_name: xds_cluster
      set_node_on_first_message_only: true
  lds_config:
    resource_api_version: V3
    api_config_source:
      api_type: GRPC
      transport_api_version: V3
      grpc_services:
      - envoy_grpc:
          cluster_name: xds_cluster
      set_node_on_first_message_only: true
```

**ADS全动态**

```c++
ads_config:
    api_type: GRPC
    transport_api_version: V3
    grpc_services:
    - envoy_grpc:
        cluster_name: xds_cluster
    set_node_on_first_message_only: true
  cds_config:
    resource_api_version: V3
    ads: {}
  lds_config:
    resource_api_version: V3
    ads: {}
```

## 客户端配置--协议变体选择

Envoy的 bootstrap配置文件中有两种[ConfigSource](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#envoy-v3-api-msg-config-core-v3-configsource)，一是指对Listener，一是针对Cluster而设置。

而上述小节知，ConfigSource配置中的ApiConfigSource则确定了xDS走的是Sotw协议变体。

具体的ApiConfigSource定义可参考[ApiConfigSource](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#envoy-v3-api-msg-config-core-v3-apiconfigsource)

## 总结

本部分主要讲述了xDS模块所涉及的配置文件的含义，Envoy中相应的源码实现，会根据不同的配置，选择不同的xDS Api。
