---
layout: post
title: 'Lily HBase Indexer(一)'
date: 2021-04-06
author: 李新
tags:  HBase源码 解决方案
---

### (1). Lily HBase Indexer是什么?
> Lily HBase Indexer是由NGDATA公司开发,用于近实时的同步:HBase的数据到Solor中.   
> 当HBase执行写入/更新/删除操作时,Indexer通过HBase的Replication功能,把这些操作抽象成一系列的Event,并用来保证写入Solr中的Hbase索引数据的一致性. 
### (2). Lily HBase Indexer流程图
!["Lily HBase Indexer架构图"](/assets/hbase/imgs/HBase-Indexer.png)
### (3). 为什么选择Replication而不选择Coprocessor来实现HBase Indexer？ 
> 1）HBase Replication的处理是由RegionServer开启独立的线程去处理的,处理方式是并行且异步的,依靠这种机制来实现HBase Indexer并不会给HBase带来入侵式的代码,而且不会影响写入性能.而通过Coprocessor来实现的话会给RegionServer带来入侵式代码,以及阻碍HBase的正常操作.  
> 2）虽然选择Replication机制只能实现近实时的索引同步,但是这种实现方式具备很高的灵活性和可扩展性,最重要的是它对HBase集群的使用是几乎没有侵占性的,不会影响HBase集群的写性能. 
> 3）你可以理解成:HBase Replication的处理方式,其实和MySQL Binlog同步是一样的.  
> 4）原理是什么?当执行增/删/改时,RegionServer会包装成Event,以推送的方式发送给:Hbase Indexer.   
> 5）推送模式下,如何保证消可靠性?HBase Indexer在消费时,是会向ZK提交commit的.  
### (4). Lily HBase Indexer有什么不足?
> Lily HBase Indexer源码,而且,已经多年不维护了,而且,目前只支持Solr(这是我认为的严重不足点).
> 其实,站在架构的角度来说:不论HBase同步数据到任何存储设备,应该抽象出一层存储引擎层,而具体的实现是什么其实不太重要,但是在存储层,Lily HBase Indexer把代码写死了.  
> 我原本的想法是对:HBase Indexer进行扩展,抽象出一层:存储引擎层,但是,发现代码里严重依赖:Solr,所以,就抽出HBase Index对Event的解析层,同步到Solr的代码自己写.  
### (5). 步骤
> 1. 从git上clone一份Lily HBase Indexer源码.  
> 2. 修改pom.xml,升级hadoop(2.7.5)和hbase(1.4.13).   
> 3. 抽出hbase-sep项目,自行扩展.  
### (6). 项目结构
!["HBase Sep项目结构"](/assets/hbase/imgs/HBase-sep.png)
### (7). 集成步骤
```
# 1. 把hbase-sep-api-1.6-SNAPSHOT.jar和hbase-sep-impl-1.6-SNAPSHOT.jar拷贝到HBase/lib目录下
cp hbase-sep-api/target/hbase-sep-api-1.6-SNAPSHOT.jar     /Users/lixin/Developer/hbase-1.4.13/lib/
cp hbase-sep-tools/target/hbase-sep-tools-1.6-SNAPSHOT.jar /Users/lixin/Developer/hbase-1.4.13/lib/
cp hbase-sep-impl/target/hbase-sep-impl-1.6-SNAPSHOT.jar   /Users/lixin/Developer/hbase-1.4.13/lib/

#  配置HBase/conf/hbase-site.xml
<configuration>
	<!-- 开启集群模式 -->
	<property>
	  <name>hbase.cluster.distributed</name>
	  <value>true</value>
	</property>

	<!-- HDFS存储路径 -->
	<property>
	  <name>hbase.rootdir</name>
	  <value>hdfs://lixin-macbook.local:9000/hbase</value>
	</property>

	<!-- *开启复制* -->
	<property>
	   <name>hbase.replication</name>
	   <value>true</value>
	</property>

	<!-- *只允许一个Hase Indexer进行复制* -->
	<property>
		<name>replication.source.ratio</name>
		<value>1.0</value>
	</property>
	
	<property>
		<name>replication.source.nb.capacity</name>
		<value>1000</value>
	</property>
	
	<!-- 配置ReplicationSource -->
	<property>
	   <name>replication.replicationsource.implementation</name>
	   <value>com.ngdata.sep.impl.SepReplicationSource</value>
	</property>
</configuration>

# 3. 启动HBase

# 4. 创建表和列簇(*注意:在列簇上要开启复制模式*)
public class DemoSchema {
    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        createSchema(conf);
    }

    public static void createSchema(Configuration hbaseConf) throws IOException {
        Admin admin = ConnectionFactory.createConnection(hbaseConf).getAdmin();
        if (!admin.tableExists(TableName.valueOf("sep-user-demo"))) {
            HTableDescriptor tableDescriptor = new HTableDescriptor(TableName.valueOf("sep-user-demo"));

            HColumnDescriptor infoCf = new HColumnDescriptor("info");
			// ***************************************
			// 开启复制模式
			// ***************************************
            infoCf.setScope(1);
            tableDescriptor.addFamily(infoCf);

            admin.createTable(tableDescriptor);
        }
        admin.close();
    }
}

# 5. 运行,HBase Index等待触发事件.
public class LoggingConsumer {
    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        conf.setBoolean("hbase.replication", true);

        ZooKeeperItf zk = ZkUtil.connect("localhost", 20000);
        SepModel sepModel = new SepModelImpl(zk, conf);

        final String subscriptionName = "logger";

        if (!sepModel.hasSubscription(subscriptionName)) {
            sepModel.addSubscriptionSilent(subscriptionName);
        }

        PayloadExtractor payloadExtractor = new BasePayloadExtractor(Bytes.toBytes("sep-user-demo"), Bytes.toBytes("info"),
                Bytes.toBytes("payload"));

        SepConsumer sepConsumer = new SepConsumer(subscriptionName, 0, new EventLogger(), 1, "localhost", zk, conf,
                payloadExtractor);

        sepConsumer.start();
        System.out.println("Started");

        while (true) {
            Thread.sleep(Long.MAX_VALUE);
        }
    }

    private static class EventLogger implements EventListener {
        @Override
        public void processEvents(List<SepEvent> sepEvents) {
            for (SepEvent sepEvent : sepEvents) {
                System.out.println("Received event:");
                System.out.println("  table = " + Bytes.toString(sepEvent.getTable()));
                System.out.println("  row = " + Bytes.toString(sepEvent.getRow()));
                System.out.println("  payload = " + Bytes.toString(sepEvent.getPayload()));
                System.out.println("  key values = ");
                for (Cell kv : sepEvent.getKeyValues()) {
                    System.out.println("    " + kv.toString());
                }
            }
        }
    }
}


# 6. 增加数据,看是否会触发上面的代码.
public class DemoIngester {
    private List<String> names;
    private List<String> domains;

    public static void main(String[] args) throws Exception {
        new DemoIngester().run();
    }

    public void run() throws Exception {
        Configuration conf = HBaseConfiguration.create();

        DemoSchema.createSchema(conf);

        final byte[] infoCf = Bytes.toBytes("info");

        // column qualifiers
        final byte[] nameCq = Bytes.toBytes("name");
        final byte[] emailCq = Bytes.toBytes("email");
        final byte[] ageCq = Bytes.toBytes("age");
        final byte[] payloadCq = Bytes.toBytes("payload");

        loadData();

        ObjectMapper jsonMapper = new ObjectMapper();

        Table htable = ConnectionFactory.createConnection(conf).getTable(TableName.valueOf("sep-user-demo"));

        while (true) {
            byte[] rowkey = Bytes.toBytes(UUID.randomUUID().toString());
            Put put = new Put(rowkey);

            String name = pickName();
            String email = name.toLowerCase() + "@" + pickDomain();
            String age = String.valueOf((int) Math.ceil(Math.random() * 100));

            put.addColumn(infoCf, nameCq, Bytes.toBytes(name));
            put.addColumn(infoCf, emailCq, Bytes.toBytes(email));
            put.addColumn(infoCf, ageCq, Bytes.toBytes(age));

            MyPayload payload = new MyPayload();
            payload.setPartialUpdate(false);
            put.addColumn(infoCf, payloadCq, jsonMapper.writeValueAsBytes(payload));

            htable.put(put);
            System.out.println("Added row " + Bytes.toString(rowkey));
        }
    }

    private String pickName() {
        return names.get((int)Math.floor(Math.random() * names.size()));
    }

    private String pickDomain() {
        return domains.get((int)Math.floor(Math.random() * domains.size()));
    }

    private void loadData() throws IOException {
        // Names
        BufferedReader reader =
                new BufferedReader(new InputStreamReader(getClass().getResourceAsStream("names/names.txt")));

        names = new ArrayList<String>();

        String line;
        while ((line = reader.readLine()) != null) {
            names.add(line);
        }

        // Domains
        domains = new ArrayList<String>();
        domains.add("gmail.com");
        domains.add("hotmail.com");
        domains.add("yahoo.com");
        domains.add("live.com");
        domains.add("ngdata.com");
    }
}
```
### (8). 验证结果
```
# 1. 查看表结构
hbase(main):010:0> describe 'sep-user-demo'

Table sep-user-demo is ENABLED
sep-user-demo
COLUMN FAMILIES DESCRIPTION
{
	NAME => 'info', 
	// ... ..
	// ********************重点 ********************
	REPLICATION_SCOPE => '1'
}

# 2. 随便找个rowkey查看数据.
hbase(main):013:0> get 'sep-user-demo','ef7001ab-10ed-43c4-b83d-d4a8fee6ef78'
COLUMN                          CELL
 info:age                       timestamp=1617836944705, value=99
 info:email                     timestamp=1617836944705, value=maye@gmail.com
 info:name                      timestamp=1617836944705, value=Maye
 info:payload                   timestamp=1617836944705, value={"partialUpdate":false}
4 row(s) in 0.1420 seconds
```

!["HBase Indexer监听结果"](/assets/hbase/imgs/HBase-Index-Console.png)
