---
title: Zimbra Exchange Web Services Crashes OS X Mail.app
layout: post
tags: [ mac, zimbra ]
---

Zimbra [added EWS support](http://wiki.zimbra.com/wiki/Exchange_Web_Services_EWS) in ZCS 8.5. Right around that time OS X 10.10 was released. Cool! Let's start syncing all our things to our brand new Mail.app, Calendar.app, and Contacts.app over HTTPS!

Strangely, [this page](https://wiki.zimbra.com/wiki/Ajcody-Apple-Mac-Issues#EWS_Configuration_And_ZCS_8.5.2B) says that:

> ZCS 8.5 targeted EWS support ONLY with Outlook for Mac's. There was no testing or expectation that the native mac apps would work with the EWS configuration type.

Really?! I hope that's not true. I haven't heard it anywhere else.

Let's try it anyway. If you created an "exchange" account in Mail.app on Yosemite, it crashed almost immediately after startup. The only way to use your existing accounts Mail.app at all was to turn off the account in the Internet Accounts setting before starting Mail.app. That [bug #94779](https://bugzilla.zimbra.com/show_bug.cgi?id=94779) was fixed in [ZCS 8.6.0](https://files.zimbra.com/website/docs/8.6/ZCS_860_NE_ReleaseNotes_UpgradeInst.pdf).

Cool, upgrade to [ZCS 8.6.0](https://files.zimbra.com/website/docs/8.6/ZCS_860_NE_ReleaseNotes_UpgradeInst.pdf) and now Mail.app runs more than a few seconds! Unfortunately, it will not complete downloading my entire account before it crashes. It seems like there is a parse error caused by the content of a particular message. The crash looks a bit like this:

```
Process:               Mail [67048]
Path:                  /Applications/Mail.app/Contents/MacOS/Mail
Identifier:            com.apple.mail
Version:               8.2 (2070.6)
Build Info:            Mail-2070006000000000~1
Code Type:             X86-64 (Native)
Parent Process:        ??? [1]
Responsible:           Mail [67048]
User ID:               1025

Date/Time:             2015-02-17 20:18:49.594 -0800
OS Version:            Mac OS X 10.10.2 (14C109)
Report Version:        11
Anonymous UUID:        97A6047A-E6A2-BD52-5661-DD0ECDCB65C1


Time Awake Since Boot: 1400000 seconds

Crashed Thread:        27  Dispatch queue: MFEWSAccountRequestResponseQueue :: NSOperation 0x6080008ae3a0 (QOS: USER_INITIATED)

Exception Type:        EXC_CRASH (SIGABRT)
Exception Codes:       0x0000000000000000, 0x0000000000000000

Application Specific Information:
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Got an invalid or absent character set for EWS MIME content, <EWSMimeContentType 0x60000223c4a0> {
}
'
abort() called
terminating with uncaught exception of type NSException

Application Specific Backtrace 1:
0   CoreFoundation                      0x00007fff98bef66c __exceptionPreprocess + 172
1   libobjc.A.dylib                     0x00007fff8b86976e objc_exception_throw + 43
2   CoreFoundation                      0x00007fff98bef44a +[NSException raise:format:arguments:] + 106
3   Foundation                          0x00007fff8bec93a9 -[NSAssertionHandler handleFailureInMethod:object:file:lineNumber:description:] + 195
4   Mail                                0x00007fff8d67053b +[MFEWSMessage dataFromMimeContent:] + 358
5   Mail                                0x00007fff8d667c8f -[MFEWSGetItemDataResponseOperation executeOperation] + 1300
6   MailCore                            0x00007fff907de2ba -[MCMonitoredOperation main] + 234
7   Foundation                          0x00007fff8bdde32c -[__NSOperationInternal _start:] + 653
8   Foundation                          0x00007fff8bdddf33 __NSOQSchedule_f + 184
9   libdispatch.dylib                   0x00007fff906d4c13 _dispatch_client_callout + 8
10  libdispatch.dylib                   0x00007fff906d8365 _dispatch_queue_drain + 1100
11  libdispatch.dylib                   0x00007fff906d9ecc _dispatch_queue_invoke + 202
12  libdispatch.dylib                   0x00007fff906d76b7 _dispatch_root_queue_drain + 463
13  libdispatch.dylib                   0x00007fff906e5fe4 _dispatch_worker_thread3 + 91
14  libsystem_pthread.dylib             0x00007fff94234637 _pthread_wqthread + 729
15  libsystem_pthread.dylib             0x00007fff9423240d start_wqthread + 13
```

Is this an Apple bug or a Zimbra bug? The relevant Zimbra [bug is 97198](https://bugzilla.zimbra.com/show_bug.cgi?id=97198).

So, turn up the EWS logging to _debug_ like this:

```
[zimbra@zimbra log]$ zmprov addAccountLogger email@domain.net zimbra.ews debug
```

Below are the last few lines of `ews.log` when my client crashed. They aren't all that interesting.

```
2015-02-17 20:18:46,068 INFO  [qtp509886383-240825:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();EWSOperation=syncFolderItem;Folder=29346;EwsClientReqSyncState={43CD6044-B74C-3886-821D-7388FA4F7435}1;] ews - Start syncFolderItem
2015-02-17 20:18:46,068 WARN  [qtp509886383-240825:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();EWSOperation=syncFolderItem;Folder=29346;EwsClientReqSyncState={43CD6044-B74C-3886-821D-7388FA4F7435}1;] ews - SyncKey error: {43CD6044-B74C-3886-821D-7388FA4F7435}1; resetting device
2015-02-17 20:18:46,068 INFO  [qtp509886383-240825:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();EWSOperation=syncFolderItem;Folder=29346;EwsClientReqSyncState={43CD6044-B74C-3886-821D-7388FA4F7435}1;] ews - End syncFolderItem: 1
2015-02-17 20:18:46,109 DEBUG [qtp509886383-240827:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();] ews - Received GetItem from Item :30917
2015-02-17 20:18:46,111 DEBUG [qtp509886383-240827:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();] ews - Received GetItem from Item :30918
2015-02-17 20:18:46,113 DEBUG [qtp509886383-240827:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();] ews - Received GetItem from Item :30920
2015-02-17 20:18:46,115 DEBUG [qtp509886383-240827:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();] ews - Received GetItem from Item :30921
2015-02-17 20:18:46,117 DEBUG [qtp509886383-240827:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();] ews - Received GetItem from Item :30922
2015-02-17 20:18:46,119 DEBUG [qtp509886383-240827:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();] ews - Received GetItem from Item :30923
2015-02-17 20:18:46,122 INFO  [qtp509886383-240827:https://10.1.200.23:443/ews/Exchange.asmx] [name=email@domain.net;ip=10.1.200.220;ua=MacOSX/(C)ExchangeWebServices/()Mail/();] ews - End getItem
```

Turn down the log level before you forget.

```
[zimbra@zimbra log]$ zmprov removeAccountLogger email@domain.net zimbra.ews debug
```

Let's use [zmmailbox](http://wiki.zimbra.com/wiki/Zmmailbox) to see what is in that message.

```
[zimbra@zimbra ~]$ zmmailbox -z -m email@domain.net getMessage 30923
Id: 30923
Conversation-Id: -30923
Folder: /certification/cisco/certification
Subject: CCNP Cert Kits and Podcasts, Plus Cisco M-Learning
From: Cisco Press <newsletter@ciscopress.com>
To: <email@DOMAIN.NET>
Date: Mon, 12 Apr 2010 23:31:52 -0700 (PDT)
Size: 27.73 KB

This newsletter has been formatted to be displayed in an HTML e-mail client.
If you are seeing this message your e-mail client does not support HTML.  To
see the newsletter follow this link:

http://www.ciscopress.com/newsletters/whatsnew.asp?ni=28&st=47442


```

Let's see if we can find more info by [looking at the metadata](http://wiki.zimbra.com/wiki/Account_mailbox_database_structure).

```
[zimbra@zimbra ~]$ zmprov getMailboxInfo email@domain.net
mailboxId: 3
quotaUsed: 2157095161
[zimbra@zimbra ~]$ expr 3 % 100
3
[zimbra@zimbra ~]$ mysql mboxgroup3
MariaDB [mboxgroup3]> select * from mail_item where id=30923 \G;
*************************** 1. row ***************************
  mailbox_id: 3
          id: 30923
        type: 5
   parent_id: NULL
   folder_id: 30878
prev_folders: NULL
    index_id: 30923
     imap_id: 30923
        date: 1271140312
        size: 28400
     locator: 1
 blob_digest: I5,YgU,COUgrmv4ck1ARg37Ax3wuVLqTyp1hBiKnl6I=
      unread: 0
       flags: 0
        tags: 0
   tag_names: NULL
      sender: Cisco Press
  recipients: email@DOMAIN.NET
     subject: CCNP Cert Kits and Podcasts, Plus Cisco M-Learning
        name: NULL
    metadata: d1:f153:This newsletter has been formatted to be displayed in an HTML e-mail client. If you are seeing this message your e-mail client does not support HTML. ...1:s41:"Cisco Press" <newsletter@ciscopress.com>1:t15:email@DOMAIN.NET1:vi10ee
mod_metadata: 53822
 change_date: 1404997612
 mod_content: 53822
        uuid: NULL
1 row in set (0.19 sec)
```

I don't see anything obvious...
