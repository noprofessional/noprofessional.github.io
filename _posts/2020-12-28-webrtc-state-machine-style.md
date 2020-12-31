---
layout: post
title:  "chrome webrtc中常用的多状态切换抽象"
date:   2020-12-24 14:23:48 +0800
categories: webrtc
---

### 基础格式

```cpp

//继承rtc::MessageHandler
class RTC_EXPORT BasicPortAllocatorSession : public PortAllocatorSession,
                                             public rtc::MessageHandler
{...};

//触发阶段
void BasicPortAllocatorSession::StartGettingPorts() {
  RTC_DCHECK_RUN_ON(network_thread_);
  state_ = SessionState::GATHERING;
  if (!socket_factory_) {
    owned_socket_factory_.reset(
        new rtc::BasicPacketSocketFactory(network_thread_));
    socket_factory_ = owned_socket_factory_.get();
  }

  //触发，this 作为 handler 传入，event的类型取决于Messageid就是MSG_CONFIG_START
  //message，会进入network_thread_内部的队列等待被触发
  network_thread_->Post(RTC_FROM_HERE, this, MSG_CONFIG_START);

  RTC_LOG(LS_INFO) << "Start getting ports with turn_port_prune_policy "
                   << turn_port_prune_policy_;
}

//被触发 根据MessageID决定处理方式
void BasicPortAllocatorSession::OnMessage(rtc::Message* message) {
  switch (message->message_id) {
    case MSG_CONFIG_START:
      GetPortConfigurations();
      break;
    case MSG_CONFIG_READY:
      OnConfigReady(static_cast<PortConfiguration*>(message->pdata));
      break;
    case MSG_ALLOCATE:
      OnAllocate();
      break;
    case MSG_SEQUENCEOBJECTS_CREATED:
      OnAllocationSequenceObjectsCreated();
      break;
    case MSG_CONFIG_STOP:
      OnConfigStop();
      break;
    default:
      RTC_NOTREACHED();
  }
}

//处理方式
void BasicPortAllocatorSession::GetPortConfigurations() {
  RTC_DCHECK_RUN_ON(network_thread_);

  PortConfiguration* config =
      new PortConfiguration(allocator_->stun_servers(), username(), password());

  for (const RelayServerConfig& turn_server : allocator_->turn_servers()) {
    config->AddRelay(turn_server);
  }
  ConfigReady(config);
}
//下一步的触发
void BasicPortAllocatorSession::ConfigReady(PortConfiguration* config) {
  RTC_DCHECK_RUN_ON(network_thread_);
  network_thread_->Post(RTC_FROM_HERE, this, MSG_CONFIG_READY, config);
}

```