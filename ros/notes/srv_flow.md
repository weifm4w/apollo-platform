### 服务连接管理
```c++
// 设计模式:ros大部分基于reactor设计模式,通过注册事件回调,数据流转流式处理
// 下面代码分析从用户代码开始，自顶向下。
// -------------------- NodeHandle
ros::NodeHandle service_node_handle_;  // 用户代码
// NodeHandle 声明
NodeHandle(const std::string& ns = std::string(), const M_string& remappings = M_string());
NodeHandle::NodeHandle(const std::string& ns, const M_string& remappings)
  construct(rhs.namespace_, true); 
void NodeHandle::construct(const std::string& ns, bool validate_name)
    ros::start();
// -------------------- init.cpp
void start()
  ServiceManager::instance()->start();
  ConnectionManager::instance()->start();
```
```c++
// -------------------- ConnectionManager
void ConnectionManager::start()
{
  poll_manager_ = PollManager::instance();
  // 通用连接管理，每个node都在一个端口监听新连接
  tcpserver_transport_ = boost::make_shared<TransportTCP>(&poll_manager_->getPollSet());
  if (!tcpserver_transport_->listen(network::getTCPROSPort(), MAX_TCPROS_CONN_QUEUE, 
				    boost::bind(&ConnectionManager::tcprosAcceptConnection, this, _1)))
  ......
}
bool TransportTCP::listen(int port, int backlog, const AcceptCallback& accept_cb)
  is_server_ = true;
  accept_cb_ = accept_cb;
  ::listen(sock_, backlog);
  if (!initializeSocket())
bool TransportTCP::initializeSocket()
    if (is_server_)
    {
      cached_remote_host_ = "TCPServer Socket";
    }
    poll_set_->addSocket(sock_, boost::bind(&TransportTCP::socketUpdate, this, _1), shared_from_this());
void TransportTCP::socketUpdate(int events)
    if ((events & POLLIN) && expecting_read_)
    {
      if (is_server_)
      {
        // server接收新client连接,一个node可多个client,对应多个transport
        TransportTCPPtr transport = accept();
        if (transport)
        {
          ROS_ASSERT(accept_cb_);
          accept_cb_(transport); // 回调 tcprosAcceptConnection
        }
      }
```
```c++
// -------------------- ConnectionManager
void ConnectionManager::tcprosAcceptConnection(const TransportTCPPtr& transport)
  // 每个client都新建一个Connection
  ConnectionPtr conn(boost::make_shared<Connection>());
  addConnection(conn);
  conn->initialize(transport, true, boost::bind(&ConnectionManager::onConnectionHeaderReceived, this, _1, _2));

bool ConnectionManager::onConnectionHeaderReceived(const ConnectionPtr& conn, const Header& header)
  if (header.getValue("topic", val))
  {
    TransportSubscriberLinkPtr sub_link(boost::make_shared<TransportSubscriberLink>());
    sub_link->initialize(conn);
    ret = sub_link->handleHeader(header);
  }
  else if (header.getValue("service", val))
  {
    // 通过rostcp协议头得知是service类型,新建ClientLink
    ServiceClientLinkPtr link(boost::make_shared<ServiceClientLink>());
    link->initialize(conn);
    ret = link->handleHeader(header);
  }
```
```c++
// -------------------- Connection
// 每个client连接对应一个Connection,初始化时注册事件可读、可写、断连事件,并开始读头4字节通过 onHeaderLengthRead 解释
void Connection::initialize(const TransportPtr& transport, bool is_server, const HeaderReceivedFunc& header_func)
  transport_->setReadCallback(boost::bind(&Connection::onReadable, this, _1));
  transport_->setWriteCallback(boost::bind(&Connection::onWriteable, this, _1));
  transport_->setDisconnectCallback(boost::bind(&Connection::onDisconnect, this, _1));
  if (header_func)
  {
    read(4, boost::bind(&Connection::onHeaderLengthRead, this, _1, _2, _3, _4));
  }
```

### 服务请求处理

#### 服务注册回调流程

