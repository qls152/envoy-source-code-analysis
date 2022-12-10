# XDS模块5

本部分主要讲解ADS在envoy中的实现，本部分讲解基于以下配置

```c++
dynamic_resources:
  ads_config:
    api_type: DELTA_GRPC
    transport_api_version: V3
    grpc_services:
    - envoy_grpc:
        cluster_name: ads_cluster
  cds_config:
    resource_api_version: V3
    ads: {}
```

也即配置中有ads_config选项，且cds_config中使用了ads配置。

## 关键类

**NewGrpcMuxImpl**

```c++
/* 声明 */
class NewGrpcMuxImpl
    : public GrpcMux,
      public GrpcStreamCallbacks<envoy::service::discovery::v3::DeltaDiscoveryResponse>,
      Logger::Loggable<Logger::Id::config> 

/* 关键属性 */
// 创建的GrpcStream实例，用来通信

GrpcStream<envoy::service::discovery::v3::DeltaDiscoveryRequest,
envoy::service::discovery::v3::DeltaDiscoveryResponse>
grpc_stream_;

// 订阅情况，Key为type_url.和api_state_类似
absl::flat_hash_map<std::string, SubscriptionStuffPtr> subscriptions_;

// ack的queue。
PausableAckQueue pausable_ack_queue_;

// 订阅的顺序 ，由初始化时add watch:addSubscription的顺序确定。CDS、EDS、LDS、RDS
std::list<std::string> subscription_ordering_;

// 本地node信息，对应配置中的Node部分
const LocalInfo::LocalInfo& local_info_;

/* 关键方法 */

// 添加watch，会创建watch、watchimpl对象，并管理起来
GrpcMuxWatchPtr addWatch(const std::string& type_url, const std::set<std::string>& resources,
SubscriptionCallbacks& callbacks,
OpaqueResourceDecoder& resource_decoder)

// 删除watch
void removeWatch(const std::string& type_url, Watch* watch);

// 添加订阅信息到subscriptions_、watch_map_中
void updateWatch(const std::string& type_url, Watch* watch,
const std::set<std::string>& resources,
const bool creating_namespace_watch = false);

// 处理xds server的response
void onDiscoveryResponse(
std::unique_ptr<envoy::service::discovery::v3::DeltaDiscoveryResponse>&& message,
ControlPlaneStats& control_plane_stats)

void trySendDiscoveryRequests();

// 添加订阅，一般和添加watch一起使用
void addSubscription(const std::string& type_url);

// 检查是否能发送消息
bool canSendDiscoveryRequest(const std::string& type_url);

// 检查谁需要发送消息。 可以是ACK 或者是订阅状态的变更。ACK的优先级高于non-ACK。non-ACK按照订阅顺序来
absl::optional<std::string> whoWantsToSendDiscoveryRequest();
```

**SubscriptionStuff**

```c++
/* 声明 */
struct SubscriptionStuff

/* 关键属性 */
// Watch的管理器，维护watches<-->subscription 的映射
WatchMap watch_map_;

//维护同一个grpc strem上多个delta xDS protocol的状态
DeltaSubscriptionState sub_state_;

/* 关键方法 */
```

**WatchMap**

```c++
/* 声明 */
class WatchMap : public UntypedConfigUpdateCallbacks, public Logger::Loggable<Logger::Id::config> 

/* 关键属性 */
// watch map管理的所有的watch
absl::flat_hash_set<std::unique_ptr<Watch>> watches_;

// 关注所有资源的watch，也就是resource name为空的
absl::flat_hash_set<Watch*> wildcard_watches_;

//延迟删除的watch
std::unique_ptr<absl::flat_hash_set<Watch*>> deferred_removed_during_update_;

//  key 为resource name，value是关注该resource的所有watch
absl::flat_hash_map<std::string, absl::flat_hash_set<Watch*>> watch_interest_;
const bool use_namespace_matching_;

/* 关键方法 */
// 创建watch对象并管理，默认添加到wildcard_watches_里
Watch* addWatch(SubscriptionCallbacks& callbacks, OpaqueResourceDecoder& resource_decoder);

// 删除watch
void removeWatch(Watch* watch);

// 根据当前的watch对应的resource names做对比，计算新增、删除两部分变更。同时更新watch对应的resource names
AddedRemoved updateWatchInterest(Watch* watch,
const std::set<std::string>& update_to_these_names);

// 传递变更消息到所有相关的watch上。
void onConfigUpdate(const Protobuf::RepeatedPtrField<ProtobufWkt::Any>& resources,
const std::string& version_info) override;

// 根据resource name 查找关注他的watch.
absl::flat_hash_set<Watch*> watchesInterestedIn(const std::string& resource_name);
```

**DeltaSubscriptionState**

