---
title: "IRC_Botnet Variant Torjan deep analysis"
classes: wide
tags: [BOTNET]
header:
    teaser: /assets/images/myIcon/background_botnet.jpg
description: "In this article, I'm going to do deep analysis on this botnet variant that's still active in the wild"
categories:
  - Malware Analysis
toc: true
ribbon: DodgerBlue
---



# IRC_Botnet Variant Torjan

The malware sample is a variant for IRC_BOTNET family, it loads through NSIS  installer, has a many malicious techniques from clipboard hijacking, worm functionalities, and botnet ...etc.



# Malware Details

> sha256: 75d4e24d52a18ef64488fe77b0f6b713ce4b1a484156a344f5cc84fce68e7512
>
> virusTotal: https://www.virustotal.com/gui/file/75d4e24d52a18ef64488fe77b0f6b713ce4b1a484156a344f5cc84fce68e7512
>
> MalShare: https://malshare.com/sample.php?action=detail&hash=75d4e24d52a18ef64488fe77b0f6b713ce4b1a484156a344f5cc84fce68e7512



# Technical summary

1. the malware uses NSIS installer to install itself, the malware firstly consists of 1 DLL (called supersonics.dll) and a cab file (called Blindworm.cab). The cab file has the key and some APIs that the DLL going to import dynamically and the payload itself all encrypted, the whole file is encrypted through a key that the DLL will resolve dynamically according to the cab file name, then after this decryption it well use the key that was decrypted from the cab to again decrypt the rest of the file (APIs, and Payload).
2. the DLL uses some anti-analysis techniques like mouse cursor check and CPU tick count, and also it identifies many useless variables and call many functions that has no affect on the actual functionality, also it resolves all its APIs dynamically.
3. the payload uses anti-vm technique to check the existence of a VM through checking some strings on the running process, and if so it will create a bat file to delete itself
4. it achieves the persistence by putting itself under some Keys in the registry, and also under %appdata% and %temp% paths.
5. The malware spreads across the system through USB and network drivers, it install itself using the old technique (.inf file and .vbs file) and also creates a .lnk file using the COM object.
6.  Also do spreads itself by searching for web hosting directories and replaces every zip, rar (and it uses the CRC32 checksum to make those files appear correctly) and exe files with it under the name of README.txt.scr.
7. it do a clipboard hijacking by replacing every cryptocurrency wallet address with itself in the clipboard.
8. and finally it connects to the C2C through hard-coded IP and hostnames, using the IRC commands to weather send info about the system or join specific channel or download files or decrypt some data 



# Deep Analysis

## 1- Unpacker

First of all, when the malware is installed through the NSIS installer to hide itself, when I decompressed it using 7zip, I found 2 interested files (the Blindworm.cab and supersonics.dll) and a normal system.dll that has no malicious functionality.

Analyzing both files, the .cab seems to hold an encrypted data, and the .dll seems to do the decryption of it.

Inside the ```DllMain()``` function, it calls a function using the ```atexit()``` API, that function has all our code, firstly it initialize a lot of variables and do a lot of loops that are useless has no affect on the actual functionality, and it also initialize some variables with some weird sentences just to lower its **Entropy**. 



[![1](/assets/images/malware-analysis/botnet-variant/1.png)](/assets/images/malware-analysis/botnet-variant/1.png)




 It resolves the ```CreateFile()``` API dynamically, and use it to get handle to the .cab file, then read its data:



[![2](/assets/images/malware-analysis/botnet-variant/2.png)](/assets/images/malware-analysis/botnet-variant/2.png)




It generate the key within a loop using some local values and the .cab file name, and also take the first step to decrypt the .cab file by the second loop using that generated key:



[![3](/assets/images/malware-analysis/botnet-variant/3.png)](/assets/images/malware-analysis/botnet-variant/3.png)




the decryption process used by this malware has three phases

1. the DLL decrypt the .cab file using a dynamically generated key,

2. then after that, it will take a second key from that file (the key resides from offset 258F -> 2A89 (RVA)), then it will use that to decrypt some data from the same file (from offset 2A89 -> 2CA6 (RVA)), this region holds many APIs string that the DLL going to use and import dynamically,

3. then it will use the same key to decrypt the rest of the file (2CA6 -> the end), which seems to hold an PE file (the payload itself).

   

