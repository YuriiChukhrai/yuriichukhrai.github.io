---
layout: post
title: "SDET. Test Automation for Mainframe (AS-400)"
subtitle: Adapting a Java 5250 terminal emulator for AS/400 test automation
date:   2022-12-24 23:59:59 -0700
categories: sdet
tags: [tn5250, pub400, tn5250j, mainframe, as-400, test automation, terminal-emulator, allure, java, maven, testng]
author: Yurii Chukhrai
---

**T**oday, automation must be an integral part of any software product, even at the initial development stage. 
It helps you test regression and control product quality before going into production, especially when developers are not given enough time for writing unit or integration tests.

For modern projects, be it Web UI or API, you can always find a couple of specialized libraries in almost any programming language (for Java, this is usually **Selenium/Selenide** for UI and **Rest-Assured** for API) that will help you automate your tests.

But what if your ecosystem includes legacy systems/applications for which modern libraries are not suitable, or the cost of their custom automation is more than you are willing to spend?
What if your **E2E** script includes steps in UI/API (web-based applications) and interaction with a legacy application, for example, running on **OS/400**?

This article is about how I adapted open-source software (A **5250 terminal** emulator for the **IBMi (AS/400)** written in Java) for test automation applications working on IBMi **OS/400**.

 ***

## Introduction

### Background
At a Silicon Valley-based company, I faced the problem of automating test cases for an application running on **OS/400 — PKMS** (Pick Ticket Warehouse Management Systems). If you Google it and do a keyword search for (**AS/400** test automation), you will be somewhat disappointed.

Of course, there will be many results on the page. Unfortunately, they all lead to companies that say that they have experience in automating this kind of application. You will not find any examples, articles, or video demonstrations of their methods; a demo is usually provided only to potential customers (which is already talking about the future price). 
I’m not even talking about some technical details that interested me as an engineer:
* What operating system does this tool run on?
* What actions are available to me in the SDK?
* Whether multi threading is supported?
* Integration with which development languages are implemented?
* Reporting. Is it possible to take screenshots?

Anyway, after a couple of hours, you begin to understand that someone else likely faced the same questions, will attend to the same or similar questions and may even come up with and do something.

