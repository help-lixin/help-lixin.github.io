---
layout: post
title: 'Playwright代码录制并生成(二)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
在这一小篇,用Playwright代码录制功能,对代码进行录制,并且学习下,录制下的代码,主要是学习下Selectors.
### (2). 运行代码生成器
```
# 1. 查看当前目录(我是通过源码下载下来后执行的)
lixin-macbook:playwright lixin$ pwd
/Users/lixin/GitRepository/playwright-java/playwright

# 2. 运行代码生成器
lixin-macbook:playwright lixin$ mvn exec:java -e -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="codegen playwright.dev"
[INFO] Error stacktraces are turned on.
[INFO] Scanning for projects...
[INFO] Inspecting build with total of 1 modules...
[INFO] Installing Nexus Staging features:
[INFO]   ... total of 1 executions of maven-deploy-plugin replaced with nexus-staging-maven-plugin
[INFO] 
[INFO] ----------------< com.microsoft.playwright:playwright >-----------------
[INFO] Building Playwright - Main Library 1.27.0
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- exec-maven-plugin:3.1.0:java (default-cli) @ playwright ---
[: Executable doesn't exist at /Users/lixin/Library/Caches/ms-playwright/chromium-1028/chrome-mac/Chromium.app/Contents/MacOS/Chromium
╔═════════════════════════════════════════════════════════════════════════╗
║ Looks like Playwright Test or Playwright was just installed or updated. ║
║ Please run the following command to download new browsers:              ║
║                                                                         ║
║     npx playwright install                                              ║
║                                                                         ║
║ <3 Playwright Team                                                      ║
╚═════════════════════════════════════════════════════════════════════════╝
] {
  name: 'Error'
}

# 提示错误,需要安装:playwright
```
### (3). 安装playwright
```
# 2. 安装playwright
lixin-macbook:playwright lixin$ npx playwright install 
Need to install the following packages:
  playwright@1.27.0
Ok to proceed? (y) y
╔═══════════════════════════════════════════════════════════════════════════════╗
║ WARNING: It looks like you are running 'npx playwright install' without first ║
║ installing your project's dependencies.                                       ║
║                                                                               ║
║ To avoid unexpected behavior, please install your dependencies first, and     ║
║ then run Playwright's install command:                                        ║
║                                                                               ║
║     npm install                                                               ║
║     npx playwright install                                                    ║
║                                                                               ║
║ If your project does not yet depend on Playwright, first install the          ║
║ applicable npm package (most commonly @playwright/test), and                  ║
║ then run Playwright's install command to download the browsers:               ║
║                                                                               ║
║     npm install @playwright/test                                              ║
║     npx playwright install                                                    ║
║                                                                               ║
╚═══════════════════════════════════════════════════════════════════════════════╝
Downloading Chromium 107.0.5304.18 (playwright build v1028) - 125.7 Mb [====================] 100% 0.0s 
Chromium 107.0.5304.18 (playwright build v1028) downloaded to /Users/lixin/Library/Caches/ms-playwright/chromium-1028
Downloading Firefox 105.0.1 (playwright build v1357) - 73.7 Mb [====================] 100% 0.0s 
Firefox 105.0.1 (playwright build v1357) downloaded to /Users/lixin/Library/Caches/ms-playwright/firefox-1357
Downloading Webkit 16.0 (playwright build v1724) - 57.8 Mb [====================] 100% 0.0s 
Webkit 16.0 (playwright build v1724) downloaded to /Users/lixin/Library/Caches/ms-playwright/webkit-1724
npm notice 
npm notice New minor version of npm available! 8.15.0 -> 8.19.2
npm notice Changelog: https://github.com/npm/cli/releases/tag/v8.19.2
npm notice Run npm install -g npm@8.19.2 to update!
npm notice 
```
### (4). 重新运行code gen
```
# 对playwright.dev网址进行代码录制
lixin-macbook:playwright lixin$ mvn exec:java -e -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="codegen playwright.dev"
```
### (5). playwright代码生成器
!["playwright代码生成器"](/assets/playwright/imgs/playwright-code-gen.jpg)
### (6). playwright代码生成结果
```
import com.microsoft.playwright.*;
import com.microsoft.playwright.options.*;
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;
import java.util.*;

public class Example {
  public static void main(String[] args) {
    try (Playwright playwright = Playwright.create()) {
      Browser browser = playwright.chromium().launch(new BrowserType.LaunchOptions()
        .setHeadless(false));
      BrowserContext context = browser.newContext();

      Page page = context.newPage();

      page.navigate("https://playwright.dev/");

      page.getByRole("link", new Page.GetByRoleOptions().setName("Get started")).click();
      assertThat(page).hasURL("https://playwright.dev/docs/intro");

      page.getByRole("link", new Page.GetByRoleOptions().setName("Writing Tests")).click();
      assertThat(page).hasURL("https://playwright.dev/docs/writing-tests");

      page.getByRole("link", new Page.GetByRoleOptions().setName("Java")).click();
      assertThat(page).hasURL("https://playwright.dev/java/docs/writing-tests");

      page.getByRole("link", new Page.GetByRoleOptions().setName("Community")).click();
      assertThat(page).hasURL("https://playwright.dev/community/welcome");

      page.getByRole("link", new Page.GetByRoleOptions().setName("Docs")).click();
      assertThat(page).hasURL("https://playwright.dev/docs/intro");
    }
  }
}
```
### (7). 总结
通过CodeGen可以生成录制代码,我们可以通过录制代码功能学习Selectors.