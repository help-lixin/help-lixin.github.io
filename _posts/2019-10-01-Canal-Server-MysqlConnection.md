---
layout: post
title: 'Canal Server源码之七(MysqlConnection)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). 查看MysqlConnection类结构图

!["MysqlConnection类结构图"](/assets/canal/imgs/MysqlConnection-uml-class.jpg)

### (2). 概述
> 1. MysqlConnection是ErosaConnection的唯一实现类.
> 2. MysqlConnection拥有一个MysqlConnector
> 3. MysqlConnection根据mysql信息(ip:port,uname,pwd)委托给MysqlConnector进行管理.
> 4. connect/fork/reconnect/disconnect等操作,委托给:MysqlConnector处理.
> 5. MysqlConnection主要实现:seek/dump请求
> 6. 思考:阿里在类的设计方面,一直坚持单一职责,感觉ErosaConnection的设计是否应该要拆开?

### (3). MysqlConnector 测试案例
```
package com.alibaba.otter.canal.parse.driver.mysql;

import java.io.IOException;
import java.net.InetSocketAddress;

import org.junit.Assert;
import org.junit.Ignore;
import org.junit.Test;

import com.alibaba.otter.canal.parse.driver.mysql.packets.server.ResultSetPacket;

//@Ignore
public class MysqlConnectorTest {

    @Test
    public void testQuery() {
        // *********************************************************************
        // 4. 创建:MysqlConnector
        // *********************************************************************
        MysqlConnector connector = new MysqlConnector(new InetSocketAddress("127.0.0.1", 3306), "root", "123456");
        try {
            connector.connect();
            MysqlQueryExecutor executor = new MysqlQueryExecutor(connector);
            ResultSetPacket result = executor.query("show variables like '%char%';");
            System.out.println(result);
        } catch (IOException e) {
            Assert.fail(e.getMessage());
        } finally {
            try {
                connector.disconnect();
            } catch (IOException e) {
                Assert.fail(e.getMessage());
            }
        }
    }//end testQuery
}
```
### (4). MysqlConnector构造器
```
public MysqlConnector(
        // 127.0.0.1:3306
        InetSocketAddress address, 
        // root
        String username, 
        // 123456
        String password){
    String addr = address.getHostString();
    int port = address.getPort();
    this.address = new InetSocketAddress(addr, port);

    this.username = username;
    this.password = password;
}// end MysqlConnector
```
### (5). MysqlConnector.connect
```
public void connect() throws IOException {
    if (connected.compareAndSet(false, true)) { //控制connect只调用一次
        try {
            // ********************************************************
            // address = 127.0.0.1:3306
            // SocketChannelPool.open(address)
            // 6.委托给:SocketChannelPool创建连接
            // ********************************************************
            channel = SocketChannelPool.open(address);

            logger.info("connect MysqlConnection to {}...", address);

            // ********************************************************
            // 7. 进行握手和验证(重点)
            // ********************************************************
            negotiate(channel);

        } catch (Exception e) {
            disconnect();
            throw new IOException("connect " + this.address + " failure", e);
        }
    } else {
        logger.error("the channel can't be connected twice.");
    }
}// end connect
```
### (6). SocketChannelPool.open
> 1. 获取系统参数(canal.socketChannel)   
> 2. 如果参数为:netty,则,委托给:NettySocketChannelPool创建连接.   
> 3. 如果参数为null,则,委托给:BioSocketChannelPool创建连接.  
> 4. SocketChannel阿里自己封装的对象,从接口功能来看,主要提供:读/写,在此处不进里面详细讲解.    

```
public static SocketChannel open(SocketAddress address) throws Exception {
    String type = chooseSocketChannel();
    if ("netty".equalsIgnoreCase(type)) {
        return NettySocketChannelPool.open(address);
    } else {
        return BioSocketChannelPool.open(address);
    }
}


private static String chooseSocketChannel() {
    // 获得系统变量:canal.socketChannel
    String socketChannel = System.getenv("canal.socketChannel");
    if (StringUtils.isEmpty(socketChannel)) {
        socketChannel = System.getProperty("canal.socketChannel");
    }
    if (StringUtils.isEmpty(socketChannel)) {
        socketChannel = "bio"; // bio or netty
    }
    return socketChannel;
}
```
### (7). MysqlConnector.negotiate