```c++
/* 声明 */
class DeltaSubscriptionState : public Logger::Loggable<Logger::Id::config>

/* 关键属性 */
//资源类型
const std::string type_url_;

//所watch的资源名称
std::set<std::string> resource_names_;

//根据状态对watch集合进行操作
UntypedConfigUpdateCallbacks& watch_map_;

//维护资源的版本状态
absl::node_hash_map<std::string, ResourceVersion> resource_versions_;

//需要被添加watch的资源
std::set<std::string> names_added_;

//需要被删除watch的资源
std::set<std::string> names_removed_;

/* 关键方法 */
//处理xds server response，并记录ack所需要的信息，供后面发送ack使用
UpdateAck handleResponse(const envoy::service::discovery::v3::DeltaDiscoveryResponse& message);

void handleGoodResponse(const envoy::service::discovery::v3::DeltaDiscoveryResponse& message);

void handleBadResponse(const EnvoyException& e, UpdateAck& ack);

void addAliasesToResolve(const std::set<std::string>& aliases);

envoy::service::discovery::v3::DeltaDiscoveryRequest getNextRequestAckless();

envoy::service::discovery::v3::DeltaDiscoveryRequest getNextRequestWithAck(const UpdateAck& ack);

//更新需要被watch的资源，主要是维护资源版本
void updateSubscriptionInterest(const std::set<std::string>& cur_added,
const std::set<std::string>& cur_removed);
```

**WatchImpl**

```c++
/* 声明 */
class WatchImpl : public GrpcMuxWatch

/* 关键属性 */
//所使用的资源类型，同GrpcSubscriptionImpl
const std::string type_url_;

//Watch的存储对象
Watch* watch_;

/* 关键方法 */
// 更新watch
void update(const std::set<std::string>& resources)
```

**Watch**

```c++
/* 声明 */
struct Watch

/* 关键属性 */
// watch触发的回调函数
SubscriptionCallbacks& callbacks_;

// 反序列号资源的方法
OpaqueResourceDecoder& resource_decoder_;

// watch资源的名字列表，该列表下资源变更会触发watch
std::set<std::string> resource_names_;

/* 关键方法 */
```

## ads初始化

ads的初始化过程位于source/common/upstream/cluster_manager_impl.cc中的ClusterManagerImpl的构造函数中，在该构造函数中有如下代码实现

```c++
// Now setup ADS if needed, this might rely on a primary cluster.
  // This is the only point where distinction between delta ADS and state-of-the-world ADS is made.
  // After here, we just have a GrpcMux interface held in ads_mux_, which hides
  // whether the backing implementation is delta or SotW.
  if (dyn_resources.has_ads_config()) {
    if (dyn_resources.ads_config().api_type() ==
        envoy::config::core::v3::ApiConfigSource::DELTA_GRPC) {
      ads_mux_ = std::make_shared<Config::NewGrpcMuxImpl>(
          Config::Utility::factoryForGrpcApiConfigSource(*async_client_manager_,
                                                         dyn_resources.ads_config(), stats, false)
              ->create(),
          main_thread_dispatcher,
          *Protobuf::DescriptorPool::generated_pool()->FindMethodByName(
              dyn_resources.ads_config().transport_api_version() ==
                      envoy::config::core::v3::ApiVersion::V3
                  // TODO(htuch): consolidate with type_to_endpoint.cc, once we sort out the future
                  // direction of that module re: https://github.com/envoyproxy/envoy/issues/10650.
                  ? "envoy.service.discovery.v3.AggregatedDiscoveryService.DeltaAggregatedResources"
                  : "envoy.service.discovery.v2.AggregatedDiscoveryService."
                    "DeltaAggregatedResources"),
          dyn_resources.ads_config().transport_api_version(), random_, stats_,
          Envoy::Config::Utility::parseRateLimitSettings(dyn_resources.ads_config()), local_info);
    } else {
      ads_mux_ = std::make_shared<Config::GrpcMuxImpl>(
          local_info,
          Config::Utility::factoryForGrpcApiConfigSource(*async_client_manager_,
                                                         dyn_resources.ads_config(), stats, false)
              ->create(),
          main_thread_dispatcher,
          *Protobuf::DescriptorPool::generated_pool()->FindMethodByName(
              dyn_resources.ads_config().transport_api_version() ==
                      envoy::config::core::v3::ApiVersion::V3
                  // TODO(htuch): consolidate with type_to_endpoint.cc, once we sort out the future
                  // direction of that module re: https://github.com/envoyproxy/envoy/issues/10650.
                  ? "envoy.service.discovery.v3.AggregatedDiscoveryService."
                    "StreamAggregatedResources"
                  : "envoy.service.discovery.v2.AggregatedDiscoveryService."
                    "StreamAggregatedResources"),
          dyn_resources.ads_config().transport_api_version(), random_, stats_,
          Envoy::Config::Utility::parseRateLimitSettings(dyn_resources.ads_config()),
          bootstrap.dynamic_resources().ads_config().set_node_on_first_message_only());
    }
  } else {
    ads_mux_ = std::make_unique<Config::NullGrpcMuxImpl>();
  }
```
由题目开始给出的配置文件可知，ads_config中的api_type为DELTA_GRPC，因此会初始化Config::NewGrpcMuxImpl实例。

