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

2. 调用entry->second->watch_map_.addWatch(callbacks, resource_decoder)来创建相应的监控回调

其会进入到位于source/common/config/watch_map.cc的WatchMap::addWatch函数，其实现如下

```c++
Watch* WatchMap::addWatch(SubscriptionCallbacks& callbacks,
                          OpaqueResourceDecoder& resource_decoder) {
  auto watch = std::make_unique<Watch>(callbacks, resource_decoder);
  Watch* watch_ptr = watch.get();
  wildcard_watches_.insert(watch_ptr);
  watches_.insert(std::move(watch));
  return watch_ptr;
}
```
上述会创建Watch对象，并将watch对象插入wildcard_watches_中，
其中wildcard_watches_在代码中有如下注释

> // Watches whose interest set is currently empty, 
> which is interpreted as "everything".

也即当wildcard_watches_为空时，表示client端订阅所有资源

3. 针对创建的watch进行相应的更新操作

该部分实现最为核心，上述会调用位于source/common/config/new_grpc_mux_impl.cc的NewGrpcMuxImpl::updateWatch函数，其实现如下
```c++
// Updates the list of resource names watched by the given watch. If an added name is new across
// the whole subscription, or if a removed name has no other watch interested in it, then the
// subscription will enqueue and attempt to send an appropriate discovery request.
void NewGrpcMuxImpl::updateWatch(const std::string& type_url, Watch* watch,
                                 const std::set<std::string>& resources) {
  // .........
  auto added_removed = sub->second->watch_map_.updateWatchInterest(watch, resources);
  sub->second->sub_state_.updateSubscriptionInterest(added_removed.added_, added_removed.removed_);
  // Tell the server about our change in interest, if any.
  if (sub->second->sub_state_.subscriptionUpdatePending()) {
    trySendDiscoveryRequests();
  }
}
```

该函数首先会调用位于source/common/config/watch_map.cc的WatchMap::updateWatchInterest函数，其有如下实现

```c++
AddedRemoved WatchMap::updateWatchInterest(Watch* watch,
                                           const std::set<std::string>& update_to_these_names) {
  if (update_to_these_names.empty()) {
    wildcard_watches_.insert(watch);
  } else {
    wildcard_watches_.erase(watch);
  }

  std::vector<std::string> newly_added_to_watch;
  std::set_difference(update_to_these_names.begin(), update_to_these_names.end(),
                      watch->resource_names_.begin(), watch->resource_names_.end(),
                      std::inserter(newly_added_to_watch, newly_added_to_watch.begin()));

  std::vector<std::string> newly_removed_from_watch;
  std::set_difference(watch->resource_names_.begin(), watch->resource_names_.end(),
                      update_to_these_names.begin(), update_to_these_names.end(),
                      std::inserter(newly_removed_from_watch, newly_removed_from_watch.begin()));

  watch->resource_names_ = update_to_these_names;

  return AddedRemoved(findAdditions(newly_added_to_watch, watch),
                      findRemovals(newly_removed_from_watch, watch));
}

```

- 如果订阅的资源为空，则表示客户端想要订阅所有的资源；否则，将其在wildcard_watches_删除

- 调用std::set_difference来计算相应要增加的资源，以及要remove的资源

