---
layout: post
title: "Full Text Search in the Sitecore User Manager"
date: 2016-02-25 12:28:53 +0000
comments: true
categories: [sitecore,lucene]
---

##How to be able to search by name, email and comments in the Sitecore User Manager

After googling on how to enhance the Sitecore User Manager, I bumped into [Teach User Manager how to search by email][1] and [Search by Email in the Sitecore User Manager][2]. However, that's not good enough for me because you need to do exact match on the email address.

So what if we use Sitecore Content Search for this? After getting an example on [Indexing an External Database in Sitecore 7][3], I decided to integrate both approches: Override the default UserProvider and DomainProvider to use Sitecore Content Search.

## Create a custom crawler and use it on your custom user provider

### Create a IndexedUser class

{% gist 150a607dcf96acb54b60 IndexedUser.cs %}

### And use it on the IndexableUser

{% gist 150a607dcf96acb54b60 IndexableUser.cs %}

### Create an user crawler

{% gist 150a607dcf96acb54b60 UserCrawler.cs %}

### And override the default UserProvider

{% gist 150a607dcf96acb54b60 CustomUserProvider.cs %}

### Finally, add your include config

{% gist 150a607dcf96acb54b60 z.Overrides.config %}

## Override the domain provider

### Create a CustomDomain

{% gist 150a607dcf96acb54b60 CustomDomain.cs %}

### And replace the Domains.config

{% gist 150a607dcf96acb54b60 Domains.config %}

This has been tested on Sitecore 8.1 Update 1.

  [1]: http://sitecoregadgets.blogspot.co.uk/2010/07/teach-user-manager-how-to-search-by.html
  [2]: https://sitecorecontextitem.wordpress.com/2014/11/10/search-by-email-in-the-sitecore-user-manager/
  [3]: http://www.awareweb.com/awareblog/9-30-14-indexingsitecore7
