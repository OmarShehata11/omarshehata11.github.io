---
title: "Inverted Call Model & Cancel-safe Framework"
classes: wide
tags: [WDM]
header:
    teaser: /assets/images/myIcon/InvertCallIcone.jpg
description: "Using the Cancel-safe Framework and Inverted Call model for WDM kernel drivers"
categories:
  - Win Kernel Drivers
toc: true
ribbon: Green
---

# Cancel-safe IRP Queue Framework

>  “I never learned from a man who agreed with me.” —Robert A. Heinlein

What's the purpose of the quote? I don't know, I just found that it would be cool if I put it in the article; anyway let's start.

## Introduction

Why we need it? a fair enough question to get start into explaining it. 

We all know the normal flow of a simple request from ring 3 to ring 0. Some user-mode app or service wants to do some action with the hardware of the computer; so it issue a system call to make the kernel do the job for it. like for example the following image :

[![1](/assets/images/kernelDriver/InvertedCall/1.jpg)](/assets/images/kernelDriver/InvertedCall/1.jpg)

as seen, some user-mode app wants to do some sort of file edit, so it issue a system call (by calling  *WriteFile()* API) so the request goes to kernel and I/O manager creates the IRP with any farther data (like IO_STACK_LOCATION) and gives it to the appropriate driver (which in our case is the file system driver) and when it finish the request, the driver will complete the IRP and return. This image also explains the same but with more details: 

[![1.5](/assets/images/kernelDriver/InvertedCall/1.5.png)](/assets/images/kernelDriver/InvertedCall/1.5.png)

But that in one case. So what happen if the driver just not want to complete the IRP at INSTANCE ? they just want to put the request in like a wait list until some sort of event or something happen ? here the Cancel-safe IRP queue framework comes to the stage..

This framework is simply gives you a way to make and manage a wait list for your IRPs, technically put it in a Queue and the framework manage it by set of routines. The framework won't implement the standard operations on the IRP queue, it will just handle the calling mechanism and order to make every thing goes right and we will explain it with an example after a while.



## Framework Routines

Microsoft provides set of routines that **you will implement**, every routine with a specific structure and job to do; but you have just to define the 'way' for it on how to do its job. those are the routines with its main job (from [microsoft documentation](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/cancel-safe-irp-queues)):

[![2](/assets/images/kernelDriver/InvertedCall/2.png)](/assets/images/kernelDriver/InvertedCall/2.png)

 How the framework manage those routines? by a structure called **IO_CSQ**. so this structure should be defined and initialized with calling to the function *IoCsqInitialize* or *IoCsqInitializeEx* (choose between them according to which version you used before *CsqInsertIrp* or *CsqInsertIrpEx*). After Implementing those functions you have just to Initialize the **IO_CSQ** with the function mentioned before.

The implementation of the framework now is done, let's go on how to use it.



## Using the Framework

The framework provides to you some ***macros*** to use the framework through it.

> Remember always that those routines are implemented as macros not function, at some point if you missed that it may cause your code to not work properly, so keep that in mind.

those are the routines :

| routine                  | used for                                                     |
| ------------------------ | ------------------------------------------------------------ |
| ***IoCsqInsertIrp***     | inserts an IRP into the queue                                |
| ***IoCsqInsertIrpEx***   | same as above but returns NTSTATUS value                     |
| ***IoCsqRemoveIrp***     | dequeues an IRP that matches context specified by the driver |
| ***IoCsqRemoveNextIrp*** | dequeues the next IRP from the queue                         |

those are the routines you need to call to do the framework job, from adding an Irp to the queue, and removing it. so now you may ask, what's the purpose of the routines I mention before? and here's comes the idea of the framework.

when you implements the framework routines (CsqInsertIrp, CsqPeekNextIrp ..etc) you now have the body of the framework. now the framework job is to connect those implemented routines together when you call one of the macros (IoCsqInsertIrp, IoCsqInsertIrpEx ...etc). Every macro has a set of routines that it calls to do its job. 

for example, if you called ***IoCsqInsertIrpEx***, this function will do so :

[![3](/assets/images/kernelDriver/InvertedCall/3.png)](/assets/images/kernelDriver/InvertedCall/3.png)

the *xxxCsqAcquireLock* means the routine you have implemented that was called ***CsqAcquireLock*** (you can name it whatever you want when you implements it, the point that you just need to put it in the right parameter when you pass it to *IoCsqInitializeEx* function to initialize the IO_CSQ structure). And also the rest of the macros has a similar way to do its work, this [Documentation](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/cancel-safe-irp-queues) explains everything.

After you queued it, the request maker (user-mode app) should know about what you have done; so you should change the status of that queued IRP to be in the pending state until you dequeue it in the way you like. this Image also explains the steps for our case:

[![4.5](/assets/images/kernelDriver/InvertedCall/4.5.jpg)](/assets/images/kernelDriver/InvertedCall/4.5.jpg)



## Cases to Use the Framework

Now you got the basics idea for the framework. There are ways that make the use of the cancel-safe queue useful, one of it is to serializing the I/O processing in kernel thread. Like this for example :

[![5](/assets/images/kernelDriver/InvertedCall/5.png)](/assets/images/kernelDriver/InvertedCall/5.png)

Here when the driver accepts the IRP, it queues it. and when the I/O thread is free, it will check if there's any IRP in the queue; if it found one it will dequeue and complete it.

 Another way is to use it to put that IRP in the queue until some event happens then you complete it, and that's the way we need it for our project here; we need it to implement the ***Inverted Call Model***



# Inverted Call Model

> "Listening to your kernel is not less important than talking to it" —Omar Shehata



## Introduction

When I wrote my first kernel driver; [Thread Priority Driver](https://github.com/OmarShehata11/threadPrority), the general structure was simple. some ring 3 app wants to do some operation so it asks the driver to do it and it do. So the ring 3 app is always the speaker, and ring 0 driver the listener. But what if I want the opposite case?. I faced this problem when I was writing my driver; [Sound Driver](https://github.com/OmarShehata11/MySoundDriver), I searched for a solutions and I found one, guess what was the solution ?  YES you are right, it's the ***INVERTED CALL MODEL*** :).



## General Idea

The kernel drivers are made to be a servant for ring 3 apps, they always listen for any request comes from above and as long as it's valid it will do the work. But reversing the case? It's actually not possible. BUT there's a trick to do so, all the kernel drivers writers use it, even microsoft.

The Idea of it is to make the ring 3 app (I will use this term always :ring 3, I like it more) sends too many IRPs, and the driver will Queue it and wait. And when the driver wants to talk to the ring 3 app to notify it for example if there's an event happened and so, it will just dequeue one IRP and complete it. 

But how the ring 3 app will know when the driver completes its request ? that done using the ***IOCP*** (I/O completion port). this topic has a lot to say about but this articles aims only for inverted call model and cancel-safe framework. I may explain it in another article. For farther technical details, you can read this perfect [article](https://www.osronline.com/article.cfm%5Eid=94.htm) from OSR. and also [this](https://www.osr.com/nt-insider/2013-issue1/inverted-call-model-kmdf/) but this is for KMDF not WDM (my example driver is a WDM).

now let's go..



# Implementing the Concepts to the Example Driver: *MySoundDriver*

you can see my driver project code [here](https://github.com/OmarShehata11/MySoundDriver), the purpose of the project are explained there in the README file.

## Applying the Cancel-safe Framework

First of all I defined the structure of my routines that I will implement 

```cpp
//
// functions for the cancel-safe Irp framework
//

IO_CSQ_INSERT_IRP_EX CsqInsertIrp;
IO_CSQ_REMOVE_IRP CsqRemoveIrp;
IO_CSQ_PEEK_NEXT_IRP CsqPeekNextIrp;
IO_CSQ_ACQUIRE_LOCK CsqAcquireLock;
IO_CSQ_RELEASE_LOCK CsqReleaseLock; 
IO_CSQ_COMPLETE_CANCELED_IRP CsqCompleteCanceledIrp;
```

I won't actually use all of those routines, some of it will just not be mandatory for my project. like the CsqAcquireLock and CsqReleaseLock, those are used to just serialize the access to the queue, this case will be helpful if there are many threads trying to access the queue at the same time, but for my driver it's only on thread. and also CsqCompleteCanceledIrp I won't use it for the project, I have to implement it in some way but I don't need to make it do its job.

then I initialized the structure of **IO_CSQ** and the LIST_ENTRY:

```cpp	
	// Initialzing the Queue and the Cancel-safe Framework functions.
	InitializeListHead(&lpDeviceExtension->IrpQueueList);
	IoCsqInitializeEx(&lpDeviceExtension->CancelSafeQueue, CsqInsertIrp, CsqRemoveIrp, CsqPeekNextIrp, CsqAcquireLock, CsqReleaseLock, CsqCompleteCanceledIrp);
```

Now we need to Implement those routines.. 

### CsqInsertIrp

here's the CsqInsertIrp : 

```cpp
	// Get the start of the DevExtension structure
	PDEVICE_EXTENSION lpDeviceExtension = CONTAINING_RECORD(Csq, DEVICE_EXTENSION, CancelSafeQueue);
	// 
	// this fucntion just insert the Irp to the doubly-linked list.
	// we will add to queue using : InsertTailList  
	//

	// just to avoid the W4 error: unreferenced parameter
	UNREFERENCED_PARAMETER(InsertContext);


	// Insert that Irp to the tail, to act as a queue
	InsertTailList(&lpDeviceExtension->IrpQueueList, &Irp->Tail.Overlay.ListEntry);

	if (IsListEmpty(&lpDeviceExtension->IrpQueueList))
	{
		KdPrint(("FAIL: ADDING THE IRP TO THE QUEUE, list is empty in function %s", __FUNCTION__));
		return STATUS_UNSUCCESSFUL;
	}

	KdPrint(("SUCCESS: ADDING THE IRP TO THE QUEUE, in function %s", __FUNCTION__));
```

The general Idea of it is to insert the IRP to the queue, and according to the parameters passed to the routines I used ```CONTAINING_RECORD``` macro to get the start of my *DeviceExtension* structure then used that info to push the IRP to the queue

> - You can see the Implementation of the *DeviceExtension* structure in the header file; this structure is very important for every kernel driver to hold it's data and info about the driver in general case.
> - As said before you can implement the structure of the queue as you want, I implemented it as a doubly linked list (LIST_ENTRY). This [Article](https://www.osronline.com/article.cfm%5Earticle=499.htm) from again OSR is perfectly explaining the LIST_ENTRY structure.



### CsqRemoveIrp

Then the Implementation of the CsqRemoveIrp, and it's much simple as it is, it just removes the passed Irp to it:

```cpp
	RemoveEntryList(&Irp->Tail.Overlay.ListEntry);
```

you can check inside it if the queue is already empty or not, but I didn't have to; in the previous routines like CsqPeekNextIrp it will call the CsqRemoveIrp, so I checked it already in the CsqPeekNextIrp.



### CsqPeekNextIrp

And here comes the boss of the routines (in my opinion), This is its implementation:

```cpp
	PDEVICE_EXTENSION lpDeviceExtension = CONTAINING_RECORD(Csq, DEVICE_EXTENSION, CancelSafeQueue);
	//
	// The idea here is to just get the first Irp I meet (only for my case).
	// So I won't use the PeekContext
	// 

	UNREFERENCED_PARAMETER(PeekContext);
	UNREFERENCED_PARAMETER(Irp);


	// chech first if the list is empty
	if (IsListEmpty(&lpDeviceExtension->IrpQueueList))
	{
		KdPrint(("ERROR: there's no pending Irps to be removed from the list, from function %s", __FUNCTION__));
		return NULL;
	}
	
	//
	// Now we know that there are IRPs in the list. Now we got two choices.
	// First if the framework passed the Irp as NULL. then we should get the FLINK 
	// to the LIST HEAD, and if it specified a valid Irp, we will get the FLINK to that
	// passed Irp
	//
	PLIST_ENTRY ListEntryIrp;
	
	if (Irp != NULL)
		ListEntryIrp = Irp->Tail.Overlay.ListEntry.Flink;
	else
		ListEntryIrp = lpDeviceExtension->IrpQueueList.Flink;
 
	//
	// Now we get the List entry for the wanted Irp. Let's first check if it's not 
	// equals to the List Head itself 
	//
	if (ListEntryIrp != &lpDeviceExtension->IrpQueueList)
	{
		// Get the start of the irp
		PIRP pIrp = CONTAINING_RECORD(ListEntryIrp, IRP, Tail.Overlay.ListEntry); 
		
		// There's no PeekContext Passed
		if (!PeekContext)
		{
			KdPrint(("PRAVO: We Found the Irp to Be dequeued!"));
			return pIrp;
		}
		
		else
			KdPrint(("WARNING: You specified a peekContext Value !!"));
	
	}
	//
	// The returned value from the above function is the ListEntry member from the Irp structure
	// Irp->Tail.Overlay.ListEntry. So now we need to get the start of the Irp Structure Itself using 
	// CONTAINING_RECORD() macro.
	// You can actually get the Irp with another way; by before removing the Irp, get the 
	// Head List then get the value of FLINK of it (pointer to the removed Irp)
	// then use the macro to get the actuall point of the star of the Irp
	//
	return NULL;
```

The basic idea of this routine is to return the next IRP that it found it suitable, and what's meant by "suitable" that it matches the peekcontext it searches for; that given by the caller. But if there's no context that it's searches for (like in my case), it will just return the next IRP it founds in the queue. So I got the *DeviceExtension* structure first, then I checked if the list is empty or not, and if not then I have two choices

- there's an IRP passed to me at first, then I will get the forward pointer of the LIST_ENTRY structure of the that IRP (that points to the next IRP) 
- and if there's IRP passed, then I choose the IRP just after the LIST_ENTRY head

then get the start of that IRP (using the CONTAINING_RECORD) and return it.

> you can use the peekcontext to differentiate between different IRPs so different events and so on.



### CsqAcquireLock & CsqReleaseLock & CsqCompleteCanceledIrp

And again I didn't used them in my project, you can implement the release and acquire lock using any kind of object that can do so like a mutex.



## Applying the Inverted Call Model concepts

The general Idea of the example project that I registered for PnP Notification. and when there's a notification came from the PnP manager; the driver needs to notify the user-mode app. 



### Dequeue the IRP

So I created this callback routine ```UsbDriverCallBackRoutine``` that will be called if there's an event related to what I just registered with the PnP manager. So inside this function we should check if there's any IRP in the Queue, and if found it will dequeue one and complete it. there's a snippet of the function code:

```cpp
	if (IsListEmpty(&lpDeviceExtension->IrpQueueList))
	{
		KdPrint(("The Queue is empty, Can't dequeue any."));
		return STATUS_UNSUCCESSFUL;
	}

	PIRP Irp = IoCsqRemoveNextIrp(&lpDeviceExtension->CancelSafeQueue, nullptr);

	if (Irp == NULL)
	{ 
		KdPrint(("Error: Can't get the Irp from the Queue. Queue seems to be empty")); 
		return STATUS_UNSUCCESSFUL;
	}

	KdPrint(("SUCCESS: the Irp now pulled from the queue."));
```

So after checking  if the event matches what we are looking for, I checked if the list is empty, and if not I will call ```IoCsqRemoveNextIrp``` to remove the next IRP without any PeekContext passed. Now after I popped it out from the queue, I need to complete it :

```cpp
	Irp->IoStatus.Status = STATUS_SUCCESS;
	Irp->IoStatus.Information = 0;

	IoCompleteRequest(Irp, IO_NO_INCREMENT);

	return STATUS_SUCCESS;
```



### Push the IRP to the Queue

When we need to Push an IRP to the Queue ? when we firstly accept it right. So when the ring 3 app call the ```DeviceIoControl()``` with our IOCTL code, the I/O manager will send to us an IRP. So inside the Dispatch routine that will accept that request ( You should set this function here ```DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL]```) we should push the incoming IRP to the queue and change its status to pending state, like so :

```cpp
	switch (StackLoc->Parameters.DeviceIoControl.IoControlCode)
	{
	case IOCTL_MY_SOUND:
		KdPrint(("Catched a new incoming IRP. Queuing it."));
		KdPrint(("is the list is empty: %d", IsListEmpty(&lpDeviceExtension->IrpQueueList)));

		// push the Irp to the queue
		status = IoCsqInsertIrpEx(&lpDeviceExtension->CancelSafeQueue, Irp, nullptr, nullptr);

		if (!NT_SUCCESS(status))
		{
			KdPrint(("Error: can't push the Irp to the queue."));
			break;
		}
	
		// Everything is well..
		KdPrint(("SUCCESS: the Irp now pushed to the queue."));


		// Increment the number of Irps.
		lpDeviceExtension->nuOfQueuedIrps++;


		KdPrint((" Number of Irps in the queue : %i.", lpDeviceExtension->nuOfQueuedIrps));
		KdPrint(("is the list is empty: %d", IsListEmpty(&lpDeviceExtension->IrpQueueList)));

		status = STATUS_SUCCESS;

		break;
	}

	Irp->IoStatus.Status = STATUS_PENDING;

	status = STATUS_PENDING;

	return status;
```

So I checked the incoming IOCTL code, and if it matches our; then I called ```IoCsqInsertIrpEx``` to push it to the queue, and finally in the last part I changed its status to ```STATUS_PENDING``` and returned that status.



# Conclusion

You can talk to ring 3 apps from kernel driver to notify it about an event or anything from kernel prospective using the Inverted Call Model, and its idea is to Park or queue an IRP from the user and when you want to notify the user you will complete one of queued IRPs. And to do your job on the queue things you need to use the Cancel-safe framework to make it more manageable for you.



# Note Resources

In this section I want to suggest some good resource to get start learning about kernel mode drivers which is really so much fun topic to learn, and it will help you to understand what's behind the scenes for those guys who love deep understand and knowledge

- ***Windows internals 1 & 2 & 3 for pavel in pluralsight*** : I can say that's it's one of the best courses I have ever studied. So much knowledge was very talented and professional instructor.
- ***Programming the Microsoft Windows Driver Model 2nd edition for Walter Oney***: this book is a big reference for you to get start getting deep on WDM driver model programming, it has a lot of knowledge from a senior kernel driver developer and it helped me a lot in this project.
- ***OSR online***: this website has a lot of articles that interested only in windows driver development. and for both models KMDF and WDM.
- and of course we can't forget ***Windows Documentation***: we all don't like reading long boring documentation, but this one is different, really I learned a lot from it and it's explained in a very good way that it will make you obsessed with it.

thanks for reading all of this boring stuff, and if you have any suggestion for me or a question or anything that may negatively affects my psyche, Please take the step and mail me :).

SEE YA.