```c++
// 注意用户 cb 的传递, 从 ops 创建 helper 智能指针, 传递给 ServicePublication 保存为 helper_
// -------------------- NodeHandle
  template<class T, class MReq, class MRes>
  ServiceServer advertiseService(const std::string& service, bool(T::*srv_func)(MReq &, MRes &), T *obj)
  {
    AdvertiseServiceOptions ops; // 虽然是栈临时变量,但成员 helper 是智能指针
    ops.template init<MReq, MRes>(service, boost::bind(srv_func, obj, _1, _2));
    return advertiseService(ops);
  }
ServiceServer NodeHandle::advertiseService(AdvertiseServiceOptions& ops)
  if (ServiceManager::instance()->advertiseService(ops))
// -------------------- ServiceManager
bool ServiceManager::advertiseService(const AdvertiseServiceOptions& ops)
    ServicePublicationPtr pub(boost::make_shared<ServicePublication>(ops.service, ops.md5sum, ops.datatype, ops.req_datatype, ops.res_datatype, ops.helper, ops.callback_queue, ops.tracked_object));
// -------------------- ServicePublication
ServicePublication::ServicePublication(const std::string& name, const std::string &md5sum, const std::string& data_type, const std::string& request_data_type,
                             const std::string& response_data_type, const ServiceCallbackHelperPtr& helper, CallbackQueueInterface* callback_queue,
                             const VoidConstPtr& tracked_object)
: name_(name)
, md5sum_(md5sum)
, data_type_(data_type)
, request_data_type_(request_data_type)
, response_data_type_(response_data_type)
, helper_(helper)  // 智能指针 helper 保存在 ServicePublication 对象中
```

#### 智能指针 helper 封装

```c++
// -------------------- AdvertiseServiceOptions
  template<class Service>
  void init(const std::string& _service, const boost::function<bool(typename Service::Request&, typename Service::Response&)>& _callback)
  {
    service = _service;
    md5sum = st::md5sum<Service>();
    datatype = st::datatype<Service>();
    req_datatype = mt::datatype<Request>();
    res_datatype = mt::datatype<Response>();
    // 创建 ServiceCallbackHelperT 智能指针 helper, 封装了用户 cb 和应答消息序列化动作
    helper = boost::make_shared<ServiceCallbackHelperT<ServiceSpec<Request, Response> > >(_callback);
  }
// -------------------- ServiceCallbackHelper
// 构造 helper
template<typename Spec>
class ServiceCallbackHelperT : public ServiceCallbackHelper
  virtual bool call(ServiceCallbackHelperCallParams& params)
    // 传递 callback_
    bool ok = Spec::call(callback_, call_params);
    // 应答消息序列化
    params.response = ser::serializeServiceResponse(ok, *res);
    return ok;
// 构造的 helper 封装了用户的 cb
template<typename MReq, typename MRes>
struct ServiceSpec
{
  static bool call(const CallbackType& cb, ServiceSpecCallParams<RequestType, ResponseType>& params)
  {
    // 回调 cb
    return cb(*params.request, *params.response);
  }
};
```

#### 服务请求处理前半部流程

```c++
// -------------------- ServiceClientLink
bool ServiceClientLink::handleHeader(const Header& header)
  ServicePublicationPtr ss = ServiceManager::instance()->lookupServicePublication(service);
    parent_ = ServicePublicationWPtr(ss);
    M_string m;
    m["request_type"] = ss->getRequestDataType();
    m["response_type"] = ss->getResponseDataType();
    m["type"] = ss->getDataType();
    m["md5sum"] = ss->getMD5Sum();
    m["callerid"] = this_node::getName();
    connection_->writeHeader(m, boost::bind(&ServiceClientLink::onHeaderWritten, this, _1));
    ss->addServiceClientLink(shared_from_this());
void ServiceClientLink::onHeaderWritten(const ConnectionPtr& conn)
  (void)conn;
  connection_->read(4, boost::bind(&ServiceClientLink::onRequestLength, this, _1, _2, _3, _4));
void ServiceClientLink::onRequestLength(const ConnectionPtr& conn, const boost::shared_array<uint8_t>& buffer, uint32_t size, bool success)
  connection_->read(len, boost::bind(&ServiceClientLink::onRequest, this, _1, _2, _3, _4));
void ServiceClientLink::onRequest(const ConnectionPtr& conn, const boost::shared_array<uint8_t>& buffer, uint32_t size, bool success)
  if (ServicePublicationPtr parent = parent_.lock())
  {
    parent->processRequest(buffer, size, shared_from_this());
  }
```

