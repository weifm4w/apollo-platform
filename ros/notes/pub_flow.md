```c++
// ---------------ConnectionManager
// Publisher作为TCP server监听Subscriber连接
void ConnectionManager::start()
  tcpserver_transport_ = boost::make_shared<TransportTCP>(&poll_manager_->getPollSet());
  if (!tcpserver_transport_->listen(network::getTCPROSPort(), MAX_TCPROS_CONN_QUEUE,
				    boost::bind(&ConnectionManager::tcprosAcceptConnection, this, _1)))

// 监听到新Subscriber连接
void ConnectionManager::tcprosAcceptConnection(const TransportTCPPtr& transport)
  ConnectionPtr conn(boost::make_shared<Connection>());
  addConnection(conn);
  conn->initialize(transport, true, boost::bind(&ConnectionManager::onConnectionHeaderReceived, this, _1, _2));

// 按RosTcp协议处理了Subscriber连接协议头
bool ConnectionManager::onConnectionHeaderReceived(const ConnectionPtr& conn, const Header& header)
  if (header.getValue("topic", val))
  {
    TransportSubscriberLinkPtr sub_link(boost::make_shared<TransportSubscriberLink>());
    sub_link->initialize(conn);
    ret = sub_link->handleHeader(header);
  }
  else if (header.getValue("service", val))
  {
    ServiceClientLinkPtr link(boost::make_shared<ServiceClientLink>());
    link->initialize(conn);
    ret = link->handleHeader(header);
  }

// ---------------TransportSubscriberLink
// 按RosTcp协议Publisher回复Subscriber
bool TransportSubscriberLink::handleHeader(const Header& header)
  PublicationPtr pt = TopicManager::instance()->lookupPublication(topic);
  M_string m;
  m["type"] = pt->getDataType();
  m["md5sum"] = pt->getMD5Sum();
  m["message_definition"] = pt->getMessageDefinition();
  m["callerid"] = this_node::getName();
  m["latching"] = pt->isLatching() ? "1" : "0";
  m["topic"] = topic_;
  connection_->writeHeader(m, boost::bind(&TransportSubscriberLink::onHeaderWritten, this, _1));
  pt->addSubscriberLink(shared_from_this());

// ---------------Publication
// 把Subscriber添加订阅链表 subscriber_links_
void Publication::addSubscriberLink(const SubscriberLinkPtr& sub_link)
    subscriber_links_.push_back(sub_link);

// 在发布消息时需放入 订阅链表 subscriber_links_ 的消息队列里等待发送
bool Publication::enqueueMessage(const SerializedMessage& m)
  for(V_SubscriberLink::iterator i = subscriber_links_.begin(); i != subscriber_links_.end(); ++i)
  {
    const SubscriberLinkPtr& sub_link = (*i);
    if(sub_link->getDefaultTransport())
	{
      sub_link->enqueueMessage(m, true, false);      
    }
  }

// 再回过头看用户调用publish的流程
void Publication::publish(SerializedMessage& m)
        publish_queue_.push_back(m);

// Publish消息先入列
void Publication::processPublishQueue()
  V_SerializedMessage queue;
    queue.insert(queue.end(), publish_queue_.begin(), publish_queue_.end());
  V_SerializedMessage::iterator it = queue.begin();
  V_SerializedMessage::iterator end = queue.end();
  for (; it != end; ++it)
  {
    // 在发布消息时需放入 订阅链表 subscriber_links_ 的消息队列里等待发送
    enqueueMessage(*it);
  }

// ---------------TopicManager
// Publish消息队列被消费处理
void TopicManager::processPublishQueues()
  V_Publication::iterator it = advertised_topics_.begin();
  V_Publication::iterator end = advertised_topics_.end();
  for (; it != end; ++it)
  {
    const PublicationPtr& pub = *it;
    pub->processPublishQueue();
  }

void TopicManager::start()
  poll_manager_->addPollThreadListener(boost::bind(&TopicManager::processPublishQueues, this));

// 信号与槽机制
boost::signals2::connection PollManager::addPollThreadListener(const VoidFunc& func)
{
  boost::recursive_mutex::scoped_lock lock(signal_mutex_);
  return poll_signal_.connect(func);
}

// ---------------PollManager
// 真正publish消息的线程
void PollManager::threadFunc()
{
  disableAllSignalsInThisThread();
  while (!shutting_down_)
  {
    {
      boost::recursive_mutex::scoped_lock lock(signal_mutex_);
      poll_signal_();
    }
    if (shutting_down_)
    {
      return;
    }
    poll_set_.update(100);
  }
}
```