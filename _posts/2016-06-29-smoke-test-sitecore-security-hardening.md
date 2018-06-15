---
layout: post
title: "Smoke Test Sitecore Security Hardening"
date: 2016-06-29 16:08:08 +0100
comments: true
categories: [sitecore,security]
---

## Add automated test on your CI to test how secure your Sitecore server

This are some test cases you can run as part of your integration tests on your CI 

### The script

{% gist 7c6a54ff9b72cbe7c060bacc39c2d931 SecurityHardeningTests.cs %}

Notice you can either set up the url on the app.config or on the environment settings which is useful if you want to dynamically change the url on for example, TeamCity.

