# XDS模块3

本部分以CDS为例，讲解Sotw协议的源码实现。

相应的架构和配置文件可参考

[XDS模块1](./XDS模块(1).md)

[XDS模块2](./XDS模块(2).md)

## 关键类说明

**GrpcStream**

```c++
/* 声明:*/
template <class RequestProto, class ResponseProto>
class GrpcStream : public Grpc::AsyncStreamCallbacks<ResponseProto>

/* 关键属性:*/

// Grpc异步client，用来创建grpc stream
Grpc::AsyncClient<RequestProto, ResponseProto> async_client_;
// Grpc异步client创建的异步stream
Grpc::AsyncStream<RequestProto> stream_{};
// 异步stream的回调函数，收到control plane response后触发
GrpcStreamCallbacks<ResponseProto>* const callbacks_;
// Grpc Server handler的相关信息
const Protobuf::MethodDescriptor& service_method_;

/* 关键接口: */
// 创建新的Stream，成功后会发起DiscoveryRequest
void establishNewStream()
// 发送消息，消息为DiscoveryRequest或DeltaDiscoveryRequest
void sendMessage(const RequestProto& request)
// Stream收到消息，为DiscoveryResponse或DelataDiscoveryResponse
void onReceiveMessage(ResponseProtoPtr<ResponseProto>&& message)

```

**GrpcMuxImpl**

```c++
/* 声明:*/
class GrpcMuxImpl : public GrpcMux,
                    public GrpcStreamCallbacks<envoy::service::discovery::v3::DiscoveryResponse>,
                    public Logger::Loggable<Logger::Id::config>

/* 关键属性:*/
// 创建的GrpcStream实例，用来通信
GrpcStream<envoy::service::discovery::v3::DiscoveryRequest,
envoy::service::discovery::v3::DiscoveryResponse>
grpc_stream_;
// 请求的api的状态，key为api的类型，如cds、eds等
absl::node_hash_map<std::string, ApiState> api_state_;
// 订阅的资源名称，就是type_urls
std::list<std::string> subscriptions_;
// stream对应的发送队列，先进队列再发送
std::unique_ptr<std::queue<std::string>> request_queue_;

/* 关键接口: */
// 添加新的watch，格式见下图
GrpcMuxWatchPtr addWatch(const std::string& type_url, const std::set<std::string>& resources,
SubscriptionCallbacks& callbacks,
OpaqueResourceDecoder& resource_decoder)

// 处理ControlPlane返回的DiscoveryResponse
void
onDiscoveryResponse(std::unique_ptr<envoy::service::discovery::v3::DiscoveryResponse>&& message,
ControlPlaneStats& control_plane_stats)
//执行DiscoveryRequest的发送，消息缓存在ApiState里
void sendDiscoveryRequest(const std::string& type_url);
//将要发送的消息加入发送队列
void queueDiscoveryRequest(const std::string& queue_item);
```

**ApiState**

```c++
/* 声明:*/
struct ApiState

/* 关键属性:*/
// 这个API上所有的Watch
std::list<GrpcMuxWatchImpl*> watches_;
// 当前的DiscoveryRequest，发送时复用
envoy::service::discovery::v3::DiscoveryRequest request_;

/* 关键接口: */

```

**GrpcSubscriptionImpl**

```c++
/* 声明:*/
class GrpcSubscriptionImpl : public Subscription,
                             SubscriptionCallbacks,
                             Logger::Loggable<Logger::Id::config>

/* 关键属性:*/
//所使用的GrpcMux
GrpcMuxSharedPtr grpc_mux_;
//所使用的资源类型
const std::string type_url_;
//下游的Subscription，为XDSApi的Impl
SubscriptionCallbacks& callbacks_;
//所使用watch
GrpcMuxWatchPtr watch_; 

/* 关键接口: */
//配置更新触发的回调，由全量更新的触发
void onConfigUpdate(const std::vector<Config::DecodedResourceRef>& resources,
const std::string& version_info) override;

//配置更新触发的回调，由增量更新的触发
void onConfigUpdate(const std::vector<Config::DecodedResourceRef>& added_resources,
const Protobuf::RepeatedPtrField<std::string>& removed_resources,
const std::string& system_version_info)

//启动并watch这些资源
void start(const std::set<std::string>& resource_names) ;

//变更watch的资源
void updateResourceInterest(const std::set<std::string>& update_to_these_names)

```

**GrpcMuxWatchImpl**

