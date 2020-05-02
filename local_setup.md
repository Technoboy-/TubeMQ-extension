### 搭建本地环境

#### Master
   1. 修改master.ini文件
      - hostName
      - webResourcePath
      - zkServerAddr
   2. MasterStartup
      ```
      String[] innerArgs = new String[]{"-f", "./conf/master.ini"};
      ```


#### velocity.properties
   1. 修改file.resource.loader.path
   file.resource.loader.path=/Users/tboy/tboy/workspace_github/incubator-tubemq/resources/templates
          
      
#### Broker
   1. 修改broker.ini文件
      - hostName
      - masterAddressList
      - primaryPath
   2. BrokerStartup
      ```
      String[] innerArgs = new String[]{"-f", "./conf/broker.ini"};
      ```         

 #### Class
   1. 修改AddressUtils      
      localIPAddress=localhost
   
   2.    
   
   
   //TODO
   
