---
layout: post
title: 'Redis秒杀设计'
date: 2021-07-03
author: 李新
tags:  Redis
---

### (1). 需求分析
+ 瞬间大量的刷新页面的操作
+ 瞬间大量的抢宝的操作
+ 可能有秒杀器的恶性竞争

### (2). 总体思路
+ CDN
  - Nginx进行动静分离.
  - 静态资源推送到CDN.  
+ 安全保护
  - [隐藏秒杀链接](https://www.cnblogs.com/myseries/p/11891132.html)  
  - 限制IP每秒请求.  
  - 限流(压测出机器的性能拐点,配置好QPS限流).  
+ 热点分离
  - 秒杀设计独立的域名和工程,不能影响其它的业务.
  - 秒杀设计弹性资源的分配.
+ 缓存
  - 在秒杀前,把秒杀的SKU和库存,预热到Redis中.  
  - 通过Lua+Redis(原子性)实现秒杀.
+ 异步削峰
  - Redis扣库存成功后,调用(生产者)MQ创建订单,并,返回一个唯一ID给前端.   
  - MQ(消费者)订阅信息,根据自身的能力,创建订单/扣库存...   
  - 前端通过ID轮询,或者,等待websocket通知:秒杀成功,跳转到支付页面.   
### (3). Lua脚本
> 在GitHub上搜了下秒杀项目,发现的[这个项目](https://github.com/liuhongdi/seconddemo)秒杀脚本,在业务需求覆盖还是比较全面的,特意摘抄出来.   

```
-- 用户ID
local userId = KEYS[1]
-- 购买数量
local buyNum = tonumber(KEYS[2])

-- skuID
local skuId = KEYS[3]
-- 限购数量
local perSkuLim = tonumber(KEYS[4])

-- 活动ID
local actId = KEYS[5]
-- 活动最大数量(外界传入)
local perActLim = tonumber(KEYS[6])
-- 订单时间
local orderTime = KEYS[7]

--用到的各个hash
-- 某个活动下的SKU数量记录(field=skuId value=N)
local sku_amount_hash = 'sec_'..actId..'_sku_amount_hash'
-- 用户参与秒杀活动记录(field="userId+actId"  value=N)
local user_act_hash = 'sec_'..actId..'_u_act_hash'
-- 用户参与秒杀商品记录(field="userId+skuId" value=N)
local user_sku_hash = 'sec_'..actId..'_u_sku_hash'
--
local second_log_hash = 'sec_'..actId..'_log_hash'

-- skuAmountStr : 某个活动下的某个SKU的库存数量
--当前sku是否还有库存
local skuAmountStr = redis.call('hget',sku_amount_hash,skuId)
if skuAmountStr == false then
    -- skuAmountStr为false的情况下,代表这个SKU压根就没有参与活动,直接返回:-3
    redis.log(redis.LOG_NOTICE,'skuAmount is nil ')
    return '-3'
end;

-- skuAmount : 某个活动下的sku库存
local skuAmount = tonumber(skuAmountStr)
redis.log(redis.LOG_NOTICE,'sku:'..skuId..';skuAmount:'..skuAmount)

-- 检查库存,当库存不足的时候,直接返回
if skuAmount <= 0 then
   return '0'
end

-------------------------------------用户参与秒杀活动限制代码块-------------------------------------
redis.log(redis.LOG_NOTICE,'perActLim:'..perActLim)
-- 对用户参与的活动进行限制
local userActKey = userId..'_'..actId
--当前用户已购买此活动多少件
-- perActLim > 0 : 代表开启了用户参与活动的限制
 if perActLim > 0 then
   local userActNumInt = 0
   local userActNum = redis.call('hget',user_act_hash,userActKey)
   if userActNum == false then
      redis.log(redis.LOG_NOTICE,'userActKey:'..userActKey..' is nil')
      userActNumInt = buyNum
   else
      redis.log(redis.LOG_NOTICE,userActKey..':userActNum:'..userActNum..';perActLim:'..perActLim)
      local curUserActNumInt = tonumber(userActNum)
      userActNumInt =  curUserActNumInt + buyNum
   end

   -- 用户秒杀该活动的次数 > 限制的次数,返回:-2
   if userActNumInt > perActLim then
       return '-2'
   end
 end


-------------------------------------用户参与秒杀商品(SKU)限制代码块-------------------------------------
local goodsUserKey = userId..'_'..skuId
redis.log(redis.LOG_NOTICE,'perSkuLim:'..perSkuLim)
--当前用户已购买此sku多少件
-- perSkuLim > 0 代表开启了限制用户购买SKU的数量
if perSkuLim > 0 then
   -- 获得SKU的库存
   local goodsUserNum = redis.call('hget',user_sku_hash,goodsUserKey)
   -- SKU临时变量
   local goodsUserNumint = 0

   if goodsUserNum == false then
      redis.log(redis.LOG_NOTICE,'goodsUserNum is nil')
      goodsUserNumint = buyNum
   else
      redis.log(redis.LOG_NOTICE,'goodsUserNum:'..goodsUserNum..';perSkuLim:'..perSkuLim)
      -- 转换成数值
      local curSkuUserNumint = tonumber(goodsUserNum)
      -- 临时做库存的增减,那么问题来了,这个值,我为什么没有看到进行保存(hset)?
      goodsUserNumint =  curSkuUserNumint + buyNum
   end

   redis.log(redis.LOG_NOTICE,'------goodsUserNumint:'..goodsUserNumint..';perSkuLim:'..perSkuLim)
   -- 用户购买的SKU超出最大限购数量了
   if goodsUserNumint > perSkuLim then
       return '-1'
   end
end

--判断是否还有库存满足当前秒杀数量
if skuAmount >= buyNum then
     -- 聪明,把购买数量变成负数
     local decrNum = 0 - buyNum
     -- 对SKU的库存,进行扣除.
     redis.call('hincrby',sku_amount_hash,skuId,decrNum)
     -- 打日志
     redis.log(redis.LOG_NOTICE,'second success:'..skuId..'-'..buyNum)


     -- 记录用户秒杀SKU的数量
     if perSkuLim > 0 then
         redis.call('hincrby',user_sku_hash,goodsUserKey,buyNum)
     end

    -- 记录用户秒杀活动的数量
     if perActLim > 0 then
         redis.call('hincrby',user_act_hash,userActKey,buyNum)
     end

     -- 产生一个唯的订单KEY
     local orderKey = userId..'_'..skuId..'_'..buyNum..'_'..orderTime
     local orderStr = '1'
     redis.call('hset',second_log_hash,orderKey,orderStr)

   return orderKey
else
   return '0'
end
```
### (4). SecondController
```
@Controller
@RequestMapping("/second")
public class SecondController {

    @Resource
    private SecondService secondService;


    //静态页
    @GetMapping("/index")
    public String Index(ModelMap modelMap) {
        return "second/index";
    }


    // 秒杀商品预约到Redis中
    //添加活动中的sku,
    //参数:活动id,sku的id,sku的库存数量，当前sku针对单个用户的购买数量限制
    @GetMapping("/skuadd")
    @ResponseBody
    public Object skuAdd(@RequestParam(value="actid",required = true,defaultValue = "") String actId,
                      @RequestParam(value="skuid",required = true,defaultValue = "") String skuId,
                      @RequestParam(value="amount",required = true,defaultValue = "0") int amount) {
        if (actId.equals("")) {
            return new ServerResponseUtil(1,"activity id不可为空","");
        }
        if (skuId.equals("")) {
            return new ServerResponseUtil(1,"sku id不可为空","");
        }
        if (amount<=0) {
            return new ServerResponseUtil(1,"sku库存必須大于0","");
        }

        boolean isSucc = secondService.skuAdd(actId,skuId,amount);
        int status = 1;
        String msg = "";

        if (isSucc == true) {
            status = 0;
            msg = "add sku amount success";
        } else {
            status = 1;
            msg = "add sku amount failed";
        }

        ServerResponseUtil response = new ServerResponseUtil(status,msg,"");
        return response;
    }


    // 这个地址应该要隐藏  TODO
    //秒杀指定sku
    //参数: 活动id,用户id,购买数量,sku的id,用户购买当前sku的数量限制，用户购买当前活动中商品的数量限制
    //说明：用户id应从session或jwt获取，为方便测试做了传递
    @GetMapping("/skusecond")
    @ResponseBody
    public Object skuSecond(@RequestParam(value="actid",required = true,defaultValue = "") String actId,
                         @RequestParam(value="userid",required = true,defaultValue = "") String userId,
                         @RequestParam(value="buynum",required = true,defaultValue = "0") int buyNum,
                         @RequestParam(value="skuid",required = true,defaultValue = "") String skuId,
                         @RequestParam(value="perskulim",required = true,defaultValue = "0") int perSkuLim,
                         @RequestParam(value="peractlim",required = true,defaultValue = "0") int perActLim
                         ) {

        if (actId.equals("")) {
            return new ServerResponseUtil(1,"用户id不可为空","");
        }
        if (userId.equals("")) {
            return new ServerResponseUtil(1,"活动id不可为空","");
        }
        if (skuId.equals("")) {
            return new ServerResponseUtil(1,"sku id不可为空","");
        }
        if (buyNum<=0) {
            return new ServerResponseUtil(1,"购买数量必須大于0","");
        }

        String result = secondService.skuSecond(actId,userId,buyNum,skuId,perSkuLim,perActLim);

        String msg = "";
        int status = 1;

        if (result.equals("-1")) {
            msg = "已超出当前活动每件sku允许每人秒杀的数量";
            status = 1;
        }else if (result.equals("-2")) {
            msg = "已超出当前活动允许每人秒杀的数量";
            status = 1;
        }else if (result.equals("-3")) {
            msg = "sku不存在或秒杀数量未设置";
            status = 1;
        } else if (result.equals("0")) {
            msg = "库存数量不足，秒杀失败";
            status = 1;
        } else {
            msg = "秒杀成功;秒杀编号:"+result;
            status = 0;
        }

        ServerResponseUtil response = new ServerResponseUtil(status,msg,"");
        return response;
    }
}
```
### (5). SecondServiceImpl
```
package com.second.demo.service.impl;

import com.second.demo.service.SecondService;
import com.second.demo.util.RedisHashUtil;
import com.second.demo.util.RedisLuaUtil;
import com.second.demo.util.TimeUtil;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

@Service
public class SecondServiceImpl implements SecondService {

    @Resource
    private RedisHashUtil redisHashUtil;

    @Resource
    private RedisLuaUtil redisLuaUtil;

    /*
    *    添加一个秒杀活动的sku
    *    actId:活动id
    *     skuId:sku id
    *     amount:sku的库存数
    * */
    @Override
    public boolean skuAdd(String actId,String skuId,int amount) {

        String nameAmount = "sec_"+actId+"_sku_amount_hash";
        boolean isSuccAmount = redisHashUtil.setHashValue(nameAmount,skuId,amount);

        if (isSuccAmount) {
            return true;
        } else {
            return false;
        }
    }

    /*
    * 秒杀功能，
    * 调用second.lua脚本
    * actId:活动id
    * userId:用户id
    * buyNum:购买数量
    * skuId:sku的id
    * perSkuLim:每个用户购买当前sku的个数限制
    * perActLim：每个用户购买当前活动内所有sku的总数量限制
    * 返回:
    * 秒杀的结果
    *  * */
    @Override
    public String skuSecond(String actId,String userId,int buyNum,String skuId,int perSkuLim,int perActLim) {

        //时间字串，用来区分秒杀成功的订单
        int START = 100000;
        int END = 900000;
        int rand_num = ThreadLocalRandom.current().nextInt(END - START + 1) + START;
        String order_time = TimeUtil.getTimeNowStr()+"-"+rand_num;

        List<String> keyList = new ArrayList();
        keyList.add(userId);
        keyList.add(String.valueOf(buyNum));

        keyList.add(skuId);
        keyList.add(String.valueOf(perSkuLim));

        keyList.add(actId);
        keyList.add(String.valueOf(perActLim));

        keyList.add(order_time);

        String result = redisLuaUtil.runLuaScript("second.lua",keyList);
        System.out.println("------------------lua result:"+result);
        return result;
    }
}
```
### (6). RedisLuaUtil
```
package com.second.demo.util;

import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.scripting.support.ResourceScriptSource;
import org.springframework.stereotype.Service;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import javax.annotation.Resource;
import java.util.*;

@Service
public class RedisLuaUtil {
    @Resource
    private StringRedisTemplate stringRedisTemplate;

    //private static final Logger logger = LoggerFactory.getLogger("ratelimiterLogger");
    private static final Logger logger = LogManager.getLogger("bussniesslog");
    /*
    run a lua script
    luaFileName: lua file name,no path
    keyList: list for redis key
    return 0: fail
           1: success
    */
    public String runLuaScript(String luaFileName,List<String> keyList) {
        DefaultRedisScript<String> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("lua/"+luaFileName)));
        redisScript.setResultType(String.class);
        String result = "";
        String argsone = "none";
        //logger.error("开始执行lua");
        try {
			// 调用lua脚本
            result = stringRedisTemplate.execute(redisScript, keyList,argsone);
        } catch (Exception e) {
            logger.error("发生异常",e);
        }

        return result;
    }
}
```
### (7). 总结
仔细研究了[这个项目的代码](https://www.cnblogs.com/liconglong/p/14347794.html),虽然,功能不是很齐全,但是,在运用Lua+Redis来做秒杀(防超卖/限购买/限活动)的业务场景,考虑比其他人的要全面一些.  