[![30](/assets/images/malware-analysis/botnet-variant/30.png)](/assets/images/malware-analysis/botnet-variant/30.png)



and this is the code of decrypting the API list :



[![4](/assets/images/malware-analysis/botnet-variant/4.png)](/assets/images/malware-analysis/botnet-variant/4.png)




and for decrypting the rest of the file :



[![5](/assets/images/malware-analysis/botnet-variant/5.png)](/assets/images/malware-analysis/botnet-variant/5.png)




and between that he inset some anti-debugging technique by checking the mouse cursor position and CPU tick count, and if found it will go for infinity loop, and it's easy to avoid it actually.

So to decrypt those data, I wrote this code to this for us using the key generated dynamically and continue as the DLL just did :

```cpp
#include <Windows.h>
#include <iostream>
#include <stdlib.h>
#include <iomanip>


using namespace std;


int key[] = {
    0x10, 0x61, 0x4A, 0x73, 0x24, 0xBD, 0x7E, 0x97, 0x60, 0x79, 0x12, 0x03, 0x1C, 0x1D, 0x6E, 0x47,
    0x60, 0x31, 0xAA, 0x6B, 0x84, 0x7D, 0x66, 0x0F, 0x10, 0x09, 0x0A, 0x7B, 0x54, 0x6D, 0x3E, 0xA7,
    0x58, 0xB1, 0x4A, 0x53, 0x3C, 0x2D, 0x36, 0x37, 0x48, 0x61, 0x5A, 0x0B, 0x94, 0x55, 0xBE, 0x47,
    0x40, 0x29, 0x3A, 0x23, 0x24, 0x55, 0x7E, 0x47, 0x18, 0x81, 0x42, 0xAB, 0x54, 0x4D, 0x26, 0x37,
    0x50, 0x51, 0x22, 0x0B, 0x34, 0x65, 0xFE, 0x3F, 0xD8, 0x21, 0x3A, 0x53, 0x44, 0x5D, 0x5E, 0x2F,
    0x18, 0x21, 0x72, 0xEB, 0x2C, 0xC5, 0x3E, 0x27, 0x40, 0x51, 0x4A, 0x4B, 0x3C, 0x15, 0x2E, 0x7F,
    0xD8, 0x19, 0xF2, 0x0B, 0x14, 0x7D, 0x6E, 0x77, 0x78, 0x09, 0x22, 0x1B, 0x4C, 0xD5, 0x16, 0xFF,
    0x18, 0x01, 0x6A, 0x7B, 0x64, 0x65, 0x16, 0x3F, 0x08, 0x59, 0xC2, 0x03, 0xEC, 0x15, 0x0E, 0x67,
    0x88, 0x91, 0x92, 0xE3, 0xCC, 0xF5, 0xA6, 0x3F, 0xF0, 0x19, 0xE2, 0xFB, 0x94, 0x85, 0x9E, 0x9F,
    0xF0, 0xD9, 0xE2, 0xB3, 0x2C, 0xED, 0x06, 0xFF, 0xE8, 0x81, 0x92, 0x8B, 0x8C, 0xFD, 0xD6, 0xEF,
    0x80, 0x19, 0xDA, 0x33, 0xCC, 0xD5, 0xBE, 0xAF, 0xB8, 0xB9, 0xCA, 0xE3, 0xDC, 0x8D, 0x16, 0xD7,
    0x20, 0xD9, 0xC2, 0xAB, 0xBC, 0xA5, 0xA6, 0xD7, 0xF0, 0xC9, 0x9A, 0x03, 0xC4, 0x2D, 0xD6, 0xCF,
    0xD8, 0xC9, 0xD2, 0xD3, 0xA4, 0x8D, 0xB6, 0xE7, 0x70, 0xB1, 0x5A, 0xA3, 0xBC, 0xD5, 0xC6, 0xDF,
    0xC0, 0xB1, 0x9A, 0xA3, 0xF4, 0x6D, 0xAE, 0x47, 0xB0, 0xA9, 0xC2, 0xD3, 0xCC, 0xCD, 0xBE, 0x97,
    0x90, 0xC1, 0x5A, 0x9B, 0x74, 0x8D, 0x96, 0xFF, 0xE0, 0xF9, 0xFA, 0x8B, 0xA4, 0x9D, 0xCE, 0x57,
    0x88, 0x61, 0x9A, 0x83, 0xEC, 0xFD, 0xE6, 0xE7, 0x98, 0xB1, 0x8A, 0xDB, 0x44, 0x85, 0x6E, 0x97,
    0x70, 0x19, 0x0A, 0x13, 0x14, 0x65, 0x4E, 0x77, 0x28, 0xB1, 0x72, 0x9B, 0x64, 0x7D, 0x16, 0x07,
    0x00, 0x01, 0xAB, 0xAB, 0xAB, 0xAB, 0xAB, 0xAB, 0xAB, 0xAB, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00
};

int keyLength = 274;

int main(int argc, char const* argv[])
{
    HANDLE hFile;
    HANDLE hPayload;

    size_t fileSize;
    size_t payloadFileSize;

    char* fileBufferData = NULL;
    char* scndKey = NULL;

    DWORD datauseless;
    

    /* 
    API REGION : (9615 + 1274) --> ( + 541)
    PAYLOAD REGION : 
    */
    int firstOffset = 9615;
    int scndOffset = 1274;
    int thirdOffset = 541;
    int fourthOffset = 2333;
    int apiOffset = firstOffset + scndOffset;
    int payloadOffset = apiOffset + thirdOffset + fourthOffset;
    int newFileSize;
    UCHAR* DecompressedFileData = NULL;

    hFile = CreateFileA("C:\\Users\\Omar Shehata\\Desktop\\malware analysis decrypt the IRC BOTNET\\Malware_encoded_data",
        GENERIC_READ, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);

    if (hFile == INVALID_HANDLE_VALUE)
    {
        cout << "error while opening the file\n";
        return -1;
    }

    fileSize = GetFileSize(hFile, NULL);
    fileBufferData = (char *)malloc(fileSize);

    if (fileBufferData == NULL)
    {
        cout << "Error while allocating the memory spcace" << endl;
        return -1;
    }

    ReadFile(hFile, fileBufferData, fileSize, &datauseless, NULL);

    cout << "first bytes of data are : ";

    for (int i = 0; i < 100; i++)
    {
        cout << hex << setw(2) << setfill('0') << (0xff & fileBufferData[i]) << " ";
    }

    for (int counter = 0; counter < fileSize; counter++)
        fileBufferData[counter] = fileBufferData[counter] ^ key[counter % keyLength];

    cout << "the decryption is done" << endl;

    cout << "the first bytes of the data are : ";

    for (int i = 0; i < 100; i++)
        cout << hex << setw(2) << setfill('0') <<(0xff & fileBufferData[i] )<< ", ";
    
    scndKey = &fileBufferData[firstOffset];

    for (int i = 0; i < 541; i++)
    {
        fileBufferData[apiOffset + i] ^= scndKey[i];
        
    }

    cout << "api decryption done.\n the APIs are : \n";
    for (int i = 0; i < 541; i++)
        cout << fileBufferData[apiOffset + i];

    newFileSize = fileSize - firstOffset - scndOffset - thirdOffset - fourthOffset;
    cout << "the new file size is (size of the payload = " << newFileSize << endl;

    for (int i = 0; i < newFileSize; i++)
    {
        fileBufferData[i + payloadOffset] ^= scndKey[i % scndOffset];
      //  cout << fileBufferData[payloadOffset + i] << " ";
    }
    cout << "the data of the payload has been decrypted.\n";

    hPayload = CreateFileA("C:\\Users\\Omar Shehata\\Desktop\\malware analysis decrypt the IRC BOTNET\\Malware_Decrypted.omar",
        GENERIC_READ|GENERIC_WRITE, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

    if (hPayload == INVALID_HANDLE_VALUE)
    {
        cout << "error while creating the new file, error code : " << GetLastError() << endl;
        return -1;
    }
    bool isWritten = WriteFile(hPayload, &fileBufferData[payloadOffset + 4], newFileSize, &datauseless, NULL);
    
    if (!isWritten)
    {
        cout << "error couldn't write to the file, error code : " << GetLastError() << endl;
        return -1;
    }

    cout << "the write to the file is done.\n";

    CloseHandle(hPayload);
    CloseHandle(hFile);
    free(fileBufferData);

    return 0;
}
```

