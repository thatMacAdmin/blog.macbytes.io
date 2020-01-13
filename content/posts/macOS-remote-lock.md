---
title: "macOS MDM Remote Lock - Useful for theft?"
date: 2020-01-07T22:27:41-05:00
draft: true
toc: false
images:
tags:
  - macOS
  - MDM
---

I was recently in an interview for an Apple Administrator job, and we were talking about APNS. Really, we were talking about why APNS was valuable. I had somewhat glibly stated that one of the great things about APNS and MDM was that if a machine was lost or stolen you could just lock it and help to keep the data on the machine secure. It was then that the interviewer said something to the effect of "Well is that really true? If the machine is secure with FileVault 2 would they really connect it to he internet to get the MDM command?" This being an interview I pivoted and said something like, well its always nice to have another layer of security outside of FileVault. And of course that is true, but it got me thinking, is Apple's Remote MDM lock really valuable if a FileVault 2 encrypted machine gets stolen?

There are a few things to unpack here, and we have to ask ourselves what is the goal of locking a computer which is stolen? If your goal is just to protect your companies data, FileVault 2 does a great job of that already. Locking the machine with a 6 digit code is probably not going to significantly enhance the security of that data. FileVault uses a solid AES-XTS 256 bit cipher with a 128 bit key. Sure, if a nation state like Russia is after your data they could probably crack this eventually, but for most folks, this level of encyption is suficiently bullet proof.

Of course if I stole a Mac, I probably would care less about the data and more about making it my own. Then I could have a cool new mac to use! So the big question still remains, if a jamf Pro admin locks a Mac with FileVault 2, which is off or not connected to the intenet at the time of the lock, will it lock?

Off to the lab for some tests!

I took a machine which was connected to my dev jamf Pro environment, and which had a dead battery from sitting in a drawer. I sent a remote lock command from jamf Pro and booted it to the FileVault unlock screen with no internet connection just to make sure all was in order. Of course there was no lock. Acting like a user I tried to login with the wrong password a few times with no luck. 

Next, I booted to recovery and connected to a Wi-Fi network. Still no magical lock. So I connected the machine to ethernet and rebooted to the FileVault login screen. Still no lock. Alright, I thought thats a little frustrating but I know that when a machine is restored it will check in with Apple in order to validate its macOS license status among other things. So I rebooted into recovery connected to a network, erased the disk and started a restore. Still no lock!

I have to confess that I am a bit sad. Apple should know which machine was tied to the GUID that the APNS lock command was sent to but doesnt seem to be doing anything with that information if the machine does not check in via the normal way. Machines can, and almost certianly do checking with their GUID upon requesting 