```c++
/* 声明:*/
struct GrpcMuxWatchImpl : public GrpcMuxWatch

/* 关键属性:*/
//订阅的所有资源的name
std::set<std::string> resources_;
//watch callback函数
SubscriptionCallbacks& callbacks_; 一般为GrpcSubscriptionImpl
//resource反序列化
OpaqueResourceDecoder& resource_decoder_;
//资源类型
const std::string type_url_;
//关联的GrpcMux
GrpcMuxImpl& parent_;
//该type api所有的watches
std::list<GrpcMuxWatchImpl*>& watches_;

/* 关键接口: */
// 更新需要watch的资源
void update(const std::set<std::string>& resources)
```

**CdsApiImpl**

```c++
/* 声明:*/
class CdsApiImpl : public CdsApi,
                   Envoy::Config::SubscriptionBase<envoy::config::cluster::v3::Cluster>,
                   Logger::Loggable<Logger::Id::upstream>

/* 关键属性:*/
//集群管理器，最核心的类之一
ClusterManager& cm_;
//GrpcSubscription实例
Config::SubscriptionPtr subscription_;

/* 关键接口: */
//资源更新回调。added_resources包括新增+变更的。removed_resources为删除的资源
void onConfigUpdate(const std::vector<Config::DecodedResourceRef>& added_resources,
const Protobuf::RepeatedPtrField<std::string>& removed_resources,
const std::string& system_version_info) override;
```

从对象的嵌套的角度看，嵌套关系如下图所示：
![](./images/xds2.png)

即一个GrpcMux维护一个Map，key为type_Url, 例如type.googleapis.com/envoy.config.cluster.v3.Cluster，value为对应的ApiState。而ApiState里维护着WatchList，每个Watch维护一组resource name和对应这些资源变更的回调函数。


## 源码讲解

### cds api初始化

在source/common/upstream/cluster_manager_impl.cc的ClusterManagerImpl::ClusterManagerImpl构造函数中有如下实现

```c++
// We can now potentially create the CDS API once the backing cluster exists.
  if (dyn_resources.has_cds_config()) {
    cds_api_ = factory_.createCds(dyn_resources.cds_config(), *this);
    init_helper_.setCds(cds_api_.get());
  }
```
上述createCds接口的调用时序图如下所示
![](./images/cdsinit.png)

## XDS核心逻辑初始化

本部分讲解Sotw协议变体时，envoy中xds相关的核心实现，也即CdsApiImpl中subscription_的初始化。

subscription_的初始化时刻在CdsApiImpl的构造函数中，其位于source/common/upstream/cds_api_impl.cc，有如下实现

```c++
subscription_ = cm_.subscriptionFactory().subscriptionFromConfigSource(
      cds_config, Grpc::Common::typeUrl(resource_name), *scope_, *this, resource_decoder_);
```

上述cm_.subscriptionFactory().subscriptionFromConfigSource接口会调用 位于source/common/config/subscription_factory_impl.cc中的SubscriptionFactoryImpl::subscriptionFromConfigSource接口，

该函数接口声明如下
```c++
SubscriptionPtr SubscriptionFactoryImpl::subscriptionFromConfigSource(
    const envoy::config::core::v3::ConfigSource& config, absl::string_view type_url,
    Stats::Scope& scope, SubscriptionCallbacks& callbacks,
    OpaqueResourceDecoder& resource_decoder)
```

上述参数说明如下：

**config:** 配置文件中的ConfigSource配置选项

**type_url:** 资源类型，其构造过程在CdsApiImpl构造函数中，具体实现如下

```c++
const auto resource_name = getResourceName();
Grpc::Common::typeUrl(resource_name);
```

**callbacks:** 该callbacks即为CdsApiImpl

回到SubscriptionFactoryImpl::subscriptionFromConfigSource函数，该函数有如下实现，负责初始化相应的**GrpcSubscriptionImpl实例 以及其grpc_mux_成员变量等**

