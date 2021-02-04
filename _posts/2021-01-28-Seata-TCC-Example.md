---
layout: post
title: 'Seata TCC模式之入门案例(七)'
date: 2021-01-28
author: 李新
tags: Seata Seata-TCC源码
---

### (1). transfer-tcc-sample项目下载并编译
```
$ git clone https://github.com/seata/seata-samples.git
$ cd seata-samples
$ mvn clean install -DskipTests
```

> https://github.com/seata/seata-samples/tree/master/tcc/transfer-tcc-sample

### (2). 导入transfer-tcc-sample到工程
```
# 查看:transfer-tcc-sample项目目录
transfer-tcc-sample
├── README.MD
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── io
│   │   │       └── seata
│   │   │           └── samples
│   │   │               └── tcc
│   │   │                   └── transfer
│   │   │                       ├── ApplicationKeeper.java
│   │   │                       ├── action
│   │   │                       │   ├── FirstTccAction.java                       # 事务参与者A
│   │   │                       │   ├── SecondTccAction.java                      # 事务参与者B
│   │   │                       │   └── impl
│   │   │                       │       ├── FirstTccActionImpl.java
│   │   │                       │       └── SecondTccActionImpl.java
│   │   │                       ├── activity
│   │   │                       │   ├── TransferService.java
│   │   │                       │   └── impl
│   │   │                       │       └── TransferServiceImpl.java           # 转账业务类(Seata中的TM)
│   │   │                       ├── dao
│   │   │                       │   ├── AccountDAO.java
│   │   │                       │   └── impl
│   │   │                       │       └── AccountDAOImpl.java
│   │   │                       ├── domains
│   │   │                       │   └── Account.java
│   │   │                       ├── env
│   │   │                       │   └── TransferDataPrepares.java         # 数据初始化
│   │   │                       └── starter
│   │   │                           ├── TransferApplication.java          # 程序入口
│   │   │                           └── TransferProviderStarter.java
│   │   └── resources
│   │       ├── db-bean
│   │       │   ├── from-datasource-bean.xml                        # 事务参与者A的配置(数据源)
│   │       │   └── to-datasource-bean.xml                          # 事务参与者B的数据(数据源)
│   │       ├── file.conf                                           # seata(tm)持久化配置
│   │       ├── registry.conf                                       # seata注册中心配置
│   │       ├── spring
│   │       │   ├── seata-dubbo-provider.xml                        # dubbo提供者
│   │       │   ├── seata-dubbo-reference.xml                       # dubbo消费者
│   │       │   └── seata-tcc.xml
│   │       └── sqlmap
│   │           ├── account.xml
│   │           └── sqlMapConfig.xml
│   └── test
│       └── java
│           └── io
│               └── seata
│                   └── samples
│                       └── tcc
│                           └── SeataServerStarter.java
└── target
```
### (3). 配置TM(并启动)

> 1. 配置:registry.conf为:file模式.

```
lixin-macbook:conf lixin$ more  registry.conf
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  # TODO lixin
  type = "file"
  # type = "eureka"
  loadBalance = "RandomLoadBalance"
  loadBalanceVirtualNodes = 10
```

> 2. 配置:file.conf为:file模式.

```
lixin-macbook:conf lixin$ more  file.conf
## transaction log store, only used in seata-server
store {
  ## store mode: file、db、redis
  # mode = "db"
  # TODO lixin
  mode = "file"
```

> 3. 启动TM

```
lixin-macbook:seata-server-1.4.0 lixin$ pwd
/Users/lixin/Developer/seata-server-1.4.0

lixin-macbook:seata-server-1.4.0 lixin$ ./bin/seata-server.sh
# 显示启动成功
15:40:58.818  INFO --- [                     main] i.s.core.rpc.netty.NettyServerBootstrap  : Server started, listen port: 8091
```

### (4). 启动提供者(TransferProviderStarter)
> 稍微的分析下:TransferProviderStarter的代码.

