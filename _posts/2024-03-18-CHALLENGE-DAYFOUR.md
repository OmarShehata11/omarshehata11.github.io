---
title: "100DayOfReverseEngineering : Day4"
classes: wide
tags: [100DayOfReverseEngineering]
header:
    teaser: /assets/images/100Day/day4.jpeg
description: Day 4 Reversing Password Checker  
categories:
  - Reverse Engineering
toc: true
ribbon: red
---

# Day 4 : Can you see me 

## Introduction

DAY 4 FROM ***#100DayOfReverseEngineering***. 

> Is it better to have  passion, or responsibility ?


## Day 4 hadith 

>  عَنْ أَبِي هُرَيْرَةَ عَبْدِ الرَّحْمَنِ بْنِ صَخْرٍ رَضِيَ الله تَعَالَى عَنْهُ قَالَ: سَمِعْتُ رَسُوْلَ اللهِ **ﷺ** يَقُوْلُ: **(مَا نَهَيْتُكُمْ عَنْهُ فَاجْتَنِبُوهُ وَمَا أَمَرْتُكُمْ بِهِ فأْتُوا مِنْهُ مَا اسْتَطَعْتُمْ؛ فَإِنَّمَا أَهْلَكَ الَّذِيْنَ مِنْ قَبْلِكُمْ كَثْرَةُ مَسَائِلِهِمْ وَاخْتِلافُهُمْ عَلَى أَنْبِيَائِهِمْ)** رواه البخاري ومسلم

## Challenge Info
Link to [Challenge](https://cybertalents.com/challenges/malware/can-you-see-me) from *CyberTalent*

## Reverse

It's a simple ELF file, nothing special from its header. When we load it into IDA and get into ```main``` function we see so :

[![?](/assets/images/100Day/day4/1.png)](/assets/images/100Day/day4/1.png)

it first check if I passed to it 3 arguments, and if so It's gonna deal only with the last argument, It should be then our needed flag, and the length of it should be 22.

Then it enters a long loop, get a value from the call to ```random``` function (just local function that return: ```73 * seed % 22```. the seed is a global variable that's signed to ```50607081```. and it incremented every time the function called) :

[![?](/assets/images/100Day/day4/2.png)](/assets/images/100Day/day4/2.png)

since it's uses % 22, so the output will be ranged from 0 to 22 as max, And it uses this value as an index to access an array (through it's hard-coded address) and check if the entry from the index is zero or not, if yes it will set it to one and do the main check.

it uses the same random number as an index to access my input, and a counter to access another array. This array is an array of pointer, that holds 22 entry each a pointer to memory that (this array holds only 15 unique pointer). Those pointers points to a strings that holds only spaces with different length. 

It will check the value from my input using the random generated number as an index, to the length of the string that's hold by ```xflag``` array using the counter as an index. 

So now we can reverse the process since we know how it do the check. For that I wrote this script that's going to automate this for us : 

```python
def Random(seed):
	return (73 * seed % 22)
	
seed =  50607081
RA_flagX = 0x1040 
baseAddress = 0x8048000
filename = "canyouseeme"
MAX_LENGTH = 150
addresses = []
xflag = []
checkArray = []
flag = []
counter = 0
for i in range(0, 22):
	checkArray.append(0)
	flag.append(".")

with open(filename, "rb") as f:
	f.seek(RA_flagX) # read the addresses
	data = f.read(88)

	for i in range(0, len(data), 4):
		addresses.append((int.from_bytes(data[i:i+4], byteorder="little", signed=True)) - baseAddress)

	for address in addresses:
		f.seek(address, 0)
		data = f.read(MAX_LENGTH).split(b'\x00')
		#print(data)
		xflag.append(len(data[0]))
	
	for i in range(0, 2202222): # main loop
		seed += 1
		rand_val = Random(seed)
		if checkArray[rand_val-1] == 0:
			checkArray[rand_val-1] = 1
			flag[rand_val] = chr(xflag[counter])
			counter += 1

	print("done.")

	for i in flag:
		print(i, end="")

	f.close()
```

First I initialized some needed arrays. Then I read the content of xflag array (22 entry * 4 byte size each = 88 byte to be read). And because this file is a ```little indian```, we need to rearrange the bytes to get the correct address. So I used the ```int.from_bytes()``` function for every 4 bytes to rearrange the addresses, and the result is subtracted from the **image base address** to get the relevant since I'm going to use it with the ```read()``` function to read the content of the strings.



For every address in this array, I'm going to get the content of it by reading a specific size (I don't know the sizes of the strings so I supposed that at max I'm going to read 150 byte) and split the result according to the null byte, and take the first entry only (that should be the needed string). Then store the length of it.

Now we got all the needed data, So let's get into the real work. In the main loop I just did as the challenge app did, but instead of comparing the result I'm going to store it, and print the flag. 

[![?](/assets/images/100Day/day4/3.png)](/assets/images/100Day/day4/3.png)



> flag : flag{hidden_in_spaces}



Note: If I did any mistake, you can reach me from my e-mail, linkedIn, etc. to notify me about anything.

***SEE YA NEXT CHALLENGE***

