```c++
// 新Publisher监听方式1: FastDDS广播机制
// ---------------BroadcastManager
// 注册有新Publisher时回调
void BroadcastManager::initCallbacks()
  REGISTRY(registerPublisher, registerPublisher);

void BroadcastManager::registerPublisherCallback(const MsgInfo& result)
  publisherUpdate(topic_name);

// 通知有新Publisher就绪
void BroadcastManager::publisherUpdate(const std::string& topic_name)
  topic_manager_->pubUpdate(topic_name, pubs);

// ---------------TopicManager
bool TopicManager::pubUpdate(const string &topic, const vector<string> &pubs)
  sub->pubUpdate(pubs);

// 新Publisher监听方式2: XMLRPC
// ---------------TopicManager
void TopicManager::start()
  xmlrpc_manager_->bind("publisherUpdate", boost::bind(&TopicManager::pubUpdateCallback, this, _1, _2));

void TopicManager::pubUpdateCallback(XmlRpc::XmlRpcValue& params, XmlRpc::XmlRpcValue& result)
  if (pubUpdate(params[1], pubs))

bool TopicManager::pubUpdate(const string &topic, const vector<string> &pubs)
  sub->pubUpdate(pubs);

// ---------------Subscription
// 按RosTcp协议协商向server握手建立连接
bool Subscription::pubUpdate(const V_string& new_pubs)
  negotiateConnection(*i);

bool Subscription::negotiateConnection(const std::string& xmlrpc_uri)
  // XMLRPC发起连接
  if (!c->executeNonBlock("requestTopic", params))
  PendingConnectionPtr conn(boost::make_shared<PendingConnection>(c, udp_transport, shared_from_this(), xmlrpc_uri));
  // 等待异步发起连接
  XMLRPCManager::instance()->addASyncConnection(conn);

void TopicManager::start()
  xmlrpc_manager_->bind("requestTopic", boost::bind(&TopicManager::requestTopicCallback, this, _1, _2));

void TopicManager::requestTopicCallback(XmlRpc::XmlRpcValue& params, XmlRpc::XmlRpcValue& result)
  if (!requestTopic(params[1], params[2], result))

// XMLRPC连接建立完成回调
void Subscription::pendingConnectionDone(const PendingConnectionPtr& conn, XmlRpcValue& result)
  if (!XMLRPCManager::instance()->validateXmlrpcResponse("requestTopic", result, proto))
  if (proto_name == "TCPROS") {
    // 发起连接
    TransportTCPPtr transport(boost::make_shared<TransportTCP>(&PollManager::instance()->getPollSet()));
    if (transport->connect(pub_host, pub_port))
    {
      ConnectionPtr connection(boost::make_shared<Connection>());
      TransportPublisherLinkPtr pub_link(boost::make_shared<TransportPublisherLink>(shared_from_this(), xmlrpc_uri, transport_hints_));
      connection->initialize(transport, false, HeaderReceivedFunc());
      pub_link->initialize(connection);
      ConnectionManager::instance()->addConnection(connection);

      boost::mutex::scoped_lock lock(publisher_links_mutex_);
      addPublisherLink(pub_link);
    }
}

// 用户订阅
Subscriber NodeHandle::subscribe(SubscribeOptions& ops)
  if (TopicManager::instance()->subscribe(ops))

bool TopicManager::subscribe(const SubscribeOptions& ops)
    SubscriptionPtr s(boost::make_shared<Subscription>(ops.topic, md5sum, datatype, ops.transport_hints));
    s->addCallback(ops.helper, ops.md5sum, ops.callback_queue, ops.queue_size, ops.tracked_object, ops.allow_concurrent_callbacks);

// 回调入队列 callbacks_
bool Subscription::addCallback(const SubscriptionCallbackHelperPtr& helper, const std::string& md5sum, 
  CallbackQueueInterface* queue, int32_t queue_size, const VoidConstPtr& tracked_object, bool allow_concurrent_callbacks)
    CallbackInfoPtr info(boost::make_shared<CallbackInfo>());
    info->helper_ = helper;
    info->callback_queue_ = queue;
    info->subscription_queue_ = boost::make_shared<SubscriptionQueue>(name_, queue_size, allow_concurrent_callbacks);
    info->tracked_object_ = tracked_object;
    info->has_tracked_object_ = false;
    callbacks_.push_back(info);

// 消息到达时, 遍历注册的回调队列 callbacks_ , 把消息放入 CallbackQueue 等待回调用户函数
uint32_t Subscription::handleMessage(const SerializedMessage& m, bool ser, bool nocopy, 
  const boost::shared_ptr<M_string>& connection_header, const PublisherLinkPtr& link)
  for (V_CallbackInfo::iterator cb = callbacks_.begin(); cb != callbacks_.end(); ++cb)
    const CallbackInfoPtr& info = *cb;
      // 把消息放入 SubscriptionQueue
      info->subscription_queue_->push(false, info->helper_, deserializer,
        info->has_tracked_object_, info->tracked_object_,
        nonconst_need_copy, receipt_time, &was_full);
      // 把消息放入 CallbackQueue
      info->callback_queue_->addCallback(info->subscription_queue_, (uint64_t)info.get());

// 消息到达方式1: 共享内存
void ShmManager::threadFunc()
  while (started_ && ros::ok())
  {
                    local_item.shm_sub_ptr->handleMessage(m, false, true, header_ptr, NULL);
  }

// 消息到达方式2: sock
void TransportPublisherLink::handleMessage(const SerializedMessage& m, bool ser, bool nocopy)
  SubscriptionPtr parent = parent_.lock();
    stats_.drops_ += parent->handleMessage(m, ser, nocopy, getConnection()->getHeader().getValues(), shared_from_this());

void TransportPublisherLink::onMessage(const ConnectionPtr& conn, const boost::shared_array<uint8_t>& buffer, uint32_t size, bool success)
    handleMessage(SerializedMessage(buffer, size), true, false);

void TransportPublisherLink::onMessageLength(const ConnectionPtr& conn, const boost::shared_array<uint8_t>& buffer, uint32_t size, bool success)
  connection_->read(len, boost::bind(&TransportPublisherLink::onMessage, this, _1, _2, _3, _4));

bool TransportPublisherLink::onHeaderReceived(const ConnectionPtr& conn, const Header& header)
  connection_->read(4, boost::bind(&TransportPublisherLink::onMessageLength, this, _1, _2, _3, _4));

bool TransportPublisherLink::initialize(const ConnectionPtr& connection)
    connection_->setHeaderReceivedCallback(boost::bind(&TransportPublisherLink::onHeaderReceived, this, _1, _2));

// 回到前面 -> XMLRPC连接建立完成回调
void Subscription::pendingConnectionDone(const PendingConnectionPtr& conn, XmlRpcValue& result)
      TransportPublisherLinkPtr pub_link(boost::make_shared<TransportPublisherLink>(shared_from_this(), xmlrpc_uri, transport_hints_));
      pub_link->initialize(connection);

```