> the malware assign the decrypted APIs into an array of strings, the APIs is stored in the encrypted file and separated by a new line.

The code above is working very well actually but it may face some problem while decrypting the payload, although the header is decrypted well; but I found it easy to just dump it dynamically while the DLL is running.

The DLL then create a process of itself in suspend state, then it empty the data inside it using the ```NtUnmapViewOfSection()``` API, then it write the decrypted file into that process and then resume the thread (Process hollowing).



[![6](/assets/images/malware-analysis/botnet-variant/6.png)](/assets/images/malware-analysis/botnet-variant/6.png)


[![7](/assets/images/malware-analysis/botnet-variant/7.png)](/assets/images/malware-analysis/botnet-variant/7.png)


[![8](/assets/images/malware-analysis/botnet-variant/8.png)](/assets/images/malware-analysis/botnet-variant/8.png)




So just before it resumes the thread, I get to the memory map of that process and dumped its data and done you have the payload.

> Note: you may face a problem when you load the DLL directly into the debugger, so instead load the installer itself into the debugger and set a breakpoint on loading DLLs:
>
> [![9](/assets/images/malware-analysis/botnet-variant/9.png)](/assets/images/malware-analysis/botnet-variant/9.png)


> Another note: you can find that the malware resolves the APIs from the encrypted .cab file from an array, so I wrote this just simple script to give it the index used to get the API in the code and it will give you that string:
> ```cpp
> #include <stdio.h>
> 
> int main()
> {
> 	int index;
> const char* apiTable[] = {
>     "CreateProcessA",
>     "NtUnmapViewOfSection",
>     "VirtualAllocEx",
>     "VirtualAlloc",
>     "WriteProcessMemory",
>     "GetThreadContext",
>     "SetThreadContext",
>     "ResumeThread",
>     "GetFileSize",
>     "ReadProcessMemory",
>     "ntdll.dll",
>     "LocalAlloc",
>     "Sleep",
>     "GetModuleFileNameA",
>     "GetCursorPos",
>     "NtResumeThread",
>     "user32",
>     "lstrcatA",
>     "ExitProcess",
>     "GetCommandLineA",
>     "lstrlenA",
>     "ExpandEnvironmentStringsA",
>     "RtlDecompressBuffer",
>     "Advapi32",
>     "InitializeAcl",
>     "SetSecurityInfo",
>     "GetTickCount",
>     "\\\\msiexec.exe",
>     "IsWow64Process",
>     "GetSystemWow64DirectoryA",
>     "GetSystemDirectoryA",
>     "GetCurrentProcess",
>     "lstrcmpiA",
>     "%APPDATA%\\\\%s.exe",
>     "wsprintfA",
>     "CopyFileA",
>     "CreateRemoteThread"
> };
> 
> 
> 	while(1)
> 	{
> 		scanf("%d", &index);
> 		printf("%s\n", apiTable[index]);
> 	}
> 	return 0;
> 
> }
> ```