[In this article](https://testguild.com/mainframe-testing/), you will find a more detailed description of the companies/products you can use for Mainframe testing.

Here is a shortlist:
* [Ui Path](https://www.uipath.com/kb-articles/automating-terminals-and-mainframes)
* [Rumba + Desktop](https://www.microfocus.com/en-us/products/rumba/overview)
* [Jagacy](http://www.jagacy.com/)
* [Altran](https://github.com/Altran-PT-GDC/Robot-Framework-Mainframe-3270-Library)
* [Mainframe Brute](https://github.com/singe/mainframe_brute)
* [IBM Rational Functional Tester](https://www.ibm.com/us-en/marketplace/rational-functional-tester)
* [ACCELQ](https://www.accelq.com/Enterprise-Tech-Automation)

I decided to adopt the open-source software terminal emulator for IBM written in Java for my needs. Let’s see what came of it.

```mermaid
graph TD
    subgraph Test_Layer [Test Layer]
        Tests[Java Tests]
        PO[Page Objects]
        Tests -->|Executes| PO
    end

    subgraph Adapter_Layer [Adapter Layer]
        TD[TerminalDriver]
        SB[SessionBean]
        
        PO -->|Calls Actions| TD
        TD -->|Uses| SB
        
        note[Implements TerminalElementsMethods]
        note -.-> TD
    end

    subgraph Library [TN5250J Library]
        TN[Terminal Emulator]
        SB -->|Wraps & Controls| TN
    end

    subgraph SUT [System Under Test]
        AS400[AS/400 Server]
    end

    TN <-->|5250 Protocol| AS400
```

## What is AS/400?

![fig-1.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-1.jpg){: .mx-auto.d-block :}

IBM introduced the Application System/400 in 1988. 
It was an integrated system featuring hardware (AS/400) and an operating system (OS/400), along with many core functions such as an integrated database.

Both the hardware and the software have gone through many upgrades, revisions, and name changes over the years. 
From the beginning, one of the strongest features of this platform has been its upward compatibility. 
You can run a program created for the AS/400 in 1988 on a Power Systems server today with little or no changes.

This seamless compatibility is one reason why many companies that purchased an AS/400 years ago continue to refer to it as an AS/400 even though their Power server is an order of magnitude faster and features cutting-edge technologies.

IBM continues to update the platform today. 
Every two to three years, they release new versions of the hardware and software that feature quantum leaps forward in processing power and functionality.

![fig-2.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-2.jpg){: .mx-auto.d-block :}
**Figure 2. OS/400 — Main menu**
{: style="text-align:center;"}

## PkMS
One of the programs for which I had to implement tests was PkMS. PkMS (Pick Ticket Management System) is a warehouse management system that controls inventory movement, such as receiving merchandise, inventory transactions, picking and packing, and shipping merchandise to a customer, working under the AS/400 OS (Vendor Manhattan Associates).

## TN5250J
After a bit of research, I found an open-source terminal emulator ([TN5250J](http://tn5250j.org/) for the IBM i (AS/400) written in Java — [GitHub](https://tn5250j.github.io/), and this could not but make me happy. 
After all, if I can interact through this terminal in manual mode, I can do all the same through its source code (creating a kind of API) and use this dependency in any project.
![fig-3.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-3.jpg){: .mx-auto.d-block :}
**Figure 3. TN5250J — Connection’s tab**
{: style="text-align:center;"}

The **tn5250j** is a 5250 terminal emulator for the **IBMi (AS/400)** written in Java. `It was created because there was no terminal emulator for Linux with features like continued edit fields, gui windows, cursor progression fields etc….`.

There are 3 modes:
1. A basic mode.
2. Enhanced mode that implements the features like masked edit fields and those listed above.
3. The third is a gui mode that turns the bland green-screen into a windows gui (See screen shots).

The GUI part comes in by manipulating the 5250 stream and painting the fields like gui constructs in windowing systems, gui looking popup windows in place of windows, 
painting the PF keys on the screen as buttons (hot spots) so when clicked it will send the appropriate aid key. 
This is basic gui enhancements that you can receive and interpret from the 5250 stream. Currently, that is all **tn5250j** does.
Initially, project [tn5250j](https://tn5250j.github.io/) used Ant, which seemed to be a little inconvenient for my purposes (Maven — it downloads dependencies and adds them to the classpath, 
I’m not talking about all the other goodies, which come with it as a build tool). 
And I transferred it to the Maven project and slightly corrected those bugs found by static analysis (PMD, CPD, Spotbug).

I used next tools:
* Java - preferred language of development;
* Maven - Build tool;
* TestNG - unit/integration test executor;
* Allure - reporting tool (HTML+JS);

After rebuilding a new project, I got this dependency in my local Maven repository:
```text
<dependency>
   <groupId>org.tn5250j</groupId>
   <artifactId>as400-tn5250j</artifactId>
   <version>0.9.6-SNAPSHOT</version>
 </dependency>
```

The resulting dependency is a Java archive file (* .jar) of compiled classes, which is the **TN5250J** standalone application. 
You can run it as a separate GUI application using the command: `$\> java -jar as400-tn5250j-0.9.6-SNAPSHOT.jar`

The project modified for Maven is no different from the original [project](https://tn5250j.github.io/). 
As a real **AS/400** operating system for the demo, I will use the **PUB400.COM** project — a public AS400 server.

![fig-4.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-4.jpg){: .mx-auto.d-block :}
**Figure 4. Web page of project PUB400.COM**
{: style="text-align:center;"}

## Demo

[![Demo#1](https://img.youtube.com/vi/W2Bkg0rE4v8/0.jpg)](https://youtu.be/W2Bkg0rE4v8 "Demo: Test Automation for Mainframe (AS-400)"){: .mx-auto.d-block :}
**YouTube. Test Automation for Mainframe (IBMi AS/400)**
{: style="text-align:center;"}

### Intro

Youtube

So, what is the demonstration going to be? The test will consist of the following steps:
1. Open TN5250J terminal.
2. Login to AS/400 server (PUB400.com) and verify that.
3. Go to the [User Task] menu and verify that.
4. Go back to [Main menu] and verify that.
5. Logoff.
6. Attach all test artifacts (screenshots/screen text representation) to Allure.
7. Close the TN5250J terminal.
8. Generate Allure report with the screenshots.


Yes, the test is not complicated, but our goal is not to conquer the whole world — this is just a proof of concept (POC). 
We will run the test itself from the command line using Maven:
```shell
$\> mvn clean site -Duri.as400=pub400.com -Duser.as400={username} -Dpsw.as400={password} -Dport.as400=992 -Dssl.type.as400=TSL -DisVisible
```

Of all the options, I would like to stop at `-DisVisible`. This option launches the terminal in GUI mode and allows you to see what is happening in the terminal and take screenshots. 
We just change the field values in the JFrame object (`jframe.setVisible` (false or true)). 
Without this option, the terminal will start in headless mode (but the representation of screens in the form of strings will remain, and you can attach them to a report). 
That solution can be used anywhere where Java is available. For my tests, I used Jenkins Master/Slave nodes on CentOS (with/without virtual monitor) to run test automation in CI (pipeline as code / JenkinsFile).

A few words about the technical part for working with `5250J terminal` before the demonstration. 
We will use the dependency of the 5250J emulator in our demo project so that we could use its classes, functions, and resources in our program.

![fig-5.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-5.jpg){: .mx-auto.d-block :}
**Figure 5. Hight-level diagram of: 5250J terminal + AS/400 + User actions**
{: style="text-align:center;"}

And a few words about how we will proceed with I/O (Input/Output) for `green screen` application. 
The general idea is very similar to Selenium (a library that allows you to communicate with the browser and send commands to it: 
open page, click etc.), where the **5250J** terminal is an analogue of the browser (which receives HTML + JavaScript and forms a page) — 
displays the contents of the screens and allows you to fill in various input fields and simulate function keys (F1 — F24).

![fig-6.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-6.jpg){: .mx-auto.d-block :}
**Figure 6. TN5250J dependency in demo project (Maven)**
{: style="text-align:center;"}

Selenium uses locators to work with HTML elements. Accessible by **XPath**, **ID**, **CSS**, by name, etc. In our case, for **AS/400** applications, 
you need to know their ID (digits) or text label on the screen before/after the input fields to access the input fields.

![fig-7.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-7.jpg){: .mx-auto.d-block :}
**Figure 7. PUB400.COM — Login**
{: style="text-align:center;"}


```text
+++++++++++++++++++++++ AS400 screens delimiter +++++++++++++++++++++++
Welcome to PUB400.COM * your public IBM i server
Server name . . . : PUB400
Subsystem . . . . : QINTER2
Display name. . . : DEVNAMETES
Your user name:
Password (max. 128):
===============================================================================
> Hope to see you at the Common Europe Congress in Copenhagen
(Oct 31 - Nov 3) see https://comeur.org/cec2021/









connect with SSH to port 2222 -> ssh name§pub400.com -p 2222
visit http://POWERbunker.com for professional IBM i hosting
© COPYRIGHT IBM CORP. 1980, 2018.
+++++++++++++++++++++++ AS400 screens delimiter +++++++++++++++++++++++
```
**Figure 8. A text representation of the AS/400 screens in the 5250J terminal (logged for debug).**
{: style="text-align:center;"}

The application screen in **AS/400** in the **5250J** terminal is represented as an array of characters[1] (each line of the screen is an element in this array).
I used Regular Expressions to navigate in them.

The **SessionBean** class is responsible for communicating with the **5250J** terminal to open a session, retrieve screen content, and send some commands or keys.

In the interface **TerminalElementsMethods**, I put those methods that I will need for simple manipulations in the demo (such as _sendKeys_, _fillFieldWith_, _isTextPresent_, etc).

![fig-9.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-9.jpg){: .mx-auto.d-block :}
**Figure 9. SRC. Interface TerminalElementsMethods**
{: style="text-align:center;"}

The **TerminalDriver** class contains the implementation of the **TerminalElementsMethods** interface. 
If you look at them, see how they manipulate the screen as text. The entire screen with its labels and fields is treated as an array of characters.

![fig-10.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-10.jpg){: .mx-auto.d-block :}
**Figure 10. Text label and field ID on the AS/400 screen**
{: style="text-align:center;"}

Since we will be using _TestNG_ (to run our tests) for the convenience of the code, we will define some test fixtures (open/close terminal) in `@BeforeTest` and `@AfterTest`.

![fig-11.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-11.jpg){: .mx-auto.d-block :}
**Figure 11. TestNG @BeforeTestd / @AfterTest**
{: style="text-align:center;"}

The same applies to the listener for tests — when the test fails, it will take screenshots of the terminal and grab all screens in a text view for a report in Allure.

![fig-12.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-12.jpg){: .mx-auto.d-block :}
**Figure 12. TestNG @Listener implementation**
{: style="text-align:center;"}

The test uses two screens, and their code is written as a Page Object Model. 
In general, nothing special if you’ve written (or are writing) tests for web UI applications. 
The classes in the POM are written so that they allow for chain invocation (just for convenience).

![fig-13.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-13.jpg){: .mx-auto.d-block :}
**Figure 13. POM. Main Menu page**
{: style="text-align:center;"}

After the above, the test class will turn out to be very concise since the test fixture is defined in `@BeforeTest` / `@AfterTest`.

![fig-14.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-14.jpg){: .mx-auto.d-block :}
**Figure 14. TestNG. Test method implementation (most of the annotations related to Allure report)**
{: style="text-align:center;"}



![fig-15.jpg](/resources/sdet/2022-12-24-test-automation-as-400/images/fig-15.jpg){: .mx-auto.d-block :}
**Figure 15. Allure report**{: .mx-auto.d-block :}
{: style="text-align:center;"}

***

## Conclusion
Never say never. The purpose of the demonstration is only to show that it is possible, using open-source software (**TN5250J** terminal emulator), 
to automate the actions of a real user for applications on Mainframe. A set of tools that were used quite standard for the Java test automation world (Maven, TestNG, and Allure report). 

The main problem will be only in defining unique text labels (before/after) for input fields or identifying their IDs, but this complexity is pretty like the write locators for the Selenium UI test (dynamic XPath’s), and the method I described will require knowledge of Regular Expressions. 
The tests themselves can be multithreaded anywhere there is Java and integrated into any CI.

## P.S. (Post Scriptum)
The more detailed information you can find in the [GitHub demo repository](https://github.com/YuriiChukhrai/ibm-as-400-demo).

In **README.md,** you can find not only a description of the project and command-line options for a demo project but also:
1. Links and documentation related to the project **TN5250J** and **PUB400.COM**
2. Zipped Allure report (with attached screenshots and text screen representation)
3. Recorded demo of test execution (MOV)


## References
* [tn5250j](https://tn5250j.github.io/)
* [IBM System i](https://en.wikipedia.org/wiki/IBM_System_i)
* [as400 dead](https://www.helpsystems.com/blog/as400-dead)
* [AS-400 online](http://pub400.com)
* [Mainframe testing](https://testguild.com/mainframe-testing/)
* [Warehouse Management System](https://docs.oracle.com/cd/E69185_01/cwdirect/pdf/180/cwdirect_user_reference/WH13_01.htm)
