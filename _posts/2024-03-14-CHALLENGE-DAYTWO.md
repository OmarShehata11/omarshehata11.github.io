---
title: "100DayOfReverseEngineering : Day2"
classes: wide
tags: [100DayOfReverseEngineering]
header:
    teaser: /assets/images/100Day/day2.jpeg
description: Day 2 Reversing simple APK  
categories:
  - Reverse Engineering
toc: true
ribbon: red
---

# Day 2 : ShAPK1 

## Introduction

DAY 2 FROM ***#100DayOfReverseEngineering***. 

> Do we need perfectionism ?


## Day 2 hadith 

> ## عَنْ أَبِيْ رُقَيَّةَ تَمِيْم بْنِ أَوْسٍ الدَّارِيِّ رضي الله عنه أَنَّ النبي **ﷺ** قَالَ:( الدِّيْنُ النَّصِيْحَةُ قُلْنَا: لِمَنْ يَا رَسُولَ اللهِ ؟ قَالَ: للهِ، ولكتابه، ولِرَسُوْلِهِ، وَلأَئِمَّةِ المُسْلِمِيْنَ، وَعَامَّتِهِمْ (رواه مسلم.



## Reverse

At first, we should know that the APK file format is just a ZIP file, so you can decompress the file to get all combined files like the resources and the main modules and so. But for this challenge since it's my first time to deal with APK file I'm going to use directly an APK decompiler (JADX). When we load the file into it this is the modules it shows : 

[![?](/assets/images/100Day/APK/1.png)](/assets/images/100Day/APK/1.png)

Most of those modules are related to java itself. So our only target is the ```com.example.shapk1```. inside it we can see many classes but ```MainActivity``` is interesting one. It contains 3 methods :

- check()
- encrypt()
- onCreate()

let's open up this class :

[![?](/assets/images/100Day/APK/2.png)](/assets/images/100Day/APK/2.png)

our target code to analyze is in the *check()* method. in the *onClick()* function it firstly try to read some string entered from a use, then it do some check, if it success it will print to us "YES! PASSWORD IS CORRECT!!" and if not it will print the fail message.. 

[![?](/assets/images/100Day/APK/3.png)](/assets/images/100Day/APK/3.png)

So how it validate our input?

It do so by first use the **encrypt()** method on our input, then encrypt it with base64 (the second argument in the base64 function it uses is the just the ***NO_WRAP*** flag, won't cause any change for the encryption). then it checks if the result value is equals to value : ***R.string.secret***. and it uses the **getString()** and **getResource()** function on it, so the secret variable in the R class seems to hold the ID to the actual value inside the resources of the APK. 



So now we need to reverse the process to know the actual password it needs to enter. so we need to reverse the **encrypt()** function.



[![?](/assets/images/100Day/APK/4.png)](/assets/images/100Day/APK/4.png)

It's a simple XOR encryption function, it reads the value from the resources with ID stored in*** R.String.key*** then XOR each value inside it with the argument that's passed to it. 



So now, we need to get the values of ***secret*** and ***key***, then we should reverse the process of the validation of the password, by firstly decode the Secret variable with base64, the XOR result with the Key value. 

instead of searching inside the resource with the ID, we can simply search for any string resided in the apk related with string **secret** and **key** :

[![?](/assets/images/100Day/APK/5.png)](/assets/images/100Day/APK/5.png)

[![?](/assets/images/100Day/APK/6.png)](/assets/images/100Day/APK/6.png)

And I wrote this simple script to do the decryption for us : 

```python
secret = "NQALCgEDDDEzUjpTBwocBgcDPTIIGwIK"
key = b'beginning'

import base64

def Dec(sec):
    plainText = ""
    for i in range(len(sec)):
        plainText += chr(sec[i] ^ key[i % len(key)])

    return plainText


base64Dec = base64.b64decode(secret)

print(Dec(base64Dec))
```



> flag: Welcome_T0_4ndroid_World



Note: If I did any mistake, you can reach me from my e-mail, linkedIn ...etc to notify me about anything.

***SEE YA NEXT CHALLENGE***