通过第一个if分支可知，Envoy中针对Config::NewGrpcMuxImpl所提供的method为

```c++
envoy.service.discovery.v3.AggregatedDiscoveryService.DeltaAggregatedResources

envoy.service.discovery.v2.AggregatedDiscoveryService.DeltaAggregatedResources
```

ads的中各个参数的初始化过程可参考[XDS模块3](./XDS模块(3).md)中相应地讲解。

## cds初始化

当配置ads时，此时cds在初始化其成员变量subscription_时，会走到位于source/common/config/subscription_factory_impl.cc的SubscriptionFactoryImpl::subscriptionFromConfigSource函数的如下if分支中

```c++
 switch (config.config_source_specifier_case()) {
  case // ...........
  case envoy::config::core::v3::ConfigSource::ConfigSourceSpecifierCase::kApiConfigSource: // .....
  case envoy::config::core::v3::ConfigSource::ConfigSourceSpecifierCase::kAds: {
    return std::make_unique<GrpcSubscriptionImpl>(
        cm_.adsMux(), callbacks, resource_decoder, stats, type_url, dispatcher_,
        Utility::configSourceInitialFetchTimeout(config), true);
  }
  default:
    throw EnvoyException("Missing config source specifier in envoy::api::v2::core::ConfigSource");
  }
  NOT_REACHED_GCOVR_EXCL_LINE;
```

也即此时初始化语句为
```c++
td::make_unique<GrpcSubscriptionImpl>(
        cm_.adsMux(), callbacks, resource_decoder, stats, type_url, dispatcher_,
        Utility::configSourceInitialFetchTimeout(config), true);
```
相应的讲解课参考[XDS模块3](./XDS模块(3).md),此处不再赘述。

## 请求发送

在ads_mux_以及cds_初始化完成后，会调用ads_mux_的start接口，其入口位于source/common/upstream/cluster_manager_impl.cc的如下实现
```c++
  ads_mux_->start();
```

start接口的内部实现如下
```c++
// TODO(fredlas) to be removed from the GrpcMux interface very soon.
void NewGrpcMuxImpl::start() { grpc_stream_.establishNewStream(); }
```
上述过程可参考[XDS模块3](./XDS模块(3).md),此处不再赘述。

相应的cds的最终会调用位于source/common/config/grpc_subscription_impl.cc的 GrpcSubscriptionImpl::start接口，该接口有如下实现
```c++
watch_ = grpc_mux_->addWatch(type_url_, resources, *this, resource_decoder_);
```

上述会调用位于source/common/config/new_grpc_mux_impl.cc的NewGrpcMuxImpl::addWatch函数，其实现如下所示
```c++
GrpcMuxWatchPtr NewGrpcMuxImpl::addWatch(const std::string& type_url,
                                         const std::set<std::string>& resources,
                                         SubscriptionCallbacks& callbacks,
                                         OpaqueResourceDecoder& resource_decoder) {
  auto entry = subscriptions_.find(type_url);
  if (entry == subscriptions_.end()) {
    // We don't yet have a subscription for type_url! Make one!
    addSubscription(type_url);
    return addWatch(type_url, resources, callbacks, resource_decoder);
  }

  Watch* watch = entry->second->watch_map_.addWatch(callbacks, resource_decoder);
  // updateWatch() queues a discovery request if any of 'resources' are not yet subscribed.
  updateWatch(type_url, watch, resources);
  return std::make_unique<WatchImpl>(type_url, watch, *this);
}
```
该函数实现如下行为

- 将资源类型type_url注册到subscriptions_中，表示要订阅该类型的资源

- 调用entry->second->watch_map_.addWatch(callbacks, resource_decoder)来创建相应的监控回调

- 针对创建的watch进行相应的更新操作

- 创建WatchImpl对象以初始化GrpcSubscriptionImpl实例的watch_成员

下面具体讲解上述各部分

1. 注册type_url到ubscriptions_的实现如下

```c++
void NewGrpcMuxImpl::addSubscription(const std::string& type_url) {
  subscriptions_.emplace(type_url, std::make_unique<SubscriptionStuff>(type_url, local_info_));
  subscription_ordering_.emplace_back(type_url);
}
````
也即在subscriptions_ 这个map中创建相应的元素，且在subscription_ordering_插入该资源类型，ADS需要保证各个资源类型订阅的顺序，便维护在该数组中。