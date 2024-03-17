---
title: "100DayOfReverseEngineering : Day3"
classes: wide
tags: [100DayOfReverseEngineering]
header:
    teaser: /assets/images/100Day/day3.jpeg
description: Day 3 Reversing Win game  
categories:
  - Reverse Engineering
toc: true
ribbon: red
---

# Day 3 : PixelPoker 

## Introduction

DAY 3 FROM ***#100DayOfReverseEngineering***. 

> Is it better to have  passion, or responsibility ?


## Day 3 hadith 

> ***عَنْ أَبِي مَالِكٍ الحَارِثِ بنِ عَاصِم الأَشْعَرِيِّ رَضِيَ اللهُ عَنْهُ قَالَ: قَالَ رَسُولُ اللهِ ﷺ : )الطُّهُورُ شَطْرُ الإِيْمَانِ، والحَمْدُ للهِ تَمْلأُ الميزانَ، وسُبْحَانَ اللهِ والحَمْدُ للهِ تَمْلآنِ - أَو تَمْلأُ - مَا بَيْنَ السَّمَاءِ والأَرْضِ، وَالصَّلاةُ نُورٌ، والصَّدَقَةُ بُرْهَانٌ، وَالصَّبْرُ ضِيَاءٌ، وَالقُرْآنُ حُجَّةٌ لَكَ أَو عَلَيْكَ، كُلُّ النَّاسِ يَغْدُو فَبَائِعٌ نَفْسَهُ فَمُعْتِقُهَا أَو مُوبِقُهَا(رواه مسلم.***


## Reverse
> Welcome to PixelPoker ^_^, the pixel game that's sweeping the nation!
>
> Your goal is simple: find the correct pixel and click it
>
> Good luck!

This is the notes comes with the game. When we start the game it appeared as so : 

[![?](/assets/images/100Day/pixel/1.png)](/assets/images/100Day/pixel/1.png)

in the window name, the name of the game appears, (x,y) coordinates and ```#0/10```. this value increases when I attempt to click on the game screen. so it's the counter for wrong attempts. and when I click 10 times wrong it shows me this message :



[![?](/assets/images/100Day/pixel/2.png)](/assets/images/100Day/pixel/2.png)



So to solve this challenge we need to know the exact pixel (x,y) to click. Of course we can't brute-force it, so let's reverse the binaryyy!



When we load the binary into IDA and get to the winMain function (since it's a GUI app), we see this, it calls to two local function, inside those both functions it do the necessarily work to make the GUI works



[![?](/assets/images/100Day/pixel/3.png)](/assets/images/100Day/pixel/3.png)



Inside the first local function, ```sub_401120```, it sets the needed data for the window class, on of this data that's our target to analyze is the *Window procedure*, which points to the function : ```sub_4012C0```.

[![?](/assets/images/100Day/pixel/4.png)](/assets/images/100Day/pixel/4.png) 

The function starts with initializing the window name, (x,y) and counter of wrong attempts : 

[![?](/assets/images/100Day/pixel/5.png)](/assets/images/100Day/pixel/5.png)

> the ```MousePosition``` variable is a 32-bit, the first 16-bit holds the X coordinate, and the remaining 16-bit holds the Y coordinate of the mouse position.
>
> I named those variables according to the order of variables pushed and the string format : ```PixelPoker (%d,%d) - #%d/%d```. 



Then it checks if the number of my wrong attempts equals to 10 or not, and if so it will print the exit message and quit. 

[![?](/assets/images/100Day/pixel/6.png)](/assets/images/100Day/pixel/6.png)

[![?](/assets/images/100Day/pixel/7.png)](/assets/images/100Day/pixel/7.png)

But if my fail counter still under the limit, it do some check, that if the X coordinate of the mouse position is equals to the result of ```dword_412004 % width_window```, and if it's right, it will check Y coordinate if it's == ```dword_412008 % hight_window``` : 

[![?](/assets/images/100Day/pixel/8.png)](/assets/images/100Day/pixel/8.png)

> all the variables above are global variables, I named the hight and width so because they were used in the ```CreateWindowExW``` API as so. 
>
> It uses the ```edx``` register that hold the remaining of the result of div process, so it uses ```%``` not normal division.

If the validation is done, it's going to enter two loops, loop for every coordinate, so it enters every pixel and do some change on it.  For every pixel, it enters a function with this information. 
[![?](/assets/images/100Day/pixel/9.png)](/assets/images/100Day/pixel/9.png)

Inside the function it first get the RGB of the pixel passed to it, then it enters a function (seems to be a RC4 decryption) and get the value from it, then set the RGB value of the pixel again and exit : 

[![?](/assets/images/100Day/pixel/10.png)](/assets/images/100Day/pixel/10.png)



So now we should know how to solve this game, we need to know the values it needs for the mouse coordinates. the X should be = dword_412004 % width_window, and Y = dword_412008 % hight_window. 

>  dword_412004 =1380011078
>
> dword_412008  = 1850682693
>
> and from run time, the width = 741
> hight = 641

So from the needed coordinates should be : (95, 313)
let's try:

[![?](/assets/images/100Day/pixel/11.png)](/assets/images/100Day/pixel/11.png)

> flag: w1nN3r_W!NneR_cHick3n_d1nNer@flare-on.com




Note: If I did any mistake, you can reach me from my e-mail, linkedIn, etc. to notify me about anything.

***SEE YA NEXT CHALLENGE***