```c++
switch (api_config_source.api_type()) {
    case envoy::config::core::v3::ApiConfigSource::hidden_envoy_deprecated_UNSUPPORTED_REST_LEGACY:
      throw EnvoyException(
          "REST_LEGACY no longer a supported ApiConfigSource. "
          "Please specify an explicit supported api_type in the following config:\n" +
          config.DebugString());
    case envoy::config::core::v3::ApiConfigSource::REST:
      ..........
    // Sotw+grpc
    case envoy::config::core::v3::ApiConfigSource::GRPC:
      return std::make_unique<GrpcSubscriptionImpl>(
          std::make_shared<Config::GrpcMuxImpl>(
              local_info_,
              Utility::factoryForGrpcApiConfigSource(cm_.grpcAsyncClientManager()，api_config_source, scope, true)
                  ->create(),
              dispatcher_, sotwGrpcMethod(type_url, api_config_source.transport_api_version()),
              api_config_source.transport_api_version(), random_, scope,
              Utility::parseRateLimitSettings(api_config_source),
              api_config_source.set_node_on_first_message_only()),
          callbacks, resource_decoder, stats, type_url, dispatcher_,
          Utility::configSourceInitialFetchTimeout(config),
          /*is_aggregated*/ false);
    case envoy::config::core::v3::ApiConfigSource::DELTA_GRPC: ...........
    default:
      NOT_REACHED_GCOVR_EXCL_LINE;
```
上述实现会初始化GrpcSubscriptionImpl实例，GrpcSubscriptionImpl 的构造函数位于source/common/config/grpc_subscription_impl.h，其声明如下所示

```c++
GrpcSubscriptionImpl(GrpcMuxSharedPtr grpc_mux, SubscriptionCallbacks& callbacks,
                       OpaqueResourceDecoder& resource_decoder, SubscriptionStats stats,
                       absl::string_view type_url, Event::Dispatcher& dispatcher,
                       std::chrono::milliseconds init_fetch_timeout, bool is_aggregated);
```

参数说明:

**grpc_mux:** 用来除初始化GrpcSubscriptionImpl的grpc_mux_成员

**callbacks:** 也即CdsApiImpl

grpc_mux_成员变量的初始化语句为：

```c++
std::make_shared<Config::GrpcMuxImpl>(
              local_info_,
              Utility::factoryForGrpcApiConfigSource(cm_.grpcAsyncClientManager(),
                api_config_source, scope, true)
                  ->create(),
              dispatcher_, sotwGrpcMethod(type_url,api_config_source.transport_api_version())
```

Config::GrpcMuxImpl的构造函数位于source/common/config/grpc_mux_impl.h，其声明如下

```c++
GrpcMuxImpl(const LocalInfo::LocalInfo& local_info, Grpc::RawAsyncClientPtr async_client,
              Event::Dispatcher& dispatcher, const Protobuf::MethodDescriptor& service_method,
              envoy::config::core::v3::ApiVersion transport_api_version,
              Runtime::RandomGenerator& random, Stats::Scope& scope,
              const RateLimitSettings& rate_limit_settings, bool skip_subsequent_node);
```

参数说明：

**async_client:** 主要用来初始化其成员grpc_stream_

async_client的初始化语句为：
```c++
Utility::factoryForGrpcApiConfigSource(cm_.grpcAsyncClientManager(),api_config_source, scope, true)
                  ->create()
```
该接口行为如下
- 调用位于source/common/grpc/async_client_manager_impl.cc中的AsyncClientManagerImpl::factoryForGrpcService接口，生成相应的AsyncClientFactoryImpl实例，

- 调用AsyncClientFactoryImpl::create接口，创建位于source/common/grpc/async_client_impl.h的 AsyncClientImpl实例

**service_method:** Protobuf::MethodDescriptor类，主要用来构造grpc的http2请求，具体可参考[grpc over http2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)