```
private void negotiate(SocketChannel channel) throws IOException {
    // https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol

    // ***************************1.读取MySQL的握手协议信息 ***************************
    // 读取Header
    HeaderPacket header = PacketManager.readHeader(channel, 4, timeout);
    // 根据Header中的:packetBodyLength获得body的长度
    byte[] body = PacketManager.readBytes(channel, header.getPacketBodyLength(), timeout);

    // body的第一个元素如果是:0/-2/或者无法访问,抛出异常
    if (body[0] < 0) {// check field_count
        if (body[0] == -1) {
            ErrorPacket error = new ErrorPacket();
            error.fromBytes(body);
            throw new IOException("handshake exception:\n" + error.toString());
        } else if (body[0] == -2) {
            throw new IOException("Unexpected EOF packet at handshake phase.");
        } else {
            throw new IOException("unpexpected packet with field_count=" + body[0]);
        }
    }

    // ***************************2.创建握手协议***************************
    HandshakeInitializationPacket handshakePacket = new HandshakeInitializationPacket();

    // ***************************************************************
    // 对body协议进行解码
    // ***************************************************************
    handshakePacket.fromBytes(body);

    // HandshakeV9处理
    if (handshakePacket.protocolVersion != MSC.DEFAULT_PROTOCOL_VERSION) { //false
        // HandshakeV9
        auth323(channel, (byte) (header.getPacketSequenceNumber() + 1), handshakePacket.seed);
        return;
    }

    
    connectionId = handshakePacket.threadId; // 记录一下connection
    logger.info("handshake initialization packet received, prepare the client authentication packet to send");

    // 客户端创建认证
    ClientAuthenticationPacket clientAuth = new ClientAuthenticationPacket();

    clientAuth.setCharsetNumber(charsetNumber);
    // 设置用户名和密码
    clientAuth.setUsername(username);
    clientAuth.setPassword(password);
    clientAuth.setServerCapabilities(handshakePacket.serverCapabilities);
    clientAuth.setDatabaseName(defaultSchema);
    clientAuth.setScrumbleBuff(joinAndCreateScrumbleBuff(handshakePacket));
    clientAuth.setAuthPluginName("mysql_native_password".getBytes());
    byte[] clientAuthPkgBody = clientAuth.toBytes();

    //创建Header包裹:auth协议信息 
    HeaderPacket h = new HeaderPacket();
    // 设置auth body的长度
    h.setPacketBodyLength(clientAuthPkgBody.length);
    // 获量,mysql要求每次:number都要进行递增
    h.setPacketSequenceNumber((byte) (header.getPacketSequenceNumber() + 1));

    // 向MySQL发送auth请求
    PacketManager.writePkg(channel, h.toBytes(), clientAuthPkgBody);
    logger.info("client authentication packet is sent out.");

    // 重新获取认证后的结果
    // check auth result
    header = null;
    // 读取协议头
    header = PacketManager.readHeader(channel, 4);
    body = null;
    // 协议头里含有协议体长度,根据长度,获得协议体
    body = PacketManager.readBytes(channel, header.getPacketBodyLength(), timeout);
    assert body != null;
    // 获得协议体(数组)中的第一个元素
    byte marker = body[0];

    if (marker == -2 || marker == 1) { // fasle
        byte[] authData = null;
        String pluginName = null;
        if (marker == 1) {
            AuthSwitchRequestMoreData packet = new AuthSwitchRequestMoreData();
            packet.fromBytes(body);
            authData = packet.authData;
        } else {
            AuthSwitchRequestPacket packet = new AuthSwitchRequestPacket();
            packet.fromBytes(body);
            authData = packet.authData;
            pluginName = packet.authName;
        }

        boolean isSha2Password = false;
        byte[] encryptedPassword = null;
        if (pluginName != null && "mysql_native_password".equals(pluginName)) {
            try {
                encryptedPassword = MySQLPasswordEncrypter.scramble411(getPassword().getBytes(), authData);
            } catch (NoSuchAlgorithmException e) {
                throw new RuntimeException("can't encrypt password that will be sent to MySQL server.", e);
            }
        } else if (pluginName != null && "caching_sha2_password".equals(pluginName)) {
            isSha2Password = true;
            try {
                encryptedPassword = MySQLPasswordEncrypter.scrambleCachingSha2(getPassword().getBytes(), authData);
            } catch (DigestException e) {
                throw new RuntimeException("can't encrypt password that will be sent to MySQL server.", e);
            }
        }
        assert encryptedPassword != null;
        AuthSwitchResponsePacket responsePacket = new AuthSwitchResponsePacket();
        responsePacket.authData = encryptedPassword;
        byte[] auth = responsePacket.toBytes();

        h = new HeaderPacket();
        h.setPacketBodyLength(auth.length);
        h.setPacketSequenceNumber((byte) (header.getPacketSequenceNumber() + 1));
        PacketManager.writePkg(channel, h.toBytes(), auth);
        logger.info("auth switch response packet is sent out.");

        header = null;
        header = PacketManager.readHeader(channel, 4);
        body = null;
        body = PacketManager.readBytes(channel, header.getPacketBodyLength(), timeout);
        assert body != null;
        if (isSha2Password) {
            if (body[0] == 0x01 && body[1] == 0x04) {
                // password auth failed
                throw new IOException("caching_sha2_password Auth failed");
            }

            header = null;
            header = PacketManager.readHeader(channel, 4);
            body = null;
            body = PacketManager.readBytes(channel, header.getPacketBodyLength(), timeout);
        }
    }// end else

    if (body[0] < 0) { // false
        if (body[0] == -1) {
            ErrorPacket err = new ErrorPacket();
            err.fromBytes(body);
            throw new IOException("Error When doing Client Authentication:" + err.toString());
        } else {
            throw new IOException("unpexpected packet with field_count=" + body[0]);
        }
    }
} //end negotiate
```
### (8). 总结
> MysqlConnector负责创建连接并与MySQL进行握手.

