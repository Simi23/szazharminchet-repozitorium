---
title: Internet Information Services
description: Windows IIS configurations
published: true
date: 2025-03-26T12:17:20.916Z
tags: windows
editor: markdown
dateCreated: 2025-03-24T11:24:29.287Z
---

# HTTPS Redirection

To configure HTTPS scheme redirection, you will need an additional module called [rewrite_amd64_en-US.msi](https://www.iis.net/downloads/microsoft/url-rewrite).

Install the module.

In the desired *Site*, click on **URL Rewrite** and create an **inbound rule** (Blank Rule template) with the following settings:

 - **Requested URL:** *Matches the pattern*
 - **Using:** *Wildcards*
 - **Pattern:** `*`
 - **Conditions:** *Match any*
 - **Add condition:**
   - **Input:** `{HTTPS}`
   - **Type:** *Matches the pattern*
   - **Pattern:** `OFF`
 - **Uncheck** *Track capture groups across conditions*
 - **Action Type:** *Redirect*
 - **Redirect URL:** `https://{HTTP_HOST}{REQUEST_URI}`
 - **Redirect type:** *Found (302)*

**Apply the rule.** Now when you navigate to this site in the browser with the *http://* scheme, you should be redirected to the *https://* scheme.

```cfg
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <rewrite>
		<rules>
                <clear />
                <rule name="redirect" patternSyntax="Wildcard" stopProcessing="true">
                    <match url="*" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
			<add input="{HTTPS}" pattern="OFF" />
                    </conditions>
                    <action type="Redirect" url="https://{HTTP_HOST}{REQUEST_URI}" redirectType="Found" />
                </rule>
		</rules>
	</rewrite>
    </system.webServer>
</configuration>
```

# Use more html files from one directory

```cfg
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <rewrite>
		<rules>
                <clear />
                <rule name="redirect" patternSyntax="Wildcard" stopProcessing="true">
                    <match url="*" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
			<add input="{HTTPS}" pattern="OFF" />
                        <add input="{HTTP_HOST}" pattern="www.paris.local" />
                        <add input="{HTTP_HOST}" pattern="help.paris.local" />
                    </conditions>
                    <action type="Redirect" url="https://{HTTP_HOST}{REQUEST_URI}"redirectType="Found" />
                </rule>

                <rule name="Serve Internal Page" stopProcessing="true">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
                        <add input="{HTTP_HOST}" pattern="^www\.paris\.local$" />
                    </conditions>
                    <action type="Rewrite" url="internal.html" />
                </rule>
                
                <rule name="Serve Help Page" stopProcessing="true">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
                        <add input="{HTTP_HOST}" pattern="^help\.paris\.local$" />
                    </conditions>
                    <action type="Rewrite" url="help.html" />
                </rule>
                
                <rule name="Serve External Page" stopProcessing="true">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
                        <add input="{HTTP_HOST}" pattern="^external\.paris\.local$" />
                    </conditions>
                    <action type="Rewrite" url="external.html" />
                </rule>
		</rules>
	</rewrite>
    </system.webServer>
</configuration>

```

# Reverse proxy configuration

To configure IIS as a reverse proxy, you need to install the **URL Rewrite** module mentioned previously, and also the [Application Request Routing](https://www.iis.net/downloads/microsoft/application-request-routing) module, **in this order**.

After the module is installed, navigate to your server in **IIS Manager**. Within the **IIS** section, open **Application Request Routing Cache**, then click **Server proxy settings** in the right-side navigation menu. **Check** *Enable proxy* and set **HTTP Version** to *Pass through*.

Then create a **new Server Farm** and add your web server's IP address as a server.

Now, when you type a URL in the browser that goes to the reverse proxy, it will forward the request to the original web server.