```c++
// -------------------- ServicePublication
// 处理服务请求
void ServicePublication::processRequest(boost::shared_array<uint8_t> buf, size_t num_bytes, const ServiceClientLinkPtr& link)
  CallbackInterfacePtr cb(boost::make_shared<ServiceCallback>(helper_, buf, num_bytes, link, has_tracked_object_, tracked_object_));
  // 放入 callback queue, 调度调用 ServiceCallback 的 call
  callback_queue_->addCallback(cb, (uint64_t)this);
// -------------------- ServiceCallback
class ServiceCallback : public CallbackInterface
  virtual CallResult call()
      // 回调 helper 封装的用户 cb, 封装的 cb 已对应答消息response序列化
      bool ok = helper_->call(params);
      if (ok != 0)
      {
        link_->processResponse(true, params.response);
      }
```

#### 服务请求处理前半部流程

```c++
// -------------------- ServiceCallbackHelper
void ServiceClientLink::processResponse(bool ok, const SerializedMessage& res)
{
  (void)ok;
  connection_->write(res.buf, res.num_bytes, boost::bind(&ServiceClientLink::onResponseWritten, this, _1));
}
// -------------------- Connection
void write(const boost::shared_array<uint8_t>& buffer, uint32_t size, const WriteFinishedFunc& finished_callback, bool immedate = true);
void Connection::write(const boost::shared_array<uint8_t>& buffer, uint32_t size, const WriteFinishedFunc& callback, bool immediate)
{
  {
    boost::mutex::scoped_lock lock(write_callback_mutex_);

    write_callback_ = callback;
    write_buffer_ = buffer;
    write_size_ = size;
    write_sent_ = 0;
    std::atomic_thread_fence(std::memory_order_release);
    has_write_callback_ = 1;
  }

  transport_->enableWrite();

  if (immediate)
  {
    writeTransport();
  }
}
```
```c++
// -------------------- Connection
void Connection::writeTransport()
{
  boost::recursive_mutex::scoped_try_lock lock(write_mutex_);
  if (!lock.owns_lock() || dropped_ || writing_)
  {
    return;
  }

  writing_ = true;
  bool can_write_more = true;

  while (has_write_callback_ && can_write_more && !dropped_)
  {
    uint32_t to_write = write_size_ - write_sent_;
    // 开始发送消息
    int32_t bytes_sent = transport_->write(write_buffer_.get() + write_sent_, to_write);
    if (bytes_sent < 0)
    {
      writing_ = false;
      return;
    }

    write_sent_ += bytes_sent;

    if (bytes_sent < (int)write_size_ - (int)write_sent_)
    {
      can_write_more = false;
    }

    if (write_sent_ == write_size_ && !dropped_)
    {
      WriteFinishedFunc callback;
      {
        boost::mutex::scoped_lock lock(write_callback_mutex_);
        ROS_ASSERT(has_write_callback_);
        // Store off a copy of the callback in case another write() call happens in it
        callback = write_callback_;
        write_callback_ = WriteFinishedFunc();
        write_buffer_ = boost::shared_array<uint8_t>();
        write_sent_ = 0;
        write_size_ = 0;
        has_write_callback_ = 0;
      }
      callback(shared_from_this()); // 发送完毕,回调 onResponseWritten
    }
  }

  {
    boost::mutex::scoped_lock lock(write_callback_mutex_);
    if (!has_write_callback_)
    {
      transport_->disableWrite();
    }
  }

  writing_ = false;
}
```
```c++
int32_t TransportTCP::write(uint8_t* buffer, uint32_t size)
{
  {
    boost::recursive_mutex::scoped_lock lock(close_mutex_);
    if (closed_)
    {
      return -1;
    }
  }
  ROS_ASSERT(size > 0);
  // never write more than INT_MAX since this is the maximum we can report back with the current return type
  uint32_t writesize = std::min(size, static_cast<uint32_t>(INT_MAX));
  int num_bytes = ::send(sock_, reinterpret_cast<const char*>(buffer), writesize, 0);
  if (num_bytes < 0)
  {
    // 发送失败处理
    if ( !last_socket_error_is_would_block() )
    {
      ROS_INFO("send() on socket [%d] failed with error [%s]", sock_, last_socket_error_string());
      close();
    }
    else
    {
      num_bytes = 0;
    }
  }

  return num_bytes;
}
```
```c++
bool last_socket_error_is_would_block() {
	if ( ( errno == EAGAIN  ) || ( errno == EWOULDBLOCK ) ) { // posix permits either
		return true;
	} else {
		return false;
	}
}
```


