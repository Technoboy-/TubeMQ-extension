#### Producer#publish
  1.  以publish(final String topic)为例:
   ```
      public synchronized void publish(final String topic) throws TubeClientException {
           checkServiceStatus();
           StringBuilder sBuilder = new StringBuilder(512);
           try {
               logger.info(sBuilder.append("[Publish begin 1] publish topic ")
                       .append(topic).append(", address = ")
                       .append(this.toString()).toString());
               sBuilder.delete(0, sBuilder.length());
               AtomicInteger curPubCnt = this.publishTopics.get(topic);
               if (curPubCnt == null) {
                   AtomicInteger tmpPubCnt = new AtomicInteger(0);
                   curPubCnt = this.publishTopics.putIfAbsent(topic, tmpPubCnt);
                   if (curPubCnt == null) {
                       curPubCnt = tmpPubCnt;
                   }
               }
               if (curPubCnt.incrementAndGet() == 1) {
                   long curTime = System.currentTimeMillis();
                   //1
                   new ProducerHeartbeatTask().run();
                   logger.info(sBuilder
                           .append("[Publish begin 1] already get meta info, topic: ")
                           .append(topic).append(", waste time ")
                           .append(System.currentTimeMillis() - curTime).append(" Ms").toString());
                   sBuilder.delete(0, sBuilder.length());
               }
               if (topicPartitionMap.get(topic) == null) {
                   throw new TubeClientException(sBuilder
                           .append("Publish topic failure, make sure the topic ")
                           .append(topic).append(" exist or acceptPublish and try later!").toString());
               }
           } finally {
               if (isStartHeart.compareAndSet(false, true)) {
                   heartbeatService.scheduleWithFixedDelay(new ProducerHeartbeatTask(), 5L,
                           tubeClientConfig.getHeartbeatPeriodMs(), TimeUnit.MILLISECONDS);
               }
           }
      }
   ```
   - 发布topic，其实是通过heartbeat进行的。finally处可以看到，最后会周期性的执行ProducerHeartbeatTask,间隔13s。
   2. ProducerHeartbeatTask就是向master不断发送心跳信息，维护连接存在。心跳信息，包含如下内如：
      ```
       private ClientMaster.HeartRequestP2M createHeartbeatRequest() throws Exception {
           ClientMaster.HeartRequestP2M.Builder builder =
                   ClientMaster.HeartRequestP2M.newBuilder();
           builder.setClientId(producerId);
           builder.addAllTopicList(publishTopics.keySet());
           builder.setBrokerCheckSum(this.brokerInfoCheckSum);
           if ((System.currentTimeMillis() - this.lastBrokerUpdatedTime)
                   > BROKER_UPDATED_TIME_AFTER_RETRY_FAIL) {
               builder.setBrokerCheckSum(-1L);
               this.lastBrokerUpdatedTime = System.currentTimeMillis();
           }
           builder.setHostName(AddressUtils.getLocalAddress());
           ClientMaster.MasterCertificateInfo.Builder authInfoBuilder =
                   genMasterCertificateInfo(true);
           if (authInfoBuilder != null) {
               builder.setAuthInfo(authInfoBuilder.build());
           }
           return builder.build();
       }
      ```
   3. 对于master来说，producer的超时时间,默认为30s，可以通过master.ini进行设置
      - producerHeartbeatTimeoutMs=45000
