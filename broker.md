#### Broker
  1. BrokerMetadataManager
     <br>Broker的元数据管理类，包含了broker的配置信息，主题，以及一些策略定义。broker通过与master的心跳来不断更新元数据信息。
     - broker启动后，调用register2Master方法，
       ```
        public void start() throws Exception {
          logger.info("Starting tube server...");
          if (!this.shutdown.get()) {
              return;
          }
          this.shutdown.set(false);
          // register to master, and heartbeat to master.
          this.register2Master();
        }
       ```
     -  当broker注册成功后，将调用metadataManager.updateBrokerTopicConfigMap()方法更新元数据信息。
       ```
       metadataManager.updateBrokerTopicConfigMap(response.getCurBrokerConfId(),
                           response.getConfCheckSumId(), response.getBrokerDefaultConfInfo(),
                           response.getBrokerTopicSetConfInfoList(), true, sBuilder);
       ```
     - 当注册失败且重试5次后，抛出StartupException，broker启动失败。  
     - 注册成功后，broker将周期性按照默认8s的间隔，向master发送心跳信息。当有flowctl或者topic变更时，将更新metadataManager。
 
 
 
