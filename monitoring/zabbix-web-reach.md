---
title: Zabbix monitor web reachability
description: Zabbix monitor web reachability
published: true
date: 2025-06-28T08:39:31.417Z
tags: linux
editor: markdown
dateCreated: 2025-06-11T13:52:58.454Z
---

# Zabbix monitor web reachability

> You can configure web reachability undert `Configurations\Templates`. Choose *HTTP service* OR *HTTPS service* or both. Go to `Web scenarios` tab and pres Create web scenario. Set up the following settings there:
{.is-info}


<kbd>Scenario:</kbd>
- **Name**: An unique name
- **Update interval**: set in interval 30s, 1m, 1h ...
- **Attempts**: 1 
- **Agent**: Zabbix

<kbd>Steps:</kbd>
- Add 
 - **Name**: An unique name
 - **URL**: http://fqdn
 - **Timeout**: 15s
 - **Require status codes**: 200
 
<kbd>Authentication:</kbd>
- If you use anything set it!
  
> I recommend you to edit Trigger, because if you use proxy, than your website is reachable, just your proxy don't and it will go down, set up a trigger to *HTTP response code*.
{.is-warning}

- Clone the one that exists
- Add this expression to the end after an OR statement: `last(/www.billund.lego.dk/web.test.rspcode[Availibility of www.billund.lego.dk,Site availibility])<>200`
- Don't forget to enable it
