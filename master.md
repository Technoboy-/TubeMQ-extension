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
 
 
 