std::set_difference:C++17引入的功能，其一个声明如下
```c++
template< class ExecutionPolicy, class ForwardIt1, class ForwardIt2,
          class ForwardIt3 >
ForwardIt3 set_difference( ExecutionPolicy&& policy,
                           ForwardIt1 first1, ForwardIt1 last1,
                           ForwardIt2 first2, ForwardIt2 last2,
                           ForwardIt3 d_first );
```
如果某个元素x在已排好序的[first1, last1）区间中，但不再已排好序的[first2, last2)区间中，则将x拷贝到以d_first开头的区间。

结果区间也将排序。等效元素被单独处理，也即，如果某个元素x在[first1，last1）中被找到m次，在[first2，last2）中被发现n次，那么它将被复制到d_first中std::max（m-n，0）次。结果区间不能与任何一个输入范围重叠。

关于set_difference详细使用可参考[set_difference](https://en.cppreference.com/w/cpp/algorithm/set_difference)

AddedRemoved类是对新增加的资源和将要移除的资源的一个汇总。
在初始化该类时，会调用findAdditions和 findRemovals接口

findAdditions接口实现如下
```c++
std::set<std::string> WatchMap::findAdditions(const std::vector<std::string>& newly_added_to_watch,
                                              Watch* watch) {
  std::set<std::string> newly_added_to_subscription;
  for (const auto& name : newly_added_to_watch) {
    auto entry = watch_interest_.find(name);
    if (entry == watch_interest_.end()) {
      newly_added_to_subscription.insert(name);
      watch_interest_[name] = {watch};
    } else {
      // Add this watch to the already-existing set at watch_interest_[name]
      entry->second.insert(watch);
    }
  }
  return newly_added_to_subscription;
}
```
也即将新增加的资源的watch注册到watch_interest_中，并将订阅的资源去重，形成一个set，返回到客户端

findRemovals的实现可参考findAdditions

回到NewGrpcMuxImpl::updateWatch函数，此时会调用位于source/common/config/delta_subscription_state.cc的DeltaSubscriptionState::updateSubscriptionInterest函数，其实现如下所示
```c++
void DeltaSubscriptionState::updateSubscriptionInterest(const std::set<std::string>& cur_added,
                                                        const std::set<std::string>& cur_removed) {
  for (const auto& a : cur_added) {
    setResourceWaitingForServer(a);
    // If interest in a resource is removed-then-added (all before a discovery request
    // can be sent), we must treat it as a "new" addition: our user may have forgotten its
    // copy of the resource after instructing us to remove it, and need to be reminded of it.
    names_removed_.erase(a);
    names_added_.insert(a);
  }
  for (const auto& r : cur_removed) {
    setLostInterestInResource(r);
    // Ideally, when interest in a resource is added-then-removed in between requests,
    // we would avoid putting a superfluous "unsubscribe [resource that was never subscribed]"
    // in the request. However, the removed-then-added case *does* need to go in the request,
    // and due to how we accomplish that, it's difficult to distinguish remove-add-remove from
    // add-remove (because "remove-add" has to be treated as equivalent to just "add").
    names_added_.erase(r);
    names_removed_.insert(r);
  }
}

```
该函数实现通过注释可以较好理解。

回到一开始的函数，接下来有如下实现
```c++
// Tell the server about our change in interest, if any.
  if (sub->second->sub_state_.subscriptionUpdatePending()) {
    trySendDiscoveryRequests();
  }
```

上述会首先调用位于source/common/config/delta_subscription_state.cc的DeltaSubscriptionState::subscriptionUpdatePending()函数

其实现为
```c++
bool DeltaSubscriptionState::subscriptionUpdatePending() const {
  return !names_added_.empty() || !names_removed_.empty() ||
         !any_request_sent_yet_in_current_stream_;
}
```
该函数有三个判断条件names_added_是否为空，names_removed_是否为空，以及any_request_sent_yet_in_current_stream_是否为false。

any_request_sent_yet_in_current_stream_默认为false。

最终通过条件判断后，会进入 NewGrpcMuxImpl::trySendDiscoveryRequests() 函数，该函数进行真正的sendRequest相关逻辑。

##  NewGrpcMuxImpl::trySendDiscoveryRequests()讲解

该函数实现如下
```c++
void NewGrpcMuxImpl::trySendDiscoveryRequests() {
  while (true) {
    // Do any of our subscriptions even want to send a request?
    absl::optional<std::string> maybe_request_type = whoWantsToSendDiscoveryRequest(); // 1
    if (!maybe_request_type.has_value()) {
      break;
    }
    // If so, which one (by type_url)?
    std::string next_request_type_url = maybe_request_type.value();
    // If we don't have a subscription object for this request's type_url, drop the request.
    auto sub = subscriptions_.find(next_request_type_url);
    RELEASE_ASSERT(sub != subscriptions_.end(),
                   fmt::format("Tried to send discovery request for non-existent subscription {}.",
                               next_request_type_url));

    // Try again later if paused/rate limited/stream down.
    if (!canSendDiscoveryRequest(next_request_type_url)) {
      break;
    }
    envoy::service::discovery::v3::DeltaDiscoveryRequest request;
    // Get our subscription state to generate the appropriate DeltaDiscoveryRequest, and send.
    if (!pausable_ack_queue_.empty()) {
      // Because ACKs take precedence over plain requests, if there is anything in the queue, it's
      // safe to assume it's of the type_url that we're wanting to send.
      request = sub->second->sub_state_.getNextRequestWithAck(pausable_ack_queue_.popFront()); // 2
    } else {
      request = sub->second->sub_state_.getNextRequestAckless(); // 3
    }
    VersionConverter::prepareMessageForGrpcWire(request, transport_api_version_);
    grpc_stream_.sendMessage(request); // 4
  }
  grpc_stream_.maybeUpdateQueueSizeStat(pausable_ack_queue_.size());
}

```

该函数实现了XDS增量协议，标注了1-4关键步骤，依次如下

1. 判断那种类型的资源有相应的更新，其通过whoWantsToSendDiscoveryRequest获得，该函数实现如下
```c++
// Checks whether we have something to say in a DeltaDiscoveryRequest, which can be an ACK and/or
// a subscription update. (Does not check whether we *can* send that DeltaDiscoveryRequest).
// Returns the type_url we should send the DeltaDiscoveryRequest for (if any).
// First, prioritizes ACKs over non-ACK subscription interest updates.
// Then, prioritizes non-ACK updates in the order the various types
// of subscriptions were activated.
absl::optional<std::string> NewGrpcMuxImpl::whoWantsToSendDiscoveryRequest() {
  // All ACKs are sent before plain updates. trySendDiscoveryRequests() relies on this. So, choose
  // type_url from pausable_ack_queue_ if possible, before looking at pending updates.
  if (!pausable_ack_queue_.empty()) {
    return pausable_ack_queue_.front().type_url_;
  }
  // If we're looking to send multiple non-ACK requests, send them in the order that their
  // subscriptions were initiated.
  for (const auto& sub_type : subscription_ordering_) {
    auto sub = subscriptions_.find(sub_type);
    if (sub != subscriptions_.end() && sub->second->sub_state_.subscriptionUpdatePending() &&
        !pausable_ack_queue_.paused(sub_type)) {
      return sub->first;
    }
  }
  return absl::nullopt;
}
```
该函数注释较为清晰，此处不做说明。

2. 当pausable_ack_queue_非空，也即要么有已经收到的response或者ads已经调用了pause接口，此时在sendRequest时，会调用位于source/common/config/delta_subscription_state.cc的DeltaSubscriptionState::getNextRequestWithAck(const UpdateAck& ack) 函数来构造相应request
其实现如下
```c++
envoy::service::discovery::v3::DeltaDiscoveryRequest
DeltaSubscriptionState::getNextRequestWithAck(const UpdateAck& ack) {
  envoy::service::discovery::v3::DeltaDiscoveryRequest request = getNextRequestAckless();
  request.set_response_nonce(ack.nonce_);
  if (ack.error_detail_.code() != Grpc::Status::WellKnownGrpcStatus::Ok) {
    // Don't needlessly make the field present-but-empty if status is ok.
    request.mutable_error_detail()->CopyFrom(ack.error_detail_);
  }
  return request;
}
```
其调用getNextRequestAckless()获得初步的request，该函数实现如下
```c++
envoy::service::discovery::v3::DeltaDiscoveryRequest
DeltaSubscriptionState::getNextRequestAckless() {
  envoy::service::discovery::v3::DeltaDiscoveryRequest request;
  if (!any_request_sent_yet_in_current_stream_) { // 1
    any_request_sent_yet_in_current_stream_ = true;
    // initial_resource_versions "must be populated for first request in a stream".
    // Also, since this might be a new server, we must explicitly state *all* of our subscription
    // interest.
    for (auto const& resource : resource_versions_) {
      // Populate initial_resource_versions with the resource versions we currently have.
      // Resources we are interested in, but are still waiting to get any version of from the
      // server, do not belong in initial_resource_versions. (But do belong in new subscriptions!)
      if (!resource.second.waitingForServer()) {
        (*request.mutable_initial_resource_versions())[resource.first] = resource.second.version(); // 2
      }
      // As mentioned above, fill resource_names_subscribe with everything, including names we
      // have yet to receive any resource for.
      names_added_.insert(resource.first);
    }
    names_removed_.clear();
  }
  std::copy(names_added_.begin(), names_added_.end(),
            Protobuf::RepeatedFieldBackInserter(request.mutable_resource_names_subscribe())); // 3
  std::copy(names_removed_.begin(), names_removed_.end(),
            Protobuf::RepeatedFieldBackInserter(request.mutable_resource_names_unsubscribe())); // 4
  names_added_.clear();
  names_removed_.clear();

  request.set_type_url(type_url_); // 5
  request.mutable_node()->MergeFrom(local_info_.node());
  return request;
}

```

上述函数注释较为清晰，不再详细说明。

3. 该实现在2中已做说明

4. 最终调用grpc_strem的sendMessage接口向server端发送请求

至此，发送请求的流程讲解完毕。

--------------------------------------------

**请求发送的时序图后续补充**

---------------------------------------------

## 资源更新

资源更新的入口函数本文以位于source/common/config/new_grpc_mux_impl.cc的NewGrpcMuxImpl::onDiscoveryResponse接口为出发点，

该函数声明如下
```c++
void NewGrpcMuxImpl::onDiscoveryResponse(
    std::unique_ptr<envoy::service::discovery::v3::DeltaDiscoveryResponse>&& message,
    ControlPlaneStats&)
```

其核心过程如下

1. 通过response的type_url查询是否被客户端订阅，也即如下实现

```c++
auto sub = subscriptions_.find(message->type_url());
  if (sub == subscriptions_.end()) {
    ENVOY_LOG(warn,
              "Dropping received DeltaDiscoveryResponse (with version {}) for non-existent "
              "subscription {}.",
              message->system_version_info(), message->type_url());
    return;
  }
```

2. 增量XDS中的response可以对资源赋予别名，在解析response时，若存在别名，则先将其转换成相应的资源订阅的watch

```c++
// When an on-demand request is made a Watch is created using an alias, as the resource name isn't
  // known at that point. When an update containing aliases comes back, we update Watches with
  // resource names.
  for (const auto& r : message->resources()) {
    if (r.aliases_size() > 0) {
      AddedRemoved converted = sub->second->watch_map_.convertAliasWatchesToNameWatches(r);
      sub->second->sub_state_.updateSubscriptionInterest(converted.added_, converted.removed_);
    }
  }
```

3. 调用位于source/common/config/delta_subscription_state.cc中的 DeltaSubscriptionState::handleResponse函数，进一步完成相应的reponse解析

其实现如下
```c++
UpdateAck DeltaSubscriptionState::handleResponse(
    const envoy::service::discovery::v3::DeltaDiscoveryResponse& message) {
  // We *always* copy the response's nonce into the next request, even if we're going to make that
  // request a NACK by setting error_detail.
  UpdateAck ack(message.nonce(), type_url_);
  try {
    handleGoodResponse(message);
  } catch (const EnvoyException& e) {
    handleBadResponse(e, ack);
  }
  return ack;
}
```

  该函数会先更新相应的ack，也即创建一个新的UpdateAck.

  调用位于同一路径下的handleGoodResponse函数进行相应的response的解析。
  
  handleGoodResponse函数首先将新增的/更新的资源解析出并将其塞入一个数组中，其实现过程如下
  ```c++
  absl::flat_hash_set<std::string> names_added_removed;
  for (const auto& resource : message.resources()) {
    if (!names_added_removed.insert(resource.name()).second) {
      throw EnvoyException(
          fmt::format("duplicate name {} found among added/updated resources", resource.name()));
    }
    // DeltaDiscoveryResponses for unresolved aliases don't contain an actual resource
    if (!resource.has_resource() && resource.aliases_size() > 0) {
      continue;
    }
    if (message.type_url() != resource.resource().type_url()) {
      throw EnvoyException(fmt::format("type URL {} embedded in an individual Any does not match "
                                       "the message-wide type URL {} in DeltaDiscoveryResponse {}",
                                       resource.resource().type_url(), message.type_url(),
                                       message.DebugString()));
    }
  }
  ```
  若response中同一种资源返回多次，则client判定为失败，不更新相应资源；若资源的type_url（资源类型)和最外层的type_url也不一致，则client也拒绝更新。

  handleGoodResponse函数会解析response中移除的资源，其实现如下
  ```c++
  for (const auto& name : message.removed_resources()) {
    if (!names_added_removed.insert(name).second) {
      throw EnvoyException(
          fmt::format("duplicate name {} found in the union of added+removed resources", name));
    }
  }
  ```
  
  handleGoodResponse函数会调用位于source/common/config/watch_map.cc中的WatchMap::onConfigUpdate函数，
  ```c++
  watch_map_.onConfigUpdate(message.resources(), message.removed_resources(), message.system_version_info());
  ```
  WatchMap::onConfigUpdate函数会最终调用位于source/common/config/grpc_subscription_impl.cc中的GrpcSubscriptionImpl::onConfigUpdate函数，完成相应的资源更新

  handleGoodResponse函数会设置相应更新/移除资源的resource，其实现如下
  ```c++
  for (const auto& resource : message.resources()) {
    setResourceVersion(resource.name(), resource.version());
  }
  // If a resource is gone, there is no longer a meaningful version for it that makes sense to
  // provide to the server upon stream reconnect: either it will continue to not exist, in which
  // case saying nothing is fine, or the server will bring back something new, which we should
  // receive regardless (which is the logic that not specifying a version will get you).
  //
  // So, leave the version map entry present but blank. It will be left out of
  // initial_resource_versions messages, but will remind us to explicitly tell the server "I'm
  // cancelling my subscription" when we lose interest.
  for (const auto& resource_name : message.removed_resources()) {
    if (resource_names_.find(resource_name) != resource_names_.end()) {
      setResourceWaitingForServer(resource_name);
    }
  }
  ```

  若handleGoodResponse调用失败，则会调用handleBadResponse函数，其实现如下
  ```c++
  void DeltaSubscriptionState::handleBadResponse(const EnvoyException& e, UpdateAck& ack) {
    // Note that error_detail being set is what indicates that a DeltaDiscoveryRequest is a NACK.
    ack.error_detail_.set_code(Grpc::Status::WellKnownGrpcStatus::Internal);
    ack.error_detail_.set_message(Config::Utility::truncateGrpcStatusMessage(e.what()));
    ENVOY_LOG(warn, "delta config for {} rejected: {}", type_url_, e.what());
    watch_map_.onConfigUpdateFailed(Envoy::Config::ConfigUpdateFailureReason::UpdateRejected, &e);
  }
  ```
  上述函数会设置ack的error_detail_域。

4. 调用NewGrpcMuxImpl::kickOffAck发送相应的request

其实现如下所示
```c++
void NewGrpcMuxImpl::kickOffAck(UpdateAck ack) {
  pausable_ack_queue_.push(std::move(ack));
  trySendDiscoveryRequests();
}
```
上述首先将更新的UpdateAck放置到pausable_ack_queue_队列，并调用trySendDiscoveryRequests()发送相应的请求。

##  WatchMap::onConfigUpdate

该函数是进行资源更新的另一个核心函数，其实现如下
```c++
void WatchMap::onConfigUpdate(
    const Protobuf::RepeatedPtrField<envoy::service::discovery::v3::Resource>& added_resources,
    const Protobuf::RepeatedPtrField<std::string>& removed_resources,
    const std::string& system_version_info) {
  // Build a pair of maps: from watches, to the set of resources {added,removed} that each watch
  // cares about. Each entry in the map-pair is then a nice little bundle that can be fed directly
  // into the individual onConfigUpdate()s.
  std::vector<DecodedResourceImplPtr> decoded_resources;
  absl::flat_hash_map<Watch*, std::vector<DecodedResourceRef>> per_watch_added;
  for (const auto& r : added_resources) {
    const absl::flat_hash_set<Watch*>& interested_in_r = watchesInterestedIn(r.name());
    // If there are no watches, then we don't need to decode. If there are watches, they should all
    // be for the same resource type, so we can just use the callbacks of the first watch to decode.
    if (interested_in_r.empty()) {
      continue;
    }
    decoded_resources.emplace_back(
        new DecodedResourceImpl((*interested_in_r.begin())->resource_decoder_, r));
    for (const auto& interested_watch : interested_in_r) {
      per_watch_added[interested_watch].emplace_back(*decoded_resources.back());
    }
  }
  absl::flat_hash_map<Watch*, Protobuf::RepeatedPtrField<std::string>> per_watch_removed;
  for (const auto& r : removed_resources) {
    const absl::flat_hash_set<Watch*>& interested_in_r = watchesInterestedIn(r);
    for (const auto& interested_watch : interested_in_r) {
      *per_watch_removed[interested_watch].Add() = r;
    }
  }

  // We just bundled up the updates into nice per-watch packages. Now, deliver them.
  for (const auto& added : per_watch_added) {
    const Watch* cur_watch = added.first;
    const auto removed = per_watch_removed.find(cur_watch);
    if (removed == per_watch_removed.end()) {
      // additions only, no removals
      cur_watch->callbacks_.onConfigUpdate(added.second, {}, system_version_info);
    } else {
      // both additions and removals
      cur_watch->callbacks_.onConfigUpdate(added.second, removed->second, system_version_info);
      // Drop the removals now, so the final removals-only pass won't use them.
      per_watch_removed.erase(removed);
    }
  }
  // Any removals-only updates will not have been picked up in the per_watch_added loop.
  for (auto& removed : per_watch_removed) {
    removed.first->callbacks_.onConfigUpdate({}, removed.second, system_version_info);
  }
}

```

上述函数

- 首先通过查找wildcard_watches_中，确定是否已经订阅了相关资源

- 然后将更新的资源依次放入相应watch列表中

- 更新需要移除的资源的watch列表

- 调用相应的onConfigUpdate接口，更新相应的资源

## 总结

本文讲解了ADS以及增量协议在Envoy中的实现，本文未讲解如何取消订阅某个资源相关的实现，以及相应源码依赖的时序图。

后续有时间补充上。



  