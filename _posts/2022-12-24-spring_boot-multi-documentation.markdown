---
layout: post
title: "SDET. SpringBoot. Multi-documentation"
subtitle: "How to aggregate API documentation from multiple sources (OpenAPI, Postman) in one place using SpringBoot + SpringFox (Swagger UI)"
date:   2022-12-24 23:58:59 -0700
categories: sdet
tags: [springboot, openAPI, postman, springfox, Swagger]
author: Yurii Chukhrai
---

## Introduction
You probably already familiar with microservices, how good it is to have them (the more, the better :) ), and how bad it is when you have a monolithic application.
In this micro-article, I would like to share how we solve some of the problems associated with documentation for APIs in a monolithic or micro-services-based application.
As a result, most companies end up with a kind of Frankenstein — most of the product is a monolithic solution and a bunch (you don’t know how many) microservices.
Still, you need to start with a systematic approach to keeping records and keeping them up to date.
If you are advanced and your developers use the **Open API** protocol — stop reading here immediately :).
Our goal:
Aggregate API documentation in one place/service. The sources of these information can be projects that implemented **Open API** or **Postman** collections. 

Not all our services use the **Open API**, but we can convert them to an **Open API JSON** file.

Ok, we will use **SpringBoot** + **SpringFox** (include **Swagger UI**) to aggregate and distribute information.
The API information will be in the following sources:
- consequence — view JSON file in **Open API** format (**Postman** collections can be converted to **Open API** JSON file — links here)
- remotely — our microservices use the **Open API** to get documentation as a **JSON** file.

![fig_01.jpg](/resources/sdet/2022-12-24-spring_boot-multi-documentation/images/fig_01.jpg){: .mx-auto.d-block :}
**Figure 1. Spring Boot configuration Bean for OpenAPI/Swagger documentation**
{: style="text-align:center;"}

You can use these two approaches to convert Postman collections to **OpenAPI JSON**, format:
- [**postman-to-openapi**](https://joolfe.github.io/postman-to-openapi)
- [**Postman2OpenAPI**](https://github.com/kevinswiber/postman2openapi)

![fig_02.jpg](/resources/sdet/2022-12-24-spring_boot-multi-documentation/images/fig_02.jpg){: .mx-auto.d-block :}
**Figure 2. OpenAPI documentation JSON (it’s how Mozilla parse JSON by default)**
{: style="text-align:center;"}

![fig_03.jpg](/resources/sdet/2022-12-24-spring_boot-multi-documentation/images/fig_03.jpg){: .mx-auto.d-block :}
**Swagger UI. Uploaded project API information from [src/main/resources/public/openapi.json]**
{: style="text-align:center;"}

The GitHub repository with code is [here](https://github.com/YuriiChukhrai/multi-openapi-documentation).
