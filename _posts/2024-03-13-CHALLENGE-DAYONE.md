---
title: "100DayOfReverseEngineering : Day1"
classes: wide
tags: [100DayOfReverseEngineering]
header:
    teaser: /assets/images/100Day/day1.jpeg
description: Day 1 Reversing simple GUI .NET app
categories:
  - Reverse Engineering
toc: true
ribbon: red
---

# Day 1 : flare-on 2023 - X

## Introduction

My first reverse challenge from ***#100DayOfReverseEngineering***. 

> Is Consistency is all what we need ?


## Day 1 hadith 

> ## عَنْ أَمِيرِ المُؤمِنينَ أَبي حَفْصٍ عُمَرَ بْنِ الخَطَّابِ قَالَ : سَمِعْتُ رَسُولَ اللهِ **ﷺ** يَقُولُ : **" إِنَّمَا الأَعْمَالُ بِالنِّيَّاتِ ، وَإنَّمَا لِكُلِّ امْرِىءٍ مَا نَوَى ، فَمَنْ كَانَتْ هِجْرَتُهُ إِلى اللهِ وَرَسُوله فَهِجْرتُهُ إلى اللهِ وَرَسُوُله ، وَمَنْ كَانَتْ هِجْرَتُهُ لِدُنْيَا يُصِيْبُهَا ، أَو امْرأَةٍ يَنْكِحُهَا ، فَهِجْرَتُهُ إِلى مَا هَاجَرَ إلَيْهِ "**



## Reverse

The app comes with many DLLs and, it's just for it's GUI needs. if we analyzed the exe file with any PE header Analyzer, we will see it's a normal exe, no big entropy, got some resources for it's GUI need and so. nothing special, but the import table has not enough function for GUI, so it seems that it depends on other file to do that work for it. 

[![?](/assets/images/100Day/X/1.png)](/assets/images/100Day/X/1.png)

from the string you may notice that it uses the string X.dll. which is a file that exist in its directory. so that's enough info to start. let's run the app and see what we got..

[![?](/assets/images/100Day/X/2.png)](/assets/images/100Day/X/2.png)

so it's a simple app, we can click some buttons to change the value inside that counters and we can click the (lock) button to see the result, if the numbers is wrong the (X) shape turns to red, so we need to know what's the correct number that we should submit. 
let's analyze the .exe file first, when I did so and load it into IDA, I couldn't find anything special about the GUI and so, it's just make every thing suitable for the GUI to run, so it's not going to be very useful for us to know the correct numbers.. 

[![?](/assets/images/100Day/X/3.png)](/assets/images/100Day/X/3.png)

I think it's related to the .NET framework, whatever. 
so let's now analyze the X.dll file. it's a .NET file 

[![?](/assets/images/100Day/X/4.png)](/assets/images/100Day/X/4.png)

So let's analyze it with .NET decompiler, I'm going to use dotpeek app. And that the result when I load it to the decompiler : 

[![?](/assets/images/100Day/X/5.png)](/assets/images/100Day/X/5.png)

ok now what we really need ? our target mission is to know when we press the lock button the app do some kind of test or check on the values in the counters, and according to so it will decide to change color of the X shape. So let's search for anything related. 
After simple search you will find under the **Game1** class, there's a function called ***_lockButton_Click***, that seems to be interesting name, When we enter it you will find this : 

[![?](/assets/images/100Day/X/6.png)](/assets/images/100Day/X/6.png)

And guess what, this is the flag, and the it checks if the entered value is (***42***) to show the flag for us, let's try : 

[![?](/assets/images/100Day/X/7.png)](/assets/images/100Day/X/7.png)

> FLAG: glorified_captcha@flare-on.com



Note: If I did any mistake, you can reach me from my e-mail, linkedIn ...etc to notify me about anything.

***SEE YA NEXT CHALLENGE***