## 2- Analyzing the payload

The payload first checks if there's a VM running by checking some strings like "vm" or so, and if found, it will create a .bat file that holds a script for self deletion. The malware also uses a mutex to make it only run once in the system, the mutex name is "b4".



[![10](/assets/images/malware-analysis/botnet-variant/10.png)](/assets/images/malware-analysis/botnet-variant/10png)




> Note: the malware do many API hammering by calling many APIs in useless way

## Persistance

The malware achieves that by installing itself under the ``` %windir%, %userprofile%, %temp%``` directory, by making a directory called ``` M-505072972047509246024758274085628272046520``` and install itself under this with the name of ```winmgr.exe``` to look legitimate  (and for some reasons, the M-505072972047509246024758274085628272046520 directory is hidden and can't be shown in the directory browser or even cmd). Also it install itself under the ```SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run\\``` registry key.



[![11](/assets/images/malware-analysis/botnet-variant/11.png)](/assets/images/malware-analysis/botnet-variant/11.png)


[![12](/assets/images/malware-analysis/botnet-variant/12.png)](/assets/images/malware-analysis/botnet-variant/12.png)




## AV-Evade

The malware applies many technique to make itself legitimate, from naming itself as winmgr, through putting itself into the authorized apps in the firewall policy in the registry, and also disabling the win defender and removing its zone.identifier ADS.

> Note: the malware deletes its zone.identifier ADS (Alternative Data Stream) just to make sure it to run normally without any warning that it's downloaded from an untrusted resource from the internet according to the zone value.



[![13](/assets/images/malware-analysis/botnet-variant/13.png)](/assets/images/malware-analysis/botnet-variant/13.png)


[![14](/assets/images/malware-analysis/botnet-variant/14.png)](/assets/images/malware-analysis/botnet-variant/14.png)




And the malware makes sure that's its running under the Temp path by create a process of it from there.

## Threads Actions:

the malware runs 4 threads in the background, every one to do a specific job:

#### THREAD 1: Worm Functionality across USB and Network Drivers

The malware enters an infinity loop while checking if there's any device connected that's type 2 or 4 (DRIVER_REMOABLE and DRIVER_REMOTE), and if so it will check if it's not has a symbolic link of A or B (just the named used before for floppy disks), then it will spread it self across it.

[![15](/assets/images/malware-analysis/botnet-variant/15.png)](/assets/images/malware-analysis/botnet-variant/15.png)


It spreads itself by 2 ways, one is old and one that can be run today. The first by creating an ```autorun.inf``` file and a ```script.vbs``` file, in the old versions of windows, when the USB enters the computer the autorun file inside it will run, so it puts a script in that .inf file to run the .vbs code, and the vbs code just run the malware that was copied in that USB ( the script of the vbs or .inf file is mentioned inside the binary itself).

[![16](/assets/images/malware-analysis/botnet-variant/16.png)](/assets/images/malware-analysis/botnet-variant/16.png)




> the malware uses randomness in the code many times for two things:
>
> 1. renaming the file itself, it may use a random string 
> 2. insert a random values inside the  binary file itself to just make sure that every copy of the malware has different hash so it won't be caught  easy by the AV (ANTI-AV Technique)

and if this way couldn't work, it will generate a ```.lnk``` file using the COM object to make this file points to the malware itself, so when the user run this file he will actually run the malware code.

[![17](/assets/images/malware-analysis/botnet-variant/17.png)](/assets/images/malware-analysis/botnet-variant/17.png)




and at the last, it will try to remove any other file in the USB that may conflict with the script files mentioned before.



#### THREAD 2: Web Hosting Files Infection

this thread also do a worm functionality by replacing every ```.zip, .rar or .exe``` file inside the **Web Hosting** Directories that may exist (if it's running on a web server). For the .exe file, it removes the whole file and replace it with a copy of the malware with putting some random bytes at the end of the file to change the hash of it. And for .zip or .rar files, it tries to replace the data with a malicious one (with also try to not corrupt the format by putting a correct CRC32 checksum and header and so) and finaly putting them under the name of ```"README.txt.scr"``` (which can be executed normally as a normal executable file).

[![18](/assets/images/malware-analysis/botnet-variant/18.png)](/assets/images/malware-analysis/botnet-variant/18.png)


[![19](/assets/images/malware-analysis/botnet-variant/19.png)](/assets/images/malware-analysis/botnet-variant/19.png)


[![20](/assets/images/malware-analysis/botnet-variant/20.png)](/assets/images/malware-analysis/botnet-variant/20.png)


> so if this is done correctly and the web was infected, every one trying to install any archieve or exe from the web is going to download the malware



#### THREAD 3: Clipboard Hijacking

In this thread, the malware do the clipboard hijacking by simply trying to get a handle to the clipboard, and if it found any possibility that it contains an address to a cryptocurrency wallet, then it will replace it with an address of the malware author itself according  of course to the type of cryptocurrency that wallet holds.



[![21](/assets/images/malware-analysis/botnet-variant/21.png)](/assets/images/malware-analysis/botnet-variant/21.png)

[![22](/assets/images/malware-analysis/botnet-variant/22.png)](/assets/images/malware-analysis/botnet-variant/22.png)



#### THREAD 4: C2C Operations

This thread is responsible for any Command and Control actions, from connecting to the malware author, through sending the messages and receiving it.

The malware tries first to create a sock with a hard coded IP and port number (220.181.87.80:5050) and other hostnames, and if it success, it will try to send the Locale data of the device (like the country region and so). It uses a function that can do five things:

USER - PONG - JOIN - PRIVMSG - NICK

those are instructions that are used in IRC protocol, you can find details about it or any other instructions from this great [website]([User Commands - IRC-nERDs](https://irc-nerds.com/wiki/index.php/User_Commands)). 



[![23](/assets/images/malware-analysis/botnet-variant/23.png)](/assets/images/malware-analysis/botnet-variant/23.png)



It starts first by sending the instruction NICK to change it's nick name on the server according to the Locale info of the device infected and with some random value.



[![24](/assets/images/malware-analysis/botnet-variant/24.png)](/assets/images/malware-analysis/botnet-variant/24.png)



After so it tries to receive any data that is sent, then it will going to sperate the command arguments through the space and store each in an entry into an array. After so It did some comparison to check what should it do, for example if it received "001", it's going to join a channel called #net#, if received PING it's going to response PONG and so.



[![25](/assets/images/malware-analysis/botnet-variant/25.png)](/assets/images/malware-analysis/botnet-variant/25.png)



Lastly it does another check if no option inside this function achieved, by checking other options and reformatting the commands received, and go inside a loop to execute every argument that was sent in the instruction :



[![26](/assets/images/malware-analysis/botnet-variant/26.png)](/assets/images/malware-analysis/botnet-variant/26.png)



Further checking also done in the ```mw_execute_recieved_commands()``` function above, like if it received ```rmrf``` it's going to delete itself, or ```su``` to join a channel (#USER OR #ADMIN) according to the user that executed the malware, and whether it's x64 or x86 if he sent ```sa``` ...etc.



[![27](/assets/images/malware-analysis/botnet-variant/27.png)](/assets/images/malware-analysis/botnet-variant/27.png)



Also if he sent the command ```d```, it do a decryption twice with the data after this argument (the second decryption is done using XOR with the key "trk"), and then it will try to connect to the internet using the user agent : ```Mozilla/5.0 (Windows NT 6.1; WOW64; rv:22.0) Gecko/20100101 Firefox/22.0```, and connect and get the home page of the website : ```http://api.wipmania.com/```, and take the command from it after the '>' string. And using other option like "x" or "u" it's going to download a file from the internet according to the link that was encrypted sent as an argument



[![28](/assets/images/malware-analysis/botnet-variant/28.png)](/assets/images/malware-analysis/botnet-variant/28.png)



and using other user agent : ```Mozilla/5.0 (Windows NT 6.1; WOW64; rv:22.0) Gecko/20100101 Firefox/22.0```, it's going to connect to the internet and download the file that was sent to the malware, then delete its ```Zone.Identifier``` ADS from it then lastly download it under the %TEMP% path using a random generated name, and execute it and make sure that it was executed.



[![29](/assets/images/malware-analysis/botnet-variant/29.png)](/assets/images/malware-analysis/botnet-variant/29.png)



## IOCs

| IOC Type   | Indicator                                                    |
| ---------- | ------------------------------------------------------------ |
| Hash       | 75d4e24d52a18ef64488fe77b0f6b713ce4b1a484156a344f5cc84fce68e7512 |
| Dir name   | M-505072972047509246024758274085628272046520                 |
| User-Agent | Mozilla/5.0 (Windows NT 6.1; WOW64; rv:22.0) Gecko/20100101 Firefox/22.0 |
| URL        | http://api.wipmania.com/                                     |
| User-Agent | Mozilla/5.0 (Windows NT 6.1; WOW64; rv:22.0) Gecko/20100101 Firefox/22.0 |
| IP         | 220.181.87.80                                                |
| IP:Port    | 220.181.87.80:5050                                           |


# YARA rule

```c
import "pe"

rule irc_boot_var {
	meta:
		author = "0xefe4"
		description = "yara rule for irc botnet variant"
		sample = "https://omarshehata11.github.io/malware%20analysis/IRC-BOTNET-VARIANT/"
		date = "2025-04-03"
		yarahub_uuid = "b128dcb5-a430-4bf8-bb14-77b9c41ac16c"
		yarahub_license = "CC0 1.0"
		yarahub_rule_sharing_tlp = "TLP:WHITE"
		yarahub_reference_md5 = "0b089978de701f3953689fa3ac98cd76"

	strings:
		// temp file name (very specific)
		$tempFileName = "M-505072972047509246024758274085628272046520" nocase ascii wide
	
		// path/file indicators 
		$s1 = "%windir%" ascii wide nocase
		$s2 = "%userprofile%" ascii wide nocase
		$s3 = "%temp%" ascii wide nocase
		$s4 = ".exe" ascii wide nocase
		$s5 = ".zip" ascii wide nocase
		$s6 = ".rar" ascii wide nocase

		// wallet addresses
		$w1 = "1of6uEzx5qfStF1HrVXaZ1eE3X4ntnbsx"
		$w2 = "t1JQVnfzuqRWWAaYgvpXT4PzZzo2M3scQja"
		$w3 = "LMimQj9RBdYbDTsV6k37TneN2Svi4e1PXF"
		$w4 = "4BrL51JCc9NGQ71kWhnYoDRffsDZy7m1HUU7MRU4nUMXAHNFBEJhkTZV9HdaL4gfuNBxLPc3BeMkLGaPbF5vWtANQni58KYZqH43YSDeqY"
		$w5 = "DHDUtYKHtEU9w9Scyan47L2YhKiVqhpXxH"
		$w6 = "B5f1bkbcmXzwZtL5ua5HYFHKxFz3HFcNi8"
		$w7 = "EcqcbRssS5tMx1WKAVeT2KUcFWaWueywoz"
		$w8 = "PRXRECu2m4gXtYFYPDpVAmYr5qM4u6UECk"

		// network signatures 
		$n_ua = "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:22.0) Gecko/20100101 Firefox/22.0"
		$n_url = "http://api.wipmania.com/"
		$n_ip = "220.181.87.80"
		$n_ip_p = "220.181.87.80:5050"

		// IRC indicators
		$irc1 = "#ADMIN"
		$irc2 = "#USER"
		$irc3 = "#X64"
		$irc_msg_format = "%s %s \"\" \"x\" :%s\r\n"


		// code patter for time stamp signature 
		  			/*	
		  				*a2 = SystemTime.wDay | (32 * SystemTime.wMonth) | ((SystemTime.wYear - 1980) << 9);
						  result = a1;
						  *a1 = (SystemTime.wSecond / 2) | (32 * SystemTime.wMinute) | (SystemTime.wHour << 11);
						  return result;
						
						  ASSEMBLY CODE
						.text:00845B0B 0B CA                                or      ecx, edx
						.text:00845B0D 0F B7 45 F6 (??)                     movzx   eax, [ebp+SystemTime.wDay]
						.text:00845B11 0B C8                                or      ecx, eax
						.text:00845B13 8B 55 0C   (??)                      mov     edx, [ebp+arg_4]
						.text:00845B16 66 89 0A                             mov     [edx], cx
						.text:00845B19 0F B7 4D F8   (??)                   movzx   ecx, [ebp+SystemTime.wHour]
						.text:00845B1D C1 E1 0B                             shl     ecx, 0Bh
						.text:00845B20 0F B7 55 FA     (??)                 movzx   edx, [ebp+SystemTime.wMinute]
						.text:00845B24 C1 E2 05                             shl     edx, 5
						.text:00845B27 0B CA                                or      ecx, edx
						.text:00845B29 0F B7 45 FC   (??)                   movzx   eax, [ebp+SystemTime.wSecond]
						.text:00845B2D 99                                   cdq
						.text:00845B2E 2B C2                                sub     eax, edx
						.text:00845B30 D1 F8                                sar     eax, 1
						.text:00845B32 0B C8                                or      ecx, eax
						.text:00845B34 8B 45 08      (??)                   mov     eax, [ebp+arg_0]
						.text:00845B37 66 89 08                             mov     [eax], cx
					*/

		$code_time = {
		0B CA 0F B7 45 ?? 0B C8 8B 55 ?? 66 89 0A 0F B7 4D ?? C1 E1 0B 0F B7 55 ?? 
		C1 E2 05 0B CA 0F B7 45 ?? 99 2B C2 D1 F8 0B C8 8B 45 ?? 66 89 08
		 }



	condition:
		uint16(0) == 0x5a4d and
		filesize < 150KB and
		(
			(any of ($w*) and (any of ($n_*))) or 
			
			($tempFileName and any of ($s*)) or 
			
			($tempFileName) or 

			(2 of ($w*)) or 
			
			(2 of ($n_*)) or 

			(2 of ($irc*) and any of ($n_*)) or 

			$code_time
		) 
}
```


# Summary 

this IRC botnet variant do a lot of old and new techniques that still working right now in the wild, from hiding itself inside an encrypted file and installer, through spreading itself across the whole system using some worm techniques, and some other techniques like clipboard hijacking and anti-vm and other, to lastly communication with the C&C using the IRC protocol to send info about the system or even install other malwares.



Thanks very much for reading this, if you found any mistake that I made or you want to ask about anything, don't wait to mail me or send a message through linkedin. 

*SEE YA*



