---
layout: post
title: 'Project Reactor Stream OOM排查'
date: 2020-10-26
author: 李新
tags: ProjectReactorStream
---

> <font color='red'>注意:代码和分析日志都是经过处理了的</font>

### (1)生产上OOM排查思路
1. 查日志,看能否查到蛛丝马迹?  
    需要查N天的日志,因为,若是只有一两天的日志,可能会有误判,但是N天的OOM日志都提示在某一处代码,误判的概率性可性比较少.
2. 模拟请求,以证实代码是否有问题?  
    模拟并发请求,查看内存回收情况和线程回收情况,以确定代码是否存在泄露的可能性.
3. 了解业务代码含义?  

### (2).查日志
!["Project Reactor Stream Bug"](/assets/project-reactor-stream/imgs/Project-ReactorStream-Bug.png)
> 连续查看4天的OOM日志,发现:线程栈信息都是停留在业务代码:TestService.saveConfig()方法上,由此可以判断:这个类的方法应该是有问题,否则,<font color='red'>不可能巧合连续几天的OOM都是在这段代码上</font>.

### (3).了解业务
1. 用户通过UI配置延迟的定时任务信息
2. 用户提交定时任务信息
3. <font color='red'>取消以前创建的定时任务</font>
4. 从DB中获取所有启用的定时任务信息
5. <font color='red'>重新创建定时任务</font>

### (4).模拟业务代码进行请求
> 刚接手项目,代码确实有很多的漏洞:  
> 在并发的情况下,Disposable是单例的,会存在BUG.  
> 定时任务是在单机创建的,代码是否有考虑健壮性?  

```
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

import reactor.core.Disposable;
import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;

public class ProjectReactorStreamOOMTest {
	// -Xmx10m -Xms10m
	public static void main(String[] args) throws Exception {
		// 模拟Tomcat每隔10秒,发起一次调用,总共500次请求
		TestService test = new TestService();
		int maxThreads = 500;
		for(int i = 0;i < maxThreads ; i++) {
			new Thread(()->{test.saveConfig();}).start();
			TimeUnit.SECONDS.sleep(10);
		}
	}
}

class TestService {
    // 该对象是单例的,如何保证安全?
    // 定时任务这样创建?如何保证代码的健壮性?分布式部署下又该如何处理?
	private Disposable disposable;

	public void saveConfig() {
		// ...
		scheduleConfig();
		// ...
	}

	public void scheduleConfig() {
		// 取消订阅(Hold住上次创建的一批定时任务对象,进行释放)
		if (null != disposable) { 
			disposable.dispose();
		}
		// 从数据据库获取所有的定时任务信息,重新添加成定时任务
		List<Integer> list = mockFindList();
		disposable = 
		  Flux.fromIterable(list)  // 遍历集合
			.flatMap((item)->{   // 对集合每一行数据进行加工和处理
				// 根据用户配置的时间进行定时任务的调度
				return Flux.interval(
				       Duration.ofMillis(item * 1000),Schedulers.newElastic("*****test***")  
				    ).doOnNext((i)->{  // 当订阅时Publiser,在触发:onNext()之前,先触发该函数
				    	// 模拟耗时操作
				    	try {
				    		TimeUnit.SECONDS.sleep(1);
				    		System.out.println(Thread.currentThread().getName());
						} catch (Exception ignore) {
						} // end catch
				    });// end doOnNext
			}).subscribe();
	}

	public List<Integer> mockFindList() {
		List<Integer> list = new ArrayList<Integer>();
		// 模拟生产上149条数据
		for (int i = 0; i < 150; i++) {
			list.add(10);
		}
		return list;
	}
}

```

### (5).通过jvisualvm监控结果图
![](/assets/project-reactor-stream/imgs/Project-ReactorStream-Bug1.png)
![](/assets/project-reactor-stream/imgs/Project-ReactorStream-Bug2.png)
![](/assets/project-reactor-stream/imgs/Project-ReactorStream-Bug3.png)
![](/assets/project-reactor-stream/imgs/Project-ReactorStream-Bug4.png)


### (6).结论
> 从业务的需求我们知道:<font color='red'>用户修改定时任务信息时,是需要停止以前创建的定时任务</font>,而,Project ReactorStream的dispose()方法有Bug.并没有把定时任务对象(Thread)给销毁掉.

### (7)RxJava是否存在有BUG呢?
> 通过监控和测试,发现RxJava并不存在该Bug,RxJava会及时释放掉定时任务实例(Thread)  

```

package help.lixin.samples;

import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import io.reactivex.Observable;
import io.reactivex.disposables.Disposable;
import io.reactivex.schedulers.Schedulers;

public class RxJavaTest {
	public static void main(String[] args) throws Exception {
		final ServiceTemp tmp = new ServiceTemp();
		int maxThreads = 500;
		for (int i = 0; i < maxThreads; i++) {
			new Thread(() -> {
				tmp.saveConfig();
			}).start();
			TimeUnit.SECONDS.sleep(15);
		}
	}
}

class ServiceTemp {
	private Disposable disposable;
	private AtomicInteger i = new AtomicInteger(0);

	public void saveConfig() {
		System.out.println("第" + i.getAndIncrement() + "次 tomcat请求");
		if (null != disposable && !disposable.isDisposed()) {
			disposable.dispose();
		}
		List<Integer> list = values();
		disposable = Observable.fromIterable(list) //
				.flatMap((sleep) -> {
					return Observable.interval(sleep, TimeUnit.SECONDS, Schedulers.io()) //
							.doOnNext((i) -> {
						// mock 网络请求
						TimeUnit.SECONDS.sleep(1);
						// mock 网络请求
						DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss:SSS");
						System.out.println(
								Thread.currentThread().getName() + "  " + df.format(new Date()) + " task: " + sleep);
					});
				})
				.doOnError((e)->{
					System.out.println("errpr: "+e);
				})
				.subscribe();
	}

	public List<Integer> values() {
		List<Integer> list = new ArrayList<Integer>();
		for(int i=0;i<149;i++) {
			list.add(10);
		}
		return list;
	}
}

```
