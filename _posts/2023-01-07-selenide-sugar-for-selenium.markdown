---
layout: post
title: "SDET. Selenide — sugar for Selenium"
subtitle: ""
date:   2023-01-07 14:00:59 -0700
categories: sdet
tags: []
author: Yurii Chukhrai
---

![fig_00.jpg](/resources/sdet/2023-01-07-selenide-sugar-for-selenium/images/fig_00.jpg){: .mx-auto.d-block :}
**Selenide**
{: style="text-align:center;"}

***

Do not fear. This is not one of many articles about the **Selenium** framework. 
I just would like to share how you can make your life easier with **Selenide** (this is only for those who work with **Selenium — UI** tests).

And so, why Selenide:

1. Build-in web driver manager — makes life very easy to update/download binary drivers for **WebDriver** (_Firefox_, _Chrome_, or another browser binary driver)
1. If you are testing **AJAX** UI elements, you will have to wrap your **Selenium** methods in various wrappers that can correctly handle **Implicit** / **Explicitly** time waits or download/upload resources. 
1. Build-in proxy. It all depends on your needs. Capture network traffic. Mock back-end calls made by the website.
1. Access the required website under Basic Authorization. Generate and save **HAR** file (_HTTP Archive_) to Allure report. 
1. If it’s interesting for you, you can check the small demo in my [GitHub repository](https://github.com/YuriiChukhrai/selenium-selenide-demo).

***

**A**bout source code in [repository](https://github.com/YuriiChukhrai/selenium-selenide-demo/).

The repository contains [Selenium](http://seleniumhq.org/) and [Selenide](https://selenide.org/) UI demo tests for the Google and Indeed pages. 
Also provided demo how to upload and download files using **Selenium** and **Selenide**. 

In **Selenide** module also provided demo with build-in proxy server and how to generate **HAR** ([_HTTP Archive_](https://en.wikipedia.org/wiki/HAR_(file_format))) file and attach it to the **Allure** report.

***


## Intro

When I am doing an interview of candidates, the previous projects (in GitHub or maybe on their machine) are much more critical for me than some fundamental questions about 
**Git**, **Java**, **Selenium**, **Maven**, or **Jenkins**. We are all people. In an interview, we all can experience anxiety or fear, 
so I do not believe you can evaluate candidates’ many years’ background in a couple of hours (even if it is in many panel interviews ~ 8 hours). 
I prefer to discuss the candidate demos (explore the code and see how clean it is and what library and why they were used) or discuss their articles in blogs or LinkedIn.

## Maven Modules

1. **selenium** — this module contains the demo on pure **Selenium API** with automatically downloading necessary **WebDriver** and **POM** (page object model — the architectural approach to organize the classes).
1. **selenide** — this module contains the demo of small portion the **Selenide API**.
1. **shared** — this module contains the shared (reusable) code (processing properties, common interfaces etc.) for both modules (selenium and selenide).

**Selenide** is a framework for test automation powered by **Selenium WebDriver** that brings the following advantages:

1. Concise fluent API for tests
1. Ajax support for stable tests
1. Powerful selectors
1. Simple configuration
1. You don’t need to think how to shut down browser, handle timeouts and StaleElement Exceptions or search for relevant log lines, debugging your tests. 
2. Just focus on your business logic and let **Selenide** do the rest!


## Keynote

1. Build-in webdrivermanager
   * checks the version of the browser installed in your machine (e.g. Chrome, Firefox).
   * checks the version of the driver (e.g. chromedriver, geckodriver). If unknown, it uses the latest version of the driver.
   * downloads the **WebDriver** binary if it is not present on the WebDriverManager cache (~/.m2/repository/webdriver by default).
   * it exports the proper **WebDriver** Java environment variables required by **Selenium**.

2. Build-in proxy. Browsermob/[BrowserUp](https://selenide.org/2019/12/18/advent-calendar-network-logs-with-proxy/) proxy. Work with traffic
   * manipulate **HTTP** requests and responses
   * capture **HTTP** content
   * export performance data as a **HAR** ([HTTP Archive](https://en.wikipedia.org/wiki/HAR_(file_format))) file
   * works well as a standalone proxy serve especially useful when embedded in Selenium tests

3. New API for download files(w/o proxy, with proxy, with interceptor)

4. Build in Explicit wait conditions

### Selenide simple test

```java
// DataProvider here :) 

@Test(priority = 1, enabled = true, groups = TestGroups.DEFAULT, dataProvider = "dp")
 public void titleTest(final String site, final String title) {

  open("https://google.com");
  $(byName("q")).shouldBe(Condition.visible).sendKeys(site + Keys.ENTER);
  $$(byXpath(String.format("(//div[contains(@data-async-context,'%1$s')]//a[contains(@href,'%1$s')])[1]", site)))
    .first()
    .shouldBe(Condition.visible)
    .click();

  $("title").shouldHave(ownText(title));
 }
```

### Selenide & POM

```java
package com.yc.qa.google.search;

import static com.codeborne.selenide.Selectors.byXpath;
import static com.codeborne.selenide.Selenide.*;
import static com.codeborne.selenide.Selenide.$;
import static com.yc.qa.util.BaseUtils.makeScreenShot;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.WebDriverRunner;
import com.esotericsoftware.reflectasm.ConstructorAccess;
import io.qameta.allure.Step;
import lombok.extern.log4j.Log4j;
import org.openqa.selenium.Keys;

/**
 *
 * @author limit (Yurii Chukhrai)
 */
@Log4j
public final class SearchPageImpl implements SearchPage {

 private final String inputFieldXpath = "//input[@name='q']";
 private final String linksXpathTemplate = "(//div[contains(@data-async-context,'%1$s')]//a[contains(@href,'%1$s')])[1]";

 private String searchContent;

 @Step("Search content [{0}]")
 @Override
 public SearchPage searchContent(String searchContent) {
  this.searchContent = searchContent;

  $(byXpath(inputFieldXpath)).shouldBe(Condition.visible).sendKeys(this.searchContent + Keys.ENTER);

  return this;
 }

 @Step("Go to first link. And return page [{0}]")
 @Override
 public <T> T goToFirstLink(Class<T> pageObject) {

  $$(byXpath(String.format(linksXpathTemplate, this.searchContent))).first().shouldBe(Condition.visible).click();

  makeScreenShot("Partial. Landing Page", WebDriverRunner.getWebDriver());

  return ConstructorAccess.get(pageObject).newInstance();
 }
}

....

 @Test(priority = 2, enabled = true, groups = TestGroups.SEARCH, dataProvider = "dp")
 public void titleTest01(final String site) {

  open("http://google.com");

  Allure.feature( BaseConfig.getProperty(Constants.DRIVER_TYPE_PROP) );

  page(SearchPageImpl.class)
      .searchContent(site)
      .goToFirstLink(LandingPageImpl.class);
 }
```

### Selenide. Upload/Download files

```java
package com.yc.qa.test.selenide;

import com.browserup.bup.BrowserUpProxy;
import com.browserup.bup.proxy.CaptureType;
import com.codeborne.selenide.Condition;
import com.codeborne.selenide.Configuration;
import com.codeborne.selenide.FileDownloadMode;
import com.codeborne.selenide.WebDriverRunner;
import com.yc.qa.util.BaseUtils;
import com.yc.qa.util.TestGroups;
import io.qameta.allure.*;
import org.openqa.selenium.By;
import org.testng.Assert;
import org.testng.annotations.AfterClass;
import org.testng.annotations.BeforeClass;
import org.testng.annotations.Test;

import java.io.File;
import java.io.IOException;
import java.nio.file.Paths;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.open;
import static com.codeborne.selenide.files.FileFilters.withName;
import static com.yc.qa.util.BaseUtils.makeScreenShot;

public class UploadDownloadFileTest extends BaseTest {

    private BrowserUpProxy bmp;

    @Override
    @BeforeClass(alwaysRun = true)
    public void beforeClass() {

        Configuration.downloadsFolder = "./target/downloads";
        Configuration.fileDownload = FileDownloadMode.HTTPGET;
        Configuration.proxyEnabled = true;

        super.beforeClass();
    }

    /*
     * In the current moment the plugin [allure-harviewer] support attachment inside the test method
     *
     * */
    @AfterClass(alwaysRun = true)
    public void afterClass() throws IOException {

        if(bmp != null){
            final File harFile = new File("harFile.har");
            bmp.getHar().writeTo(harFile);

            BaseUtils.addHar(harFile.getName(), harFile);
        }
    }

    @Features({ @Feature("UPLOAD"), @Feature("DOWNLOAD") })
    @Issues({ @Issue("GA-011"), @Issue("GTA-012") })
    @Stories({ @Story("Stories: CIR-098"), @Story("Stories: CIR-099") })
    @Epics({ @Epic("Epic05"), @Epic("Epic06") })
    @TmsLinks({ @TmsLink("1234"), @TmsLink("4321") })
    @Links({ @Link(url="https://github.com/YuriiChukhrai/selenium-selenide-demo", name="GitHub"), @Link(url="https://www.linkedin.com/in/yurii-c-b55aa6174/", name="LinkedIn") })
    @Lead("Yurii Chukhrai")
    @Flaky
    @Severity(SeverityLevel.NORMAL)
    @Description("Upload and Download files.")
    @Owner("Yurii Chukhrai")
    @Test(priority = 0, enabled = true, groups = TestGroups.DEFAULT)
    public void uploadDownloadFileTest() throws IOException {

        final String fileName01 = "selenide-logo.png";
        final String fileName02 = "selenium-logo.png";

        final String uploadPath = "../doc/%s";
        final String uploadedFileXpath = "//a[text()='%s']";
        final String downloadPath = "//a[@title='%1$s']/img[contains(@src,'%1$s')]/parent::a";

        open("https://blueimp.github.io/jQuery-File-Upload/");

        // After the proxy started - get it
        bmp = WebDriverRunner.getSelenideProxy().getProxy();

        // remember body of requests (body is not stored by default because it can be large)
        bmp.setHarCaptureTypes(CaptureType.getAllContentCaptureTypes());

        // remember both requests and responses
        bmp.enableHarCaptureTypes(CaptureType.REQUEST_CONTENT, CaptureType.RESPONSE_CONTENT);

        // start recording!
        bmp.newHar("test.com");


        // Upload
        $(By.xpath("//input[@type='file']")).uploadFile(Paths.get(String.format(uploadPath, fileName01)).toFile(), Paths.get(String.format(uploadPath, fileName02)).toFile());
        $(By.xpath("//button//span[contains(text(),'Start upload')]")).shouldBe(Condition.visible).click();


        makeScreenShot("Partial. Landing Page", WebDriverRunner.getWebDriver());

        // Assertions
        $(By.xpath(String.format(uploadedFileXpath, fileName01))).shouldBe(Condition.visible);
        $(By.xpath(String.format(uploadedFileXpath, fileName02))).shouldBe(Condition.visible);

        final File file01 = $(By.xpath(String.format(downloadPath, fileName01))).download(withName(fileName01));
        final File file02 = $(By.xpath(String.format(downloadPath, fileName02))).download(withName(fileName02));

        // Attach files to the report
        BaseUtils.attachImageFile(fileName01, file01);
        BaseUtils.attachImageFile(fileName02, file02);

        // Download
        Assert.assertTrue(file01.exists());
        Assert.assertTrue(file02.exists());

        /*
        * If Proxy working:
        *   a) pull the HAR file from proxy
        *   b) save HAR file
        *   c) stop Proxy
        *   d) attach HAR file to the Allure report
        * */
        if(bmp != null){
            final File harFile = new File("./target/harFile.har");
            bmp.getHar().writeTo(harFile);
            bmp.stop();
            BaseUtils.addHar(harFile.getName(), harFile);
        }
    } //Test
}
```

## Test Suites

The following are valid **TestNG** test suites for use in the `-Dtest.suite={test suite name}` parameter:
* easy
* advance
* all
* fileUploadDonload

## CLO (Command Line Options)

The following are valid test parameters:
* `-Dtest.suite` - which test scenario need to run - TestNG XML file name **all**, **easy**, **advance**.
* `-Ddriver.type` - which browser need to use, for example `CHROME`, `IE`, `FF`, `SAFARI`, `HTMLUNIT`. Optional, default `CHROME` (case-insensitive).
* `-Ddriver.version` - which the driver version need to use.
* `-Dgroups` - which test group need to run (**TestNG** groups) [SEARCH, DEFAULT or others].
* `-Preporting` - The **Maven** profile what enable generation of report (by default reporting disabled).

## Reports

In project exist 4 kind of reports (location: `{project.base.dir}\{module}\target\site\index.html`):

1) [TestNG](http://testng.org/doc/documentation-main.html) produce `index.html` report, and it resides in the same test-output folder. 
This reports gives the link to all the different component of the **TestNG** reports like Groups & Reporter Output.

2) [Surefire report](http://maven.apache.org/surefire/maven-surefire-plugin/). The Surefire Plugin is used during the test phase of the build life-cycle to execute the unit tests of an application.

![fig_01.jpg](/resources/sdet/2023-01-07-selenide-sugar-for-selenium/images/fig_01.jpg){: .mx-auto.d-block :}
**Surefire report**
{: style="text-align:center;"}

![fig_02.jpg](/resources/sdet/2023-01-07-selenide-sugar-for-selenium/images/fig_02.jpg){: .mx-auto.d-block :}
**Surefire report. Selenide module**
{: style="text-align:center;"}

3) [Allure 2 report](https://docs.qameta.io/allure/). An open-source framework designed to create test execution reports clear to everyone in the team.
> ⚠️ **NOTE:** To run the report (**HTML** + **JS**) in _Firefox_ You need leverage the restriction by going to `about:config` url and then **uncheck** `privacy.file_unique_origin` **boolean** value.

![fig_03.jpg](/resources/sdet/2023-01-07-selenide-sugar-for-selenium/images/fig_03.jpg){: .mx-auto.d-block :}
**Allure report. Selenium module**
{: style="text-align:center;"}

4) [JaCoCo](https://www.jacoco.org/) report. **JaCoCo** is a free code coverage library for **Java**, 
which has been created by the **EclEmma** team based on the lessons learned from using and integration existing libraries for many years.

![fig_04.jpg](/resources/sdet/2023-01-07-selenide-sugar-for-selenium/images/fig_04.jpg){: .mx-auto.d-block :}
**JaCoCo report. Selenium module**
{: style="text-align:center;"}

***

## Allure. HAR Viewer

The build-in proxy ([BrowserUp (Selenide)](https://selenide.org/2019/12/18/advent-calendar-network-logs-with-proxy/)) can generate **HAR** file. 
You can attach it to the **Allure** report in two different ways:

1. Like regular `text/plain` file (in demo its file `{project.base.dir}/selenide/target/harFile.har`). And after, you should use any **HAR Viewer** tools to open and analyze it (for Example: [Google HAR Analyzer](https://toolbox.googleapps.com/apps/har_analyzer/) OR [HAR Viewer](http://www.softwareishard.com/blog/har-viewer/)).
2. Or you can use the [HAR Viewer Allure plugin](https://github.com/kolsys/allure-harviewer)

In **Selenide** demo was implemented in both ways. But to enable the second one, we should proceed with the next steps:

1. Install [HAR Viewer Allure plugin](https://github.com/kolsys/allure-harviewer)
    * Go to the directory `/selenium-selenide-demo/selenide/.allure/allure-{version}/plugins/`
    * Clone the plugin repository: `git clone https://github.com/kolsys/allure-harviewer.git`
    * Enable this plugin in the Allure configuration. Open file in any text editor: `/selenium-selenide-demo/selenide/.allure/allure-{version}/config/allure.yml`

Add new plugin **allure-harviewer** to the configuration:
```yaml
plugins:
 - junit-xml-plugin
 - xunit-xml-plugin
 - trx-plugin
 - behaviors-plugin
 - packages-plugin
 - screen-diff-plugin
 - xctest-plugin
 - jira-plugin
 - xray-plugin
 - allure-harviewer
```

2. The java code for attaching and using **HAR** Viewer **Allure** plugin should (file name should be harFile.har). **Allure** DOC:
```java
@Attachment(fileExtension = ".har", value = "{0}", type = "")
public static byte[] addHar(final String filename, final File file) {...}
```

3. Attach the **HAR** file inside the Test method (current implementation in the _HAR Viewer Allure plugin_)

![fig_05.jpg](../resources/sdet/2023-01-07-selenide-sugar-for-selenium/images/fig_05.jpg){: .mx-auto.d-block :}
**Allure. HAR attachment**
{: style="text-align:center;"}

![fig_06.jpg](../resources/sdet/2023-01-07-selenide-sugar-for-selenium/images/fig_06.jpg){: .mx-auto.d-block :}
**HAR tab in Allure**
{: style="text-align:center;"}

> ⚠️ **NOTE:** If you would like to attach the HAR file outside the test method (AfterClass or AfterSuite, for example) you should modify the plugin source code.

1. Open file `/selenium-selenide-demo/selenide/.allure/allure-{version}/plugins/allure-harviewer/static/index.js` in any text editor.
2. Modify the code block

   **FROM:**
```javascript
template: function (data) {
      if (!data.testStage) {
        return;
      }
      var links = [];
      for (var i in data.testStage.attachments) {
        var attach = data.testStage.attachments[i];
        if (attach.name.match(/\.har$/)) {
          links.push({url: 'data/attachments/'+attach.source, name: attach.name});
        }
      }
```

    **TO:**
```javascript
template: function (data) {
      if (!data.testStage && !data.afterStages) {
        return;
      }

      var links = [];

      if(data.afterStages) {
        var afterStagesArray = data.afterStages 
          for (var i in afterStagesArray) {
            for (var j in afterStagesArray[i].attachments) {
              var attach = afterStagesArray[i].attachments[j];
              if (attach.name.match(/\.har$/)) { 
                links.push({url: 'data/attachments/' + attach.source, name: attach.name}); 
              }
            }
          }
      } 
      if(data.afterStages) {
            for (var i in data.testStage.attachments) {
              var attach = data.testStage.attachments[i];
              if (attach.name.match(/\.har$/)) {
                links.push({url: 'data/attachments/' + attach.source, name: attach.name});
              }
            }
        }
```


## References
* [Selenide — Official website](https://selenide.org/)
* [Selenide Wiki](https://github.com/selenide/selenide/wiki)
* [BrowserUp proxy](https://selenide.org/2019/12/18/advent-calendar-network-logs-with-proxy/)
* [HAR Viewer](http://www.softwareishard.com/blog/har-viewer/)
* [HAR Viewer Allure plugin](https://github.com/kolsys/allure-harviewer)