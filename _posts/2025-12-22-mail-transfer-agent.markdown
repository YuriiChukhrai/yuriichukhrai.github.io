---
layout: post
title: "SDET. Mail transfer server with Greenmail"
subtitle: Testing email flows locally with GreenMail and Docker
date:   2025-12-22 23:59:59 -0700
categories: sdet
tags: [thunderbird, docker, email, greenmail, mta]
author: Yurii Chukhrai
---

## Introduction

When working on applications that send emails, testing is often more complicated than it should be.
Real **SMTP** servers introduce delays, security concerns, and dependencies that slow development and testing.

Email is a critical part of many systems:
* account registration/activation
* password reset
* notifications
* alerts from background jobs

Testing email against real SMTP providers (Gmail, Outlook, corporate mail servers) is problematic:
* Requires credentials and secrets
* Emails may go to real users by mistake
* Hard to automate in CI/CD
* Network or provider issues break tests

A local email server solves these problems by allowing us to:
* capture emails without sending them to the outside world
* inspect message content (subject, headers, body, attachments)
* fully automate email-related tests

[GreenMail](https://greenmail-mail-test.github.io/greenmail/#about) is an open source, intuitive and easy-to-use test suite of email servers for testing purposes.
It supports:
* SMTP, IMAP, and POP3
* user management
* REST API
* Docker and Kubernetes

It is designed specifically for testing, not for production email delivery.

## Server

### Docker

The fastest way to start is Docker.
A single container gives you a fully working mail server in seconds.

```shell
docker run -t -i \
-e GREENMAIL_OPTS='-Dgreenmail.setup.test.all -Dgreenmail.hostname=0.0.0.0 -Dgreenmail.auth.disabled -Dgreenmail.verbose' \
-e JAVA_OPTS='-Djava.net.preferIPv4Stack=true -Xmx512m' \
-p 3025:3025 -p 3110:3110 -p 3143:3143 \
-p 3465:3465 -p 3993:3993 -p 3995:3995 -p 8080:8080 \
greenmail/standalone: 2.1.8
```

### Docker Compose with Greenmail and Roundcube
To make email inspection easier, I added a webmail UI using Roundcube (for demo). Open http://localhost:8000 in your browser.

This allows:
* browsing inboxes in a browser
* reading email bodies
* checking attachments

**Example** `docker-compose.yml`
```yaml
#$> docker compose -f docker-compose.yaml up -d
#$> docker compose -f docker-compose.yaml down

services:
  greenmail:
    image: greenmail/standalone:latest
    environment:
      - JAVA_OPTS=-Djava.net.preferIPV4Stack=true -Xmx512m
      - GREENMAIL_OPTS=-Dgreenmail.setup.test.all -Dgreenmail.hostname=0.0.0.0 -Dgreenmail.auth.disabled -Dgreenmail.verbose -Dgreenmail.startup.timeout=5000 -Dgreenmail.users=user01:Test1234@qa.bla.com,user02:Test1234@qa.bla.com

    ports:
      - 3025:3025   # SMTP
      - 3110:3110   # POP3
      - 3143:3143   # IMAP
      - 3465:3465   # SMTPS
      - 3993:3993   # IMAPS
      - 3995:3995   # POP3S
      - 8089:8080   # API
    networks:
      - mail_network

  roundcube:
    image: roundcube/roundcubemail:latest
    depends_on:
      - greenmail
    ports:
      - 8000:80
    environment:
      - ROUNDCUBEMAIL_DEFAULT_HOST=greenmail    # IMAP server - tls:// prefix for STARTTLS, ssl:// for SSL/TLS
      - ROUNDCUBEMAIL_DEFAULT_PORT=3143         # IMAP port
      - ROUNDCUBEMAIL_SMTP_SERVER=greenmail     # SMTP server - tls:// prefix for STARTTLS, ssl:// for SSL/TLS
      - ROUNDCUBEMAIL_SMTP_PORT=3025            # SMTP port
      - ROUNDCUBEMAIL_DB_TYPE=sqlite
      - ROUNDCUBEMAIL_SKIN=elastic
    networks:
      - mail_network

networks:
  mail_network:
```

![roundcube#1](/resources/sdet/2025-12-22-mail-transfer-agent/images/roundcubemail_01.jpg){: .mx-auto.d-block :}

#### Thunderbird Email Client
To verify that everything behaves like a real mail server, I tested with [Thunderbird](https://www.thunderbird.net/en-US/thunderbird).

![thunderbird#1](/resources/sdet/2025-12-22-mail-transfer-agent/images/thunderbird_01.jpg){: .mx-auto.d-block :}
![thunderbird#2](/resources/sdet/2025-12-22-mail-transfer-agent/images/thunderbird_02.jpg){: .mx-auto.d-block :}

### Simple standalone server
The fat JAR can be downloaded from [Maven central repository](https://mvnrepository.com/artifact/com.icegreen/greenmail-standalone).

**server.sh**
```shell
#!/bin/bash

clear;

java --version;

java -Djava.net.preferIPV4Stack=true -Xmx1024m \
-Dgreenmail.setup.test.all -Dgreenmail.hostname=localhost -Dgreenmail.auth.disabled -Dgreenmail.verbose \
-Dgreenmail.startup.timeout=5000 \
-Dgreenmail.users=user01:Test1234@qa.bla.com,user02:Test1234@qa.bla.com \
-jar greenmail-standalone-2.1.8.jar
```

## API
GreenMail also exposes a REST API, which is extremely useful for automation.

With the API, you can:
* create users
* list existing users
* check server readiness
* inspect received messages programmatically

**Example**. Check if the Server is ready
```shell
curl -X GET "http://localhost:8089/api/service/readiness" \
-H 'accept: application/json'
```

**Example**. Create the user
```shell
curl -X POST "http://localhost:8089/api/user" \
-H 'accept: application/json'\
-H 'content-type: application/json' \
-d '{"email": "user03@qa.bla.com", "login": "user03", "password": "Test1234"}'
```
![api#1](/resources/sdet/2025-12-22-mail-transfer-agent/images/greenmail_api-users.jpg){: .mx-auto.d-block :}

## Summary
With Docker, a web UI, real clients, and APIs, GreenMail becomes a powerful tool for any modern development workflow.

## References

* [GitHub](https://github.com/greenmail-mail-test/greenmail)
* [Documentation](https://greenmail-mail-test.github.io/greenmail/#about)
* [Integration clients](https://github.com/greenmail-mail-test/greenmail-client-integrations)
* [k8s](https://github.com/greenmail-mail-test/greenmail-example-k8s)
* [DockerHub](https://hub.docker.com/r/greenmail/standalone/)
* [YouTube. Demo](https://www.youtube.com/watch?v=p6gNiJryjvo)
* [GitHub examples](https://github.com/JensKnipper/greenmail-example/blob/main/docker-compose.yml)
* [Email client. Thunderbird](https://www.thunderbird.net/en-US/thunderbird/all/)
* [Maven. Fat JAR](https://mvnrepository.com/artifact/com.icegreen/greenmail-standalone)