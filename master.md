#### Master

1. ZkOffsetStorage
   ```
   public ZkOffsetStorage(final ZKConfig zkConfig) {
       this.zkConfig = zkConfig;
       this.tubeZkRoot = normalize(this.zkConfig.getZkNodeRoot());
       this.consumerZkDir = this.tubeZkRoot + "/consumers-v3";
       try {
           this.zkw = new ZooKeeperWatcher(zkConfig);
       } catch (Throwable e) {
           logger.error(new StringBuilder(256)
                   .append("Failed to connect ZooKeeper server (")
                   .append(this.zkConfig.getZkServerAddr()).append(") !").toString(), e);
           System.exit(1);
       }
       logger.info("ZooKeeper Offset Storage initiated!");
       }
   ```
   构造方法可以看出，在zk下/tubemq/consumers-v3为消费者的位移目录。
   ```
   String offsetPath = sb.append(this.consumerZkDir).append("/")
       .append(group).append("/offsets/").append(topic).append("/")
       .append(info.getBrokerId()).append(TokenConstants.HYPHEN)
       .append(info.getPartitionId()).toString();
   ```
   从offsetPath可以了解到，位移路劲类似：/tubemq/consumers-v3/tboy/offsets/test/2130706433-0
   ```
   String offsetData =
       sb.append(msgId).append(TokenConstants.HYPHEN).append(newOffset).toString();
   ```
   offset的值为以上形式。
 
2. DefaultBdbStoreService
    - 将broker，consumer，producer等元数据信息保存到BekerleyDB中的存储服务。
   从DefaultBdbStoreService构造方法可以看出，TubeMQ使用了ReplicationGroupAdmin，这是BekerleyDB的集群模式。集群模式，存在一主多从，master负责读写
   ，从实例负责读操作。
      ```
        private void initEnvConfig() throws InterruptedException {
           repConfig = new ReplicationConfig();
           // #1
           TimeConsistencyPolicy consistencyPolicy = new TimeConsistencyPolicy(3, TimeUnit.SECONDS,
                   3, TimeUnit.SECONDS);
           repConfig.setConsistencyPolicy(consistencyPolicy);
           // Wait up to 3 seconds for commitConsumed acknowledgments.
           repConfig.setReplicaAckTimeout(3, TimeUnit.SECONDS);
           repConfig.setConfigParam(ReplicationConfig.TXN_ROLLBACK_LIMIT, "1000");
           repConfig.setGroupName(bdbConfig.getBdbRepGroupName());
           repConfig.setNodeName(bdbConfig.getBdbNodeName());
           repConfig.setNodeHostPort(this.nodeHost + TokenConstants.ATTR_SEP
                   + bdbConfig.getBdbNodePort());
           if (TStringUtils.isNotEmpty(bdbConfig.getBdbHelperHost())) {
               logger.info("ADD HELP HOST");
               repConfig.setHelperHosts(bdbConfig.getBdbHelperHost());
           }
    
           //A replicated environment must be opened with transactions enabled. Environments on a master
           //must be read/write, while environments on a client can be read/write or read/only. Since the
           //master's identity may change, it's most convenient to open the environment in the default
           //read/write mode. All write operations will be refused on the client though.
           envConfig = new EnvironmentConfig();
           envConfig.setTransactional(true);
           Durability durability =
                   new Durability(bdbConfig.getBdbLocalSync(), bdbConfig.getBdbReplicaSync(),
                           bdbConfig.getBdbReplicaAck());
           envConfig.setDurability(durability);
           envConfig.setAllowCreate(true);
    
           envHome = new File(bdbConfig.getBdbEnvHome());
    
           // An Entity Store in a replicated environment must be transactional.
           storeConfig.setTransactional(true);
           // Note that both Master and Replica open the store for write.
           storeConfig.setReadOnly(false);
           storeConfig.setAllowCreate(true);
        }
      ``` 
      - 表示slave和master需要保持3s的同步窗口。
   
    - 由于master可能在运行中，进行切换，所以，需要设置listener监听。
      ```
      repEnv.setStateChangeListener(listener);
      ```
        
    
 
