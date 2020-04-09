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
    
2. heartbeat response
  ```
 heartbeat response : success: true
 errCode: 200
 errMsg: "OK!"
 brokerCheckSum: 1586266138478
 topicInfos: "test#2130706433:3:1"
 topicInfos: "tboy#2130706433:4:1"
 authorizedInfo {
   visitAuthorizedToken: 1586269735059
 }
 ```

 3. sendMessage

 
 