```
public class TransferProviderStarter {

    private static TestingServer server;

    public static void main(String[] args) throws Exception {
		// 1. 提供者模拟启动ZK,这一点Seata做得还是不错的,因为尽可能把外置依赖降到最低.
        //mock zk server
        mockZKServer();

		
		// spring/seata-tcc.xml                      这个XML申明Spring中的类信息
		// spring/seata-dubbo-provider.xml           这个XML申明Dubbo暴露哪些接口
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext(
            new String[] {"spring/seata-tcc.xml", "spring/seata-dubbo-provider.xml",
                "db-bean/to-datasource-bean.xml", "db-bean/from-datasource-bean.xml"});

        // 初始化数据库和账号余额
		// 把数据库的依赖都降到最低为:h2
		// 值得开发人员学习,尽可能让测试代码把业务逻辑都能覆盖到位
        TransferDataPrepares transferDataPrepares = (TransferDataPrepares) applicationContext.getBean("transferDataPrepares");
        transferDataPrepares.init(100);

		// Hood住线程不关闭.
        new ApplicationKeeper(applicationContext).keep();
    }

	// 模似ZK
    private static void mockZKServer() throws Exception {
        //Mock zk server，作为 transfer 配置中心
        server = new TestingServer(2181, true);
        server.start();
    }
}
```
### (5). 启动消费者(TransferApplication)并测试
```
package io.seata.samples.tcc.transfer.starter;

import java.sql.SQLException;

import io.seata.samples.tcc.transfer.activity.TransferService;
import io.seata.samples.tcc.transfer.dao.AccountDAO;
import io.seata.samples.tcc.transfer.env.TransferDataPrepares;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * 发起转账
 *
 * @author zhangsen
 */
public class TransferApplication {

    protected static ApplicationContext applicationContext ;

    /**
     * 转账服务
     */
    protected static TransferService transferService ;

    /**
     * 转出账户数据 DAO
     */
    protected static AccountDAO fromAccountDAO;

    /**
     * 转入账户数据 DAO
     */
    protected static AccountDAO toAccountDAO;

    /**
     * 转账数据初始化
     */
    protected static TransferDataPrepares transferDataPrepares;

    public static void main(String[] args) throws SQLException {
        applicationContext = new ClassPathXmlApplicationContext("spring/seata-tcc.xml",
            "spring/seata-dubbo-reference.xml",
                "db-bean/to-datasource-bean.xml", "db-bean/from-datasource-bean.xml");

        transferService = (TransferService) applicationContext.getBean("transferService" );
        fromAccountDAO = (AccountDAO) applicationContext.getBean("fromAccountDAO" );
        toAccountDAO = (AccountDAO) applicationContext.getBean("toAccountDAO" );

        //执行 A->C 转账成功 demo, 分布式事务提交
        doTransferSuccess(100, 10);

        //执行 B->XXX 转账失败 demo， 分布式事务回滚
        // doTransferFailed(100, 10);
    }

    /**
     * 执行转账成功 demo
     *
     * @param initAmount 初始化余额
     * @param transferAmount  转账余额
     */
    private static void doTransferSuccess(double initAmount, double transferAmount) throws SQLException {
        //执行转账操作
        doTransfer("A", "C", transferAmount);

        //校验A账户余额：initAmount - transferAmount
        checkAmount(fromAccountDAO, "A", initAmount - transferAmount);

        //校验C账户余额：initAmount + transferAmount
        checkAmount(toAccountDAO, "C", initAmount + transferAmount);
    }

    /**
     * 执行转账 失败 demo， 'B' 向未知用户 'XXX' 转账，转账失败分布式事务回滚
     * @param initAmount 初始化余额
     * @param transferAmount  转账余额
     */
    private static void doTransferFailed(int initAmount, int transferAmount) throws SQLException {
        // 'B' 向未知用户 'XXX' 转账，转账失败分布式事务回滚
        try{
            doTransfer("B", "XXX", transferAmount);
        }catch (Throwable t){
            System.out.println("从账户B向未知账号XXX转账失败.");
        }

        //校验A2账户余额：initAmount
        checkAmount(fromAccountDAO, "B", initAmount);

        //账户XXX 不存在，无需校验
    }

    /**
     * 执行转账 操作
     * @param transferAmount 转账金额
     */
    private static boolean doTransfer(String from, String to, double transferAmount) {
        //转账操作
        boolean ret = transferService.transfer(from, to, transferAmount);
        if(ret){
            System.out.println("从账户"+from+"向"+to+"转账 "+transferAmount+"元 成功.");
            System.out.println();
        }else {
            System.out.println("从账户"+from+"向"+to+"转账 "+transferAmount+"元 失败.");
            System.out.println();
        }
        return ret;
    }


    /**
     * 校验账户余额
     * @param accountDAO
     * @param accountNo
     * @param expectedAmount
     * @throws SQLException
     */
    private static void checkAmount(AccountDAO accountDAO, String accountNo, double expectedAmount) throws SQLException {
        try {
//            Account account = accountDAO.getAccount(accountNo);
//            Assert.isTrue(account != null, "账户不存在");
//            double amount = account.getAmount();
//            double freezedAmount = account.getFreezedAmount();
//            Assert.isTrue(expectedAmount == amount, "账户余额校验失败");
//            Assert.isTrue(freezedAmount == 0, "账户冻结余额校验失败");
        }catch (Throwable t){
            t.printStackTrace();
        }
    }
}
```
### (6). 查看转账结果
```
# 在消费者提示这个代表转账成功.
transfer amount[10.0] from [A] to [C] finish.
[2m16:13:50.053[0;39m [32m INFO[0;39m [2m---[0;39m [2m[                     main][0;39m [36mi.seata.tm.api.DefaultGlobalTransaction [0;39m [2m:[0;39m [172.17.23.50:8091:100264119163363328] commit status: Committed
从账户A向C转账 10.0元 成功.
```