#### Producer#sendMessage
  1. 检查消息checkMessageAndStatus：
   ```
      private void checkMessageAndStatus(final Message message) throws TubeClientException {
          //1
          if (message == null) {
              throw new TubeClientException("Illegal parameter: null message package!");
          }
          if (TStringUtils.isBlank(message.getTopic())) {
              throw new TubeClientException("Illegal parameter: blank topic in message package!");
          }
          if ((message.getData() == null)
                  || (message.getData().length == 0)) {
              throw new TubeClientException("Illegal parameter: null data in message package!");
          }
          int msgSize = TStringUtils.isBlank(message.getAttribute())
                  ? message.getData().length : (message.getData().length + message.getAttribute().length());
          //2
          if (msgSize > TBaseConstants.META_MAX_MESSAGEG_DATA_SIZE) {
              throw new TubeClientException(new StringBuilder(512)
                      .append("Illegal parameter: over max message length for the total size of")
                      .append(" message data and attribute, allowed size is ")
                      .append(TBaseConstants.META_MAX_MESSAGEG_DATA_SIZE)
                      .append(", message's real size is ").append(msgSize).toString());
          }
          if (isShutDown.get()) {
              throw new TubeClientException("Status error: producer has been shutdown!");
          }
          //3
          if (this.publishTopicMap.get(message.getTopic()) == null) {
              throw new TubeClientException(new StringBuilder(512)
                      .append("Topic ").append(message.getTopic())
                      .append(" not publish, please publish first!").toString());
          }
          //4
          if (this.producerManager.getTopicPartition(message.getTopic()) == null) {
              throw new TubeClientException(new StringBuilder(512)
                      .append("Topic ").append(message.getTopic())
                      .append(" not publish, make sure the topic exist or acceptPublish and try later!").toString());
          }
      }
   ```
   - 消息不能为空/topic不能为空/data不能为空。
   - 消息最大为TBaseConstants.META_MAX_MESSAGEG_DATA_SIZE(1024*1024), 即1M。
   - 消息发送前，需要调用publishTopic方法。
   - topic需要存在且设置为允许-可发布。

  2. 选择分区selectPartition
     ```
       private Partition selectPartition(final Message message,
                                         Class clazz) throws TubeClientException {
           String topic = message.getTopic();
           StringBuilder sBuilder = new StringBuilder(512);
           //1
           Map<Integer, List<Partition>> brokerPartList =
                   this.producerManager.getTopicPartition(topic);
           if (brokerPartList == null || brokerPartList.isEmpty()) {
               throw new TubeClientException(sBuilder.append("Null partition for topic: ")
                       .append(message.getTopic()).append(", please try later!").toString());
           }
           //2
           List<Partition> partList =
                   this.brokerRcvQltyStats.getAllowedBrokerPartitions(brokerPartList);
           if (partList == null || partList.isEmpty()) {
               throw new TubeClientException(sBuilder.append("No available partition for topic: ")
                       .append(message.getTopic()).toString());
           }
           //3
           Partition partition =
                   this.partitionRouter.getPartition(message, partList);
           if (partition == null) {
               throw new TubeClientException(new StringBuilder(512)
                       .append("Not found available partition for topic: ")
                       .append(message.getTopic()).toString());
           }
           //4
           if (rpcServiceFactory.isServiceEmpty()) {
               return partition;
           }
           BrokerInfo brokerInfo = partition.getBroker();
           int count = 0;
           while (count++ < partList.size()) {
               if (rpcServiceFactory.getOrCreateService(clazz, brokerInfo, rpcConfig) != null) {
                   break;
               }
               partition = this.partitionRouter.getPartition(message, partList);
               brokerInfo = partition.getBroker();
           }
           return partition;
       }
   
     ```
     - 获取所有包含指定topic的broker列表。key为broker id， value为分区列表。
     - 选择那些允许topic发布的broker，并返回列表。
     - 选择分区，默认实现为RoundRobinPartitionRouter，即轮询策略。
     - 请注意，对于同步sendMessage，传入的clazz为BrokerWriteService，异步为BrokerWriteService.AsyncService。
     - TODO
    
  3. RoundRobinPartitionRouter说明
     ```
      public Partition getPartition(final Message message, final List<Partition> partitions) throws TubeClientException {
          if (partitions == null || partitions.isEmpty()) {
              throw new TubeClientException(new StringBuilder(512)
                      .append("No available partition for topic: ")
                      .append(message.getTopic()).toString());
          }
          AtomicInteger currRouterCount = partitionRouterMap.get(message.getTopic());
          if (null == currRouterCount) {
              AtomicInteger newCounter = new AtomicInteger(0);
              currRouterCount = partitionRouterMap.putIfAbsent(message.getTopic(), newCounter);
              if (null == currRouterCount) {
                  currRouterCount = newCounter;
              }
          }
          Partition roundPartition = null;
          int partSize = partitions.size();
          for (int i = 0; i < partSize; i++) {
              roundPartition = partitions.get((currRouterCount.incrementAndGet() & Integer.MAX_VALUE) % partSize);
              //1
              if (roundPartition != null && roundPartition.getDelayTimeStamp() < System.currentTimeMillis()) {
                  return roundPartition;
              }
          }
          //2
          return partitions.get((steppedCounter.incrementAndGet() & Integer.MAX_VALUE) % partSize);
      }
     ```
     - 如果当前分区发送失败超过3次后，会跳过当前分区。这里的roundPartition.getDelayTimeStamp()，在消息发送失败且大于3次时会延迟delayTimeStamp：
     ```
     public void handleError(Throwable error) {
         partition.increRetries(1);
         brokerRcvQltyStats.addReceiveStatistic(brokerId, false);
         cb.onException(error);
     }
     public void increRetries(int retries) {
         this.retries += retries;
         if (this.retries > 3) {
             this.delayTimeStamp = System.currentTimeMillis() + 3 * 1000;
             this.retries = 0;
         }
     } 
     ```
     - 如果所有分区在短时内都发送失败，将进入外层轮询选择分区。
    

  3. sendMessage

 
 
 
