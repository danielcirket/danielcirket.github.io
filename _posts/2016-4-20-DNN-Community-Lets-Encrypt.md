---
layout: post
title: Using Lets Encrypt with DNN Community Edition 
---

So this is a quick guide to getting Lets Encrypt up and running with DNN Community Edition with using the "Advanced" URL rewrite option, after some unfortunate circumstances we had to migrate our customer websites to a new webserver at work, and in the process decided to give LetsEncrypt a go.

Initial tests with a static site were successful and took only a few seconds to setup using [letsencrypt-win-simple](https://github.com/Lone-Coder/letsencrypt-win-simple).

However getting the same results with DNN using it's advanced URLs were another story as the authorisation request was failing citing a 404.

So I had to spend a while looking to why this was failing, initially I thought it might just be the extensionless MIME type, but adding this to the IIS site didn't yield any succesful results.

After a few simple settings changes I decided it'd be easiest to grab the [DNN source code](https://github.com/dnnsoftware/Dnn.Platform) from GitHub and take a look at what happens with the request.

I noticed that there was a check near the start of the rewrite which determines if DNN should ignore the request or continue running through it's advanced url rewriter:

The file can be found:  

```txt
DNN Platform\Library\Entities\Urls\AdvancedUrlRewriter.cs
```

and the check:

![ignore request](https://raw.githubusercontent.com/danielcirket/danielcirket.github.io/master/images/2016-04-20-DNN-Community-Lets-Encrypt/Ignore-Request.PNG)  

So noting that this used a regex match from some settings somewhere I tracked down where this was being set.

The regex gets set in the following file:

```txt
DNN Platform\Library\Entities\Urls\FriendlyUrlSettings.cs
```

and located here:

![ignore regex](https://raw.githubusercontent.com/danielcirket/danielcirket.github.io/master/images/2016-04-20-DNN-Community-Lets-Encrypt/Ignore-Regex.PNG)  

So what this method does, is try to load some settings, or if none are found use the default value which is specified in the screenshot above.

So this looked promising, so I grabbed the key for the setting from the top of the class, and added the entry to the database with the LetsEncrypt auth challenge directory appended to the end.

And this worked, success!

Below are the steps required to get LetsEncrypt up and running with your DNN Community Install.
<br>

**Step 1:** Setting up an entry in the HostSettings table  

For this I used management studio, and manually entered the row into the database using the UI:

![host settings entry](https://raw.githubusercontent.com/danielcirket/danielcirket.github.io/master/images/2016-04-20-DNN-Community-Lets-Encrypt/AUM_IgnoreUrlRegex.PNG)  

The values are:

```
SettingName: AUM_IgnoreUrlRegex
SettingValue: (?<!linkclick\.aspx.+)(?:(?<!\?.+)(\.pdf$|\.gif$|\.png($|\?)|\.css($|\?)|\.js($|\?)|\.jpg$|\.axd($|\?)|\.swf$|\.flv$|\.ico$|\.xml($|\?)|\.txt$|/\.well-known/acme-challenge/))
SettingIsSecure: False
CreatedByUserID: -1
CreatedOnDate: GETDATE()
LastModifiedByUserID: -1
LastModifiedOnDate: GETDATE()
```  

The default value simply has the LetsEncrypt challenge directory appended so that the DNN advanced url rewriter doesn't take over the request and 404 the page.

If you would prefer a script rather than using the Management Studio UI _(Change the table name to match your actual table)_:

```sql
INSERT INTO HostSettings
(
    SettingName,
    SettingValue,
    SettingIsSecure,
    CreatedByUserID,
    CreatedOnDate,
    LastModifiedByUserID,
    LastModifiedOnDate
)
VALUES
(
    'AUM_IgnoreUrlRegex',
    '(?<!linkclick\.aspx.+)(?:(?<!\?.+)(\.pdf$|\.gif$|\.png($|\?)|\.css($|\?)|\.js($|\?)|\.jpg$|\.axd($|\?)|\.swf$|\.flv$|\.ico$|\.xml($|\?)|\.txt$|/\.well-known/acme-challenge/))',
    0,
    -1,
    GETDATE(),
    -1,
    GETDATE()
)
```
<br>
**Step 2:** Restart your website

So, to prevent DNN using the older cached settings which were updated before, you will need to restart your website through IIS.
<br>

**Step 3:** Test you can access an extensionless file in the LetsEncrypt challenge directory

You will need to now test if you can access an extensionless file in the directory that will be used for the challenge:

```txt
http://www.yourwebsite.com/.well-known/acme-challenge/
```

*LetsEncrypt will add the approriate files here when adding the certificate, including a web.config which should allow extensionless files to be served. I had problems with this locally but did not on my webserver, so for the website (not IIS overall, this didn't work for me) I had to add the following MIME type:*

```txt
Extension: "."
Type: "text/json"
``` 

You should now be able to put an extensionless file in the directory and successfully navigate to it in your browser.
<br>

**Step 4:** Run the LetsEncryptTool and install the certificate

So providing that you managed to access the extensionless file in the previous step, you should now be setup and ready to run the LetsEncrypt tool to install your websites certificate.

For this you will need to download and extract the [letsencrypt-win-simple](https://github.com/Lone-Coder/letsencrypt-win-simple) tool to a folder on the computer/web server.

![LetsEncrypt Tool Folder](https://raw.githubusercontent.com/danielcirket/danielcirket.github.io/master/images/2016-04-20-DNN-Community-Lets-Encrypt/lets-encrypt-tool-folder.PNG)  

Within the folder, you will now need to run the 'letsencrypt' executable.

![LetsEncrypt Tool Step 1](https://raw.githubusercontent.com/danielcirket/danielcirket.github.io/master/images/2016-04-20-DNN-Community-Lets-Encrypt/lets-encrypt-tool-running-step1.PNG) 

Next, you will need to select either one of the bindings (numbered), or one of the other available options. For simplicity, I am going to assume that we are selecting a specific binding.

Once you have entered which host (In my example: 1), you can press enter to proceed to the next step which will run through, get a certificate and apply it to the binding you have selected.

Providing that the request for a certificate was successful you should now see the following in IIS for your site binding:

![IIS after certificate install](https://raw.githubusercontent.com/danielcirket/danielcirket.github.io/master/images/2016-04-20-DNN-Community-Lets-Encrypt/iis-after-install.png) 

Where the hostname and certificate are now set approriately to match your environment.

The LetsEncrypt tool will also setup a scheduled task to renew the certificate automatically when due (at the time of writing every 60 days by default)

So that should now mean that your site is up and running with an SSL certificate from LetsEncrypt.
<br>