该参数的初始化语句为
```c++
sotwGrpcMethod(type_url, api_config_source.transport_api_version())
```
上述接口位于source/common/config/type_to_endpoint.cc，其实现如下
```c++
const Protobuf::MethodDescriptor&
sotwGrpcMethod(absl::string_view type_url,
               envoy::config::core::v3::ApiVersion transport_api_version) {
  const auto it = typeUrlToVersionedServiceMap().find(static_cast<TypeUrl>(type_url));
  ASSERT(it != typeUrlToVersionedServiceMap().cend());
  return *Protobuf::DescriptorPool::generated_pool()->FindMethodByName(
      it->second.sotw_grpc_.methods_[effectiveTransportApiVersion(transport_api_version)]);
}
```
在typeUrlToVersionedServiceMap()接口中会将所有的xDS method的name进行注册，具体注册接口流程可参见
```c++
TypeUrlToVersionedServiceMap* buildTypeUrlToServiceMap()
```
Protobuf::DescriptorPool::generated_pool()，该接口的功能可参考[generated_pool](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.descriptor#DescriptorPool.generated_pool.details)

[FindMethodByName](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.descriptor)


回到GrpcMuxImpl初始化, 该实例的构造函数较短，故将其实现粘贴如下

```c++
GrpcMuxImpl::GrpcMuxImpl(const LocalInfo::LocalInfo& local_info,
                         Grpc::RawAsyncClientPtr async_client, Event::Dispatcher& dispatcher,
                         const Protobuf::MethodDescriptor& service_method,
                         envoy::config::core::v3::ApiVersion transport_api_version,
                         Runtime::RandomGenerator& random, Stats::Scope& scope,
                         const RateLimitSettings& rate_limit_settings, bool skip_subsequent_node)
    : grpc_stream_(this, std::move(async_client), service_method, random, dispatcher, scope,
                   rate_limit_settings),
      local_info_(local_info), skip_subsequent_node_(skip_subsequent_node),
      first_stream_request_(true), transport_api_version_(transport_api_version) {
  Config::Utility::checkLocalInfo("ads", local_info);
}
```

上述部分说明如下

**skip_subsequent_node_:** [xDS REST and gRPC protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#basic-protocol-overview) 在**Basic Protocol Overview**部分有如下描述

> Only the first request on a stream is guaranteed to
> carry the node identifier. The subsequent discovery
> requests on the same stream may carry an empty node 
> identifier. This holds true regardless of the
> acceptance of the discovery responses on the same 
> stream. The node identifier should always be identical  
> if present more than once on the stream. It is 
> sufficient to only check the first message for the 
> node identifier as a result.

该变量便是用来保证后续XDS request不携带node id标识符。

**transport_api_version_:** 该变量便是[xDS REST and gRPC protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#basic-protocol-overview)中**Transport API version**，该变量表示 资源类型的版本，譬如资源类型是否是V3，V2甚至V4等。

**grpc_stream_:** 其同XDS server通信，发送和接收相应的 DiscoveryRequest 和 DiscoveryResponse. 

grpc_stream_的构造函数位于source/common/config/grpc_stream.h，其构造函数声明为

```c++
GrpcStream(GrpcStreamCallbacks<ResponseProto>* callbacks, Grpc::RawAsyncClientPtr async_client,
             const Protobuf::MethodDescriptor& service_method, Runtime::RandomGenerator& random,
             Event::Dispatcher& dispatcher, Stats::Scope& scope,
             const RateLimitSettings& rate_limit_settings)
```

在GrpcStream的构造函数中会初始化其成员变量async_client_，该变量的声明如下

```c++
Grpc::AsyncClient<RequestProto, ResponseProto> async_client_;
```
其构造函数声明位于source/common/grpc/typed_async_client.h，定义如下

```c++
AsyncClient() = default;
AsyncClient(RawAsyncClientPtr&& client) : client_(std::move(client)) {}
```

至此初步将CDS初始化流程讲解完毕。

其初始化过程汇总如下

![](./images/xds4.png)

## 请求流程

本文以CDS为例讲解Sotw协议在envoy中的实现，上述讲解了CDS的初始化流程，本部分主要讲解CDS如何触发grpc stream向XDS server发送请求以及解析服务。

在source/common/upstream/cluster_manager_impl.cc文件的ClusterManagerInitHelper::maybeFinishInitialize()函数中有如下代码实现

```c++
if (state_ == State::WaitingToStartSecondaryInitialization && cds_) {
    ENVOY_LOG(info, "cm init: initializing cds");
    state_ = State::WaitingToStartCdsInitialization;
    cds_->initialize();
  } 
```

关于Cluster初始化不是本文讲解的主题，因此跳过。上述代码中会调用

```c++
cds_->initialize();
```
其实现位于source/common/upstream/cds_api_impl.h，实现如下

```c++
void initialize() override { subscription_->start({}); }
```

上述会调用位于source/common/config/grpc_subscription_impl.cc中的GrpcSubscriptionImpl::start接口，其实现如下所示

```c++
void GrpcSubscriptionImpl::start(const std::set<std::string>& resources) {
  // ..........
  watch_ = grpc_mux_->addWatch(type_url_, resources, *this, resource_decoder_);
  // .....
  // ADS initial request batching relies on the users of the GrpcMux *not* calling start on it,
  // whereas non-ADS xDS users must call it themselves.
  if (!is_aggregated_) {
    grpc_mux_->start();
  }
}
```

上述实现中，首先调用GrpcMuxImpl的addWatch接口，用来初始化GrpcSubscriptionImpl的watch_成员变量。

watch_成员是一个GrpcMuxWatchImpl实例，其实例化过程在addWatch接口中，其具体实现如下

```c++

GrpcMuxWatchPtr GrpcMuxImpl::addWatch(const std::string& type_url,
                                      const std::set<std::string>& resources,
                                      SubscriptionCallbacks& callbacks,
                                      OpaqueResourceDecoder& resource_decoder) {
  auto watch =
      std::make_unique<GrpcMuxWatchImpl>(resources, callbacks, resource_decoder, type_url, *this);
  ENVOY_LOG(debug, "gRPC mux addWatch for " + type_url);

  // Lazily kick off the requests based on first subscription. This has the
  // convenient side-effect that we order messages on the channel based on
  // Envoy's internal dependency ordering.
  // TODO(gsagula): move TokenBucketImpl params to a config.
  if (!api_state_[type_url].subscribed_) {
    api_state_[type_url].request_.set_type_url(type_url);
    api_state_[type_url].request_.mutable_node()->MergeFrom(local_info_.node());
    api_state_[type_url].subscribed_ = true;
    subscriptions_.emplace_back(type_url);
  }

  // This will send an updated request on each subscription.
  // TODO(htuch): For RDS/EDS, this will generate a new DiscoveryRequest on each resource we added.
  // Consider in the future adding some kind of collation/batching during CDS/LDS updates so that we
  // only send a single RDS/EDS update after the CDS/LDS update.
  queueDiscoveryRequest(type_url);

  return watch;
}

```

上述函数做了如下工作

- 创建GrpcMuxWatchImpl

- 初始化api_state_ map,其key为type_url

- 将request入队列

先具体看一下创建GrpcMuxWatchImpl的过程，其构造函数如下

```c++
GrpcMuxWatchImpl(const std::set<std::string>& resources, SubscriptionCallbacks& callbacks,
                     OpaqueResourceDecoder& resource_decoder, const std::string& type_url,
                     GrpcMuxImpl& parent)
        : resources_(resources), callbacks_(callbacks), resource_decoder_(resource_decoder),
          type_url_(type_url), parent_(parent), watches_(parent.api_state_[type_url].watches_) {
      // 将this，也即本Watch插入到api_state_相应的列表中
      watches_.emplace(watches_.begin(), this);
    }
```

上述watches_成员变量为api_states_中相应资源对应的watches_的引用，然后在将该GrpcMuxWatchImpl插入到订阅相应资源的watches_列表中。

初始化时，resources_为空，通过[xDS REST and gRPC protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#basic-protocol-overview)可知，

> For example, in SotW:
> Client sends a request with resource_names unset. Server interprets this as a subscription to *.

queueDiscoveryRequest接口的实现如下

```c++
void GrpcMuxImpl::queueDiscoveryRequest(const std::string& queue_item) {
  request_queue_.push(queue_item);
  drainRequests();
}
```

drainRequests接口实现如下

```c++
void GrpcMuxImpl::drainRequests() {
  while (!request_queue_.empty() && grpc_stream_.checkRateLimitAllowsDrain()) {
    // Process the request, if rate limiting is not enabled at all or if it is under rate limit.
    sendDiscoveryRequest(request_queue_.front());
    request_queue_.pop();
  }
  grpc_stream_.maybeUpdateQueueSizeStat(request_queue_.size());
}
```

由于此时grpc_stream_还未和server建立连接，故此时request还在队列中。

回到GrpcSubscriptionImpl::start接口中，在创建watch_成功后，会调用grpc_mux_的start接口，也即

```c++
void GrpcMuxImpl::start() { grpc_stream_.establishNewStream(); }
```

上述接口会调用grpc_stream_的establishNewStream()创建和server的连接，其实现如下

```c++
void establishNewStream() {
    // .......
    stream_ = async_client_->start(service_method_, *this, Http::AsyncClient::StreamOptions());
    // ....
    callbacks_->onStreamEstablished();
  }
```

关于async_client_创建stream_成员变量的过程本文不再讲解，其过程同http2 filter有些雷同，后续在讲解http2 filter时，再来讲解这一块。

在创建stream_成功后，也即此时同server连接成功，会调用GrpcMuxImpl::onStreamEstablished()接口，其实现如下

```c++
void GrpcMuxImpl::onStreamEstablished() {
  first_stream_request_ = true;
  for (const auto& type_url : subscriptions_) {
    queueDiscoveryRequest(type_url);
  }
}

```

上述会将订阅的资源request依次通过grpc stream发送到server端。

上述queueDiscoveryRequest接口 最终会调用GrpcMuxImpl::sendDiscoveryRequest接口，其实现如下

```c++
void GrpcMuxImpl::sendDiscoveryRequest(const std::string& type_url) {
  // ......
  ApiState& api_state = api_state_[type_url];
  // ....

  auto& request = api_state.request_;
  request.mutable_resource_names()->Clear();

  // Maintain a set to avoid dupes.
  std::unordered_set<std::string> resources;
  for (const auto* watch : api_state.watches_) {
    for (const std::string& resource : watch->resources_) {
      if (resources.count(resource) == 0) {
        resources.emplace(resource);
        request.add_resource_names(resource);
      }
    }
  }

  if (skip_subsequent_node_ && !first_stream_request_) {
    request.clear_node();
  }
  VersionConverter::prepareMessageForGrpcWire(request, transport_api_version_);
  ENVOY_LOG(trace, "Sending DiscoveryRequest for {}: {}", type_url, request.DebugString());
  grpc_stream_.sendMessage(request);
  first_stream_request_ = false;

  // clear error_detail after the request is sent if it exists.
  if (api_state_[type_url].request_.has_error_detail()) {
    api_state_[type_url].request_.clear_error_detail();
  }
}

```

该函数具体实现

- 获得api_state_ map中相应type_url（资源类型)的对应watch列表

- 将所有订阅的资源都塞到同一个request中

- 只有第一个request携带node id，后续request均不带node id

- 将transport_api_version_合并到request中

- 调用grpc_stream_的sendMessage接口发送request

- 清空相应的error_detail的内容

至此，将CDS订阅资源的请求流程梳理完毕。

## 资源更新

在讲解资源更新源码之前先看一下[xDS REST and gRPC protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#basic-protocol-overview)中关于sotw相关协议的描述

```c++
Each xDS stream begins with a DiscoveryRequest from the client, which specifies the list of resources to subscribe to, the type URL corresponding to the subscribed resources, the node identifier, and an optional resource type instance version indicating the most recent version of the resource type that the client has already seen (see ACK/NACK and resource type instance version for details).

The server will then send a DiscoveryResponse containing any resources that the client has subscribed to that have changed since the last resource type instance version that the client indicated it has seen. The server may send additional responses at any time when the subscribed resources change.

Whenever the client receives a new response, it will send another request indicating whether or not the resources in the response were valid (see ACK/NACK and resource type instance version for details).

All server responses will contain a nonce, and all subsequent requests from the client must set the response_nonce field to the most recent nonce received from the server on that stream. This allows servers to determine which response a given request is associated with, which avoids various race conditions in the SotW protocol variants. Note that the nonce is valid only in the context of an individual xDS stream; it does not survive stream restarts.

Only the first request on a stream is guaranteed to carry the node identifier. The subsequent discovery requests on the same stream may carry an empty node identifier. This holds true regardless of the acceptance of the discovery responses on the same stream. The node identifier should always be identical if present more than once on the stream. It is sufficient to only check the first message for the node identifier as a result.

Every xDS resource type has a version string that indicates the version for that resource type. Whenever one resource of that type changes, the version is changed.

In a response sent by the xDS server, the version_info field indicates the current version for that resource type. The client then sends another request to the server with the version_info field indicating the most recent valid version seen by the client. This provides a way for the server to determine when it sends a version that the client considers invalid.
```

总结来说：

- 每个XDS grpc stream启动时均会发送DiscoveryRequest给server端，其DiscoveryRequest中包含了所订阅的资源名和资源类型

- server端回复的DiscoveryResponse给client端，该response中包含client端所感兴趣的所有资源

- 每当client端收到DiscoveryResponse时，client便会回复server端，且需要将request的reponse_nonce设置为DiscoveryResponse的nonce

文档[xDS REST and gRPC protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#basic-protocol-overview)给出了EDS 请求的例子，其请求设置如下

```c++
version_info:
node: { id: envoy }
resource_names:
- foo
- bar
type_url: type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
response_nonce:
```

相应的回复如下所示

```c++
version_info: X
resources:
- foo ClusterLoadAssignment proto encoding
- bar ClusterLoadAssignment proto encoding
type_url: type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
nonce: A
```

为了偷懒，我直接根据EDS的request和response等交互图来说明CDS相关的源码分析。

解析server端response的入口可从位于source/common/grpc/async_client_impl.h的AsyncStreamImpl::onData接口开始，

该接口有如下实现

```c++
if (!callbacks_.onReceiveMessageRaw(frame.data_ ? std::move(frame.data_)
        : std::make_unique<Buffer::OwnedImpl>())) {
      streamError(Status::WellKnownGrpcStatus::Internal);
      return;
    }
```

上述接口最终调用位于source/common/config/grpc_stream.h的onReceiveMessage(std::unique_ptr<ResponseProto>&& message)接口，该函数实现如下

```c++
void onReceiveMessage(std::unique_ptr<ResponseProto>&& message) override {
    // Reset here so that it starts with fresh backoff interval on next disconnect.
    backoff_strategy_->reset();
    // Sometimes during hot restarts this stat's value becomes inconsistent and will continue to
    // have 0 until it is reconnected. Setting here ensures that it is consistent with the state of
    // management server connection.
    control_plane_stats_.connected_state_.set(1);
    callbacks_->onDiscoveryResponse(std::move(message), control_plane_stats_);
  }
```

其会调用位于source/common/config/grpc_mux_impl.cc的GrpcMuxImpl::onDiscoveryResponse，该函数是Sotw协议的关键实现，本文会着重分析，其实现如下

```c++
void GrpcMuxImpl::onDiscoveryResponse(
    std::unique_ptr<envoy::service::discovery::v3::DiscoveryResponse>&& message,
    ControlPlaneStats& control_plane_stats) {
  const std::string& type_url = message->type_url();
  // .........
  if (api_state_.count(type_url) == 0) { // 1
    ENVOY_LOG(warn, "Ignoring the message for type URL {} as it has no current subscribers.",
              type_url);
    // TODO(yuval-k): This should never happen. consider dropping the stream as this is a
    // protocol violation
    return;
  }
  if (api_state_[type_url].watches_.empty()) { // 2
    // update the nonce as we are processing this response.
    api_state_[type_url].request_.set_response_nonce (message->nonce()); // 3
    if (message->resources().empty()) { // 4
      // No watches and no resources. This can happen when envoy unregisters from a
      // resource that's removed from the server as well. For example, a deleted cluster
      // triggers un-watching the ClusterLoadAssignment watch, and at the same time the
      // xDS server sends an empty list of ClusterLoadAssignment resources. we'll accept
      // this update. no need to send a discovery request, as we don't watch for anything.
      api_state_[type_url].request_.set_version_info (message->version_info()); // 5
    } else {
      // No watches and we have resources - this should not happen. send a NACK (by not
      // updating the version).
      ENVOY_LOG(warn, "Ignoring unwatched type URL {}", type_url);
      queueDiscoveryRequest(type_url);
    }
    return;
  }
  try {
    // To avoid O(n^2) explosion (e.g. when we have 1000s of EDS watches), we
    // build a map here from resource name to resource and then walk watches_.
    // We have to walk all watches (and need an efficient map as a result) to
    // ensure we deliver empty config updates when a resource is dropped. We make the map ordered
    // for test determinism.
    std::vector<DecodedResourceImplPtr> resources;
    absl::btree_map<std::string, DecodedResourceRef> resource_ref_map;
    std::vector<DecodedResourceRef> all_resource_refs;
    OpaqueResourceDecoder& resource_decoder =
        api_state_[type_url].watches_.front()->resource_decoder_;
    // 6
    for (const auto& resource : message->resources()) {
      if (type_url != resource.type_url()) {
        throw EnvoyException(
            fmt::format("{} does not match the message-wide type URL {} in DiscoveryResponse {}",
                        resource.type_url(), type_url, message->DebugString()));
      }
      resources.emplace_back(
          new DecodedResourceImpl(resource_decoder, resource, message->version_info()));
      all_resource_refs.emplace_back(*resources.back());
      resource_ref_map.emplace(resources.back()->name(), *resources.back());
    }
    // 7
    for (auto watch : api_state_[type_url].watches_) {
      // onConfigUpdate should be called in all cases for single watch xDS (Cluster and
      // Listener) even if the message does not have resources so that update_empty stat
      // is properly incremented and state-of-the-world semantics are maintained.
      if (watch->resources_.empty()) {
        watch->callbacks_.onConfigUpdate(all_resource_refs, message->version_info()); // 8
        continue;
      }
      std::vector<DecodedResourceRef> found_resources;
      for (const auto& watched_resource_name : watch->resources_) {
        auto it = resource_ref_map.find(watched_resource_name);
        if (it != resource_ref_map.end()) {
          found_resources.emplace_back(it->second);
        }
      }
      // onConfigUpdate should be called only on watches(clusters/routes) that have
      // updates in the message for EDS/RDS.
      if (!found_resources.empty()) {
        watch->callbacks_.onConfigUpdate(found_resources, message->version_info()); // 9
      }
    }
    // TODO(mattklein123): In the future if we start tracking per-resource versions, we
    // would do that tracking here.
    api_state_[type_url].request_.set_version_info(message->version_info()); // 10
    Memory::Utils::tryShrinkHeap();
  } catch (const EnvoyException& e) {
    for (auto watch : api_state_[type_url].watches_) {
      watch->callbacks_.onConfigUpdateFailed(
          Envoy::Config::ConfigUpdateFailureReason::UpdateRejected, &e);
    }
    ::google::rpc::Status* error_detail = api_state_[type_url].request_.mutable_error_detail(); // 11
    error_detail->set_code(Grpc::Status::WellKnownGrpcStatus::Internal);
    error_detail->set_message(Config::Utility::truncateGrpcStatusMessage(e.what()));
  }
  api_state_[type_url].request_.set_response_nonce(message->nonce()); // 12
  queueDiscoveryRequest(type_url); // 13
}
```

上述1-5中所描述的场景本文不讲(主要由于本人也不是主要搞XDS这一块，所以对于边缘场景理解可能不够通透)

从6开始讲起，

6: 此处的for循环主要是将response中的resource收集到一个map中

7: 此处的for循环主要是通知相应的CdsApi更新相应的资源

8: cds在一开始订阅时，资源列表为空，因此server端会返回所有资源，此时均会被client更新，上述
```c++
watch->callbacks_.onConfigUpdate(all_resource_refs, message->version_info());
```
会调用位于source/common/config/grpc_subscription_impl.cc的
```c++
GrpcSubscriptionImpl::onConfigUpdate(const std::vector<Config::DecodedResourceRef>& resources,
                                          const std::string& version_info)
```
接口

其会调用位于source/common/upstream/cds_api_impl.cc的CdsApiImpl::onConfigUpdate接口，该接口实现如下

```c++
void CdsApiImpl::onConfigUpdate(const std::vector<Config::DecodedResourceRef>& resources,
                                const std::string& version_info) {
  ClusterManager::ClusterInfoMap clusters_to_remove = cm_.clusters();
  std::vector<envoy::config::cluster::v3::Cluster> clusters;
  for (const auto& resource : resources) {
    clusters_to_remove.erase(resource.get().name());
  }
  Protobuf::RepeatedPtrField<std::string> to_remove_repeated;
  for (const auto& cluster : clusters_to_remove) {
    *to_remove_repeated.Add() = cluster.first;
  }
  onConfigUpdate(resources, to_remove_repeated, version_info);
}
```

上述会进行如下工作

- 将 已经存在于resource中的cluster的从clusters_to_remove中移除

- 调整相应的cluster，并调用onConfigUpdate(resources, to_remove_repeated, version_info)接口

上述接口最终调用
```c++
CdsApiImpl::onConfigUpdate(const std::vector<Config::DecodedResourceRef>& added_resources,
                                const Protobuf::RepeatedPtrField<std::string>& removed_resources,
                                const std::string& system_version_info)
```
在该接口中有如下实现
```c++
if (cm_.addOrUpdateCluster(cluster, resource.get().version())) {
        any_applied = true;
        ENVOY_LOG(info, "cds: add/update cluster '{}'", cluster.name());
      } 
```
最终调用cm_.addOrUpdateCluster更新相应的cluster，关于该接口的讲解，会放到cluster manager初始化部分。

至此，完成相应的资源更新。

10: 更新成功后 需要将相应资源的request的version_info设置为server端response中的version_info

11: 更新成功后需要将相应资源的request的reponse_nonce设置为response的nonce

这也是文档[xDS REST and gRPC protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#basic-protocol-overview) ACK/NACK讲解的实现。

譬如在文档中，关于ACK有如下所示

![](./images/xds5.png)

关于NACK有如下所示

![](./images/xds6.png)

通过和源码讲解结合便可以加深理解

## 总结

至此，讲解完了XDS Sotw协议在envoy中的实现，该部分讲解主要分为三部分，分别为

[XDS模块1](./XDS模块(1).md)

[XDS模块2](./XDS模块(2).md)

以及本篇文章。

后续会讲解ADS以及Incremental xDS在envoy的实现。


