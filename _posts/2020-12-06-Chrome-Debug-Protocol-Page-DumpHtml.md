---
layout: post
title: 'Chrome Debug Protocol--DumpHtmlFromPageExample(七)'
date: 2020-12-06
author: 李新
tags: CDP
---

### (1). DumpHtmlFromPageExample
```
package com.github.kklisura.cdt.examples;

import java.io.File;
import java.io.IOException;
import java.util.concurrent.CountDownLatch;

import org.apache.commons.io.FileUtils;

import com.github.kklisura.cdt.launch.ChromeLauncher;
import com.github.kklisura.cdt.protocol.commands.Network;
import com.github.kklisura.cdt.protocol.commands.Page;
import com.github.kklisura.cdt.protocol.commands.Runtime;
import com.github.kklisura.cdt.protocol.types.runtime.Evaluate;
import com.github.kklisura.cdt.services.ChromeDevToolsService;
import com.github.kklisura.cdt.services.ChromeService;
import com.github.kklisura.cdt.services.types.ChromeTab;
import com.github.kklisura.cdt.utils.FilesUtils;

/**
 * The following example dumps the index html from github.com.
 *
 * @author Kenan Klisura
 */
public class DumpHtmlFromPageExample{
  public static void main(String[] args)   throws Exception {
    // Create chrome launcher.
    final ChromeLauncher launcher = new ChromeLauncher();

    // Launch chrome either as headless (true) or regular (false).
    final ChromeService chromeService = launcher.launch(false);

    // Create empty tab ie about:blank.
    final ChromeTab tab = chromeService.createTab();

    // Get DevTools service to this tab
    final ChromeDevToolsService devToolsService = chromeService.createDevToolsService(tab);

    // Get individual commands
    final Page page = devToolsService.getPage();
    final Network network = devToolsService.getNetwork();
    final Runtime runtime = devToolsService.getRuntime();
    
    network.onRequestWillBeSent(
            event ->
                System.out.printf(
                    "request: %s %s%s",
                    event.getRequest().getMethod(),
                    event.getRequest().getUrl(),
                    System.lineSeparator()));

    // Wait for on load event
    page.onLoadEventFired(
        event -> {
        	System.out.println("==============onLoadEventFired==================");
        	try {
				Thread.sleep(2000L);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
          // Evaluate javascript
          Evaluate evaluation = runtime.evaluate("document.documentElement.outerHTML");
          System.out.println(evaluation.getResult().getValue());
          byte[] bodys = evaluation.getResult().getValue().toString().getBytes();
          try {
			FileUtils.writeByteArrayToFile(new File("/Users/lixin/GitRepository/chrome-devtools-java-client/cdt-examples/hello.html"), bodys);
		} catch (IOException e) {
			e.printStackTrace();
		}
//          // Close devtools.
//          devToolsService.close();
        });

    // Enable page events.
    network.enable();
    page.enable();

    // Navigate to github.com.
//    page.navigate("https://open.tongtool.com/apiDoc.html#/?docId=43a41f3680e04756a122d8671f2fc0ca");
    page.navigate("https://github.com");
    
    
    CountDownLatch latch = new CountDownLatch(1);
    latch.await();

    // Wait until devtools is closed.
//    devToolsService.waitUntilClosed();

    // Close tab.
//    chromeService.closeTab(tab);
  }
}
```
### (2). 结果
> Page.onLoadEventFired只调用了一次.
> 在解析到的HTML里,是包含有JS渲染后的内容.