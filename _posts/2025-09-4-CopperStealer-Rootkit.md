---
title: "CopperStealer Rootkit deep analysis"
classes: wide
tags: [ROOTKIT]
header:
    teaser: /assets/images/myIcon/copperavater.png
description: "In this article, I'm going to do a deep analysis on rootkit that was droped from CopperStealer Malware"
categories:
  - Malware Analysis
toc: true
ribbon: DodgerBlue
---

# Malware Details

> sha256: d4d3127047979a1b9610bc18fd6a4d2f8ac0389b893bcb36506759ce2f20e7e4
>
> Malware bazaar: https://bazaar.abuse.ch/sample/d4d3127047979a1b9610bc18fd6a4d2f8ac0389b893bcb36506759ce2f20e7e4/

# Technical Summary

1. the rootkit was used in one of the stages of the CopperStealer malware attack, as it was dropped from it. 
2.  the rootkit is used to provide a protection layer to the malware files and binaries and itself.
3. it's build as legacy filter driver so it receives all kind of major functions, but it only cares about IRP_MJ_SET_INFORMATION & IRP_MJ_READ.
4. the IRP_MJ_SET_INFORMATION major routine is used to check if there's any attempt to delete the malware files and binaries by any process, and if found it will stop that action from passing to the file system driver and returns STATUS_ACCESS_DENIED. otherwise it will pass normally.
5. And the IRP_MJ_READ major routine is used to check if there's any attempt to access the malwares files (like the cookies and logs and so), and if so it will check if the malware process itself is accessing it, otherwise it will prevent the access with STATUS_ACCESS_DENIED.
6. It also register a callback routine for registry modification, to check if there's an attempt to modify or delete the registry path of the rootkit, and if so it will prevent with STATUS_ACCESS_DENIED too.



# Deep Analysis

This rootkit is build as a legacy filter kernel driver, which is the old way of writing filter kernel driver (now the common and recommended to use is mini-filter), So it has to register for every kind of IRP types, along with Fast I/O. But the whole functionality of the rootkit resides within the dispatch routines of IRP_MJ_SET_INFORMATION and IRP_MJ_READ.

> NOTE: I'm assuming that the reader has a bit knowledge in kernel driver development, so I may skip some parts that the author of the rootkit implemented because it's only needed to make sure that the driver LOADs and OPERATEs correctly, so anything I skip is not related to the malicious functionality of the rootkit

[![1](/assets/images/malware-analysis/copperstealer-rootkit/1.png)](/assets/images/malware-analysis/copperstealer-rootkit/1.png)

## IRP_MJ_SET_INFORMATION dispatch routine

It starts with checking the name of the file that's the target of the operation, and if it same as the rootkit file path it will prevent it. 

You may check the member of the IRP object that's access from [Here ](https://www.vergiliusproject.com/kernels/x86/windows-xp/sp3/IRP)  from [vergiliusproject](https://www.vergiliusproject.com/) because it uses an old version of the object (it access the IO_STACK_LOCATION, then FILE_OBJECT, then the UNICODE_STRING of the file name).

As shown in the code:

[![2](/assets/images/malware-analysis/copperstealer-rootkit/2.png)](/assets/images/malware-analysis/copperstealer-rootkit/2.png)


It returns STATUS_ACCESS_DENIED if so ,otherwise it will pass it normally. And this code is just mainly used to prevent any edit of the file or an attempt to DELETE the file.

> [*] you know that cmd can pass this technique of delete protection?
> CMD uses 2 ways to delete a file:
>  - One with calling **NtSetInformationFile** with _FileInformationClass_ set to _FileDispositionInformation_ or _FileDispositionInformationEx_, then set the FILE_DISPOSITION_INFORMATION (or FILE_DISPOSITION_INFORMATION_EX) with the appropriate values. Those actions will cause creation of IRP_MJ_SET_INFORMATION so the rootkit will catch it.
> - And the other way is to open the file with DELETE_ON_CLOSE flag, then close the handle. Those actions creates IRP_MJ_CREATE. So actually the rootkit won't catch it so it will be removed.
>
> I have wrote a minifilter project that  prevents those 2 techniques, you may check it: [DeleteProtector](https://github.com/OmarShehata11/DeleteProtector-minifilter-Driver)



## IRP_MJ_READ dispatch routine

This IRP is generated when there's any attempt to read a file object, so the rootkit uses this to detect any read attempt to the malware files, like the cookies file and log and login data...etc. 

It first starts with comparing the name of the file object (as before) with some encrypted strings

[![3](/assets/images/malware-analysis/copperstealer-rootkit/3.png)](/assets/images/malware-analysis/copperstealer-rootkit/3.png)


The decryption done by initializing the string with ``"REGISTRY\MACHINE\SOFTWARE"`` as a wide string, then do those athematic and math operations one it. Those are the strings after decryption:

```
\cookie.db
\cookies.sqlite
\Login Data
\Cookies
\WebCacheV01
```

And if it found that the file name contains any of those strings in it, it will go to the next step which is checking which process is accessing it.

[![4](/assets/images/malware-analysis/copperstealer-rootkit/4.png)](/assets/images/malware-analysis/copperstealer-rootkit/4.png)


As you can see, after it gets the process name, it compares that name with a list of names of allowed processes. And also those name are encrypted using the same way:

[![5](/assets/images/malware-analysis/copperstealer-rootkit/5.png)](/assets/images/malware-analysis/copperstealer-rootkit/5.png)


The encrypted strings:
```
\explorer.exe
\firefox.exe
\chrome.exe
\opera.exe
\Yandex.exe
\baidu.exe
\MicrosoftEdge.exe
\MicrosoftEdgeCP.exe
\rundll32.exe
```

> SIDE NOTE: The driver gets the handle of the process that asks for the read action by calling `PsGetCurrentProcessId()`, because as you know that thread of the calling process is actually the same one that runs that kernel code now.
> So whenever a user mode process asks for a kernel action, the thread of that process goes from user mode to kernel mode with of course changing some values of the ETHREAD of it (the TCB: Thread Control Block) then runs that driver's code, then come back again to user-mode with the return.



## Registry Callback
Another action that this rootkit do is to register a callback routine for the registry, so whenever any modification happens in the Registry, the rootkit will be notified and the callback routine that was register will be called.

[![6](/assets/images/malware-analysis/copperstealer-rootkit/6.png)](/assets/images/malware-analysis/copperstealer-rootkit/6.png)


So as it appears in the screenshot, the callback check if there's an attempt to *Delete* or *SetValue* (by checking ***RegNtSetValueKey*** and ***RegNtDeleteKey*** flags), it will compare that target registry path with the rootkit registry path (the rootkit set this value to its registry path in the Driver Entry), and if so it will return STATUS_ACCESS_DENIED. 

So this is another way to protect itself from being deleted.

## YARA Rule

> Coming soon...


## Conclusion

As it appears that the rootkit is a very valuable and strong weapon for threat actors, it can monitor and modify the system in a way that's very hard to security engineers to detect or resolve. 
It's becoming widely used again now in many attacks, as it serves the attack by hiding or protecting it, or in more advanced way. 


*SEE YA*
