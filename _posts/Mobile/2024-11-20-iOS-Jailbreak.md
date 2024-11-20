---
title: "iOS Jailbreak"
date: 2024-11-20 10:00:00
classes: wide
header:
  teaser: /assets/images/ios-jailbreack/logo.png
  overlay_image: /assets/images/ios-jailbreak/logo.png
  overlay_filter: 0.5
ribbon: DarkSlateGray
excerpt: "Unlocking the Full Potential of Your Apple Device"
description: "Unlocking the Full Potential of Your Apple Device"
categories:
  - Forensics
  - Mobile
tags:
  - iOS
  - Jailbreak
  - Checkm8
  - Exploit
toc: true
toc_sticky: true
toc_label: "On This Blog"
toc_icon: "biohazard"
---
# Introduction

It's been 12 years since the fist time I jailbroken an iOS device. That fist time was with the first iPhone I ever owned-a device that brings back a rollercoaster of emotions and memories, which I'll be sharing with you as we go along.

It was an iPhone 4 CDMA from Verizon, and I was scammed when I bought it (by the way). The folk who sold it to me forgot mentioned it was a phone with a theft report in the U.S. Not only that, but he also claimed it was an iPhone 4S. However, since the CDMA version of the iPhone 4 was identical to the iPhone 4s, and I had zero experience buying secon-hand iPhones at the time (plus I was only 15 years old), I traded my Motorola Moto Razr (the ultra-thin model with kevlar finish on the back) for this device. My intention was to play with the incredible, revolutionary first version of Siri. However, the iPhone 4 was not compatible with Siri.

To use this iPhone, it had to be unlocked from the carrier, and an RSIM wasn't an option due to the lack of a SIM tray... Thanks, CDMA.

While researching, I discovered that by jailbreacking the phone, you could unlock it from the carrier, regardless of the theft report, and even install Siri as a tweak from Cydia. And that's exaclty how I had my fisrt encounter with iOS security and mobile application security.

So, thank you to that guy who tricked a 15-years-old teenager... because of you, I'm here today, writing this blog with the propose of explaining, on a technical level, the process of jailbreacking an iPhone. However, my intention is not to limit this to just using tools like a Checkra1n or Palera1n. The goal of this blog is to give you an insight into the complex and extensive process these tools automate to achive the caining of vulnerabilities and exploits that ultimately result what we could call iOS privilege escalation.


iOS jailbreaking involves exploiting iPhone security flaws in Apple’s mobile operating system to remove restrictions and gain full access to the system. This elevated access allows users to install apps and tweaks not available through Apple’s App Store, customize the interface, and modify system settings. While iPhone customization through jailbreaking offers increased freedom, it also introduces jailbreak vulnerabilities, voids warranties, and can lead to system instability if not properly managed.

In this blog, we will explore the history of iOS jailbreaking, different types of jailbreaks, essential jailbreaking tools, notable iPhone exploits, and key resources for those interested in the evolving iOS hacking culture. This deep dive will help you understand how iOS jailbreaking has evolved over time and why it remains a crucial aspect of the jailbreak community.

# History 

The history of iOS jailbreaking is as fascinating as the technology itself, characterized by an ongoing "cat and mouse" game between hackers and Apple. Each time Apple patches a security flaw, the jailbreak community finds a new way to break the system, continuing the cycle.

- **2007: The Birth of Jailbreaking**  
   The first iPhone was released without an App Store, limiting its functionality to the apps provided by Apple. Hackers were quick to find ways to open the system, and within months, the first jailbreak was developed by a hacker named "Geohot" (George Hotz). This exploit allowed users to bypass Apple's restrictions and install third-party apps using the Installer.app, the precursor to Cydia.
  
- **2008-2010: The Rise of Cydia**  
   With the introduction of the App Store in 2008, Apple allowed users to officially download third-party apps, but many apps were still subject to strict guidelines. The creation of **Cydia**, developed by Jay Freeman (Saurik), became the go-to platform for jailbroken devices, providing access to apps and tweaks that could bypass Apple’s restrictions. During this period, tools like **redsn0w** and **greenpois0n** became popular for jailbreaking early iOS versions.

- **2010-2015: Jailbreaking Goes Mainstream**  
   Jailbreaking reached its peak popularity between 2010 and 2015, fueled by tools such as **evasi0n** and **Pangu**, which allowed users to jailbreak iOS devices with ease. At this point, jailbreaking was widely accepted, and even legally permitted in certain jurisdictions like the United States under DMCA exemptions.

- **2019: Checkm8 Revolution**  
   The discovery of the **checkm8** bootrom exploit in 2019 was a turning point in the jailbreaking community. This exploit, which affects iPhones from the 4S to the X, provided an unpatchable vulnerability that Apple couldn’t fully address with software updates, allowing for persistent jailbreaking on affected devices.
   
- **Current State of Jailbreaking**  
   While jailbreaking has become more challenging due to Apple’s improved security measures, tools like **checkra1n** and **unc0ver** continue to offer jailbreak solutions for modern iOS versions, though the community has shrunk considerably as the risks and effort outweigh the benefits for many users.


# Types of Jailbreaking

There are several types of iOS jailbreaks, each offering different levels of persistence and functionality:

- **Untethered Jailbreak**: This is the most coveted type of jailbreak, as it allows the device to remain jailbroken even after a reboot. No additional actions are required to maintain the jailbreak, making it the most convenient for users. Examples include **evasi0n** and **greenpois0n**.

- **Tethered Jailbreak**: In a tethered jailbreak, the device must be connected (tethered) to a computer every time it is rebooted in order to reapply the jailbreak. Without doing this, the device will not boot into a usable state, making this type less popular due to its inconvenience.

- **Semi-Tethered Jailbreak**: Similar to a tethered jailbreak, but in this case, the device can reboot without being tethered to a computer. However, you’ll lose jailbreak functionality until you reapply it via a computer.

- **Semi-Untethered Jailbreak**: In a semi-untethered jailbreak, the device can reboot and still function normally without a computer. However, the jailbreak itself becomes inactive, and an app installed on the device must be used to re-jailbreak the device after a reboot. Examples include **unc0ver** and **Chimera**.

# Jailbreaking Tools

Over the years, several tools have been developed to make jailbreaking easier and accessible to the general public. Some of the most popular include:

- **checkra1n**: Based on the checkm8 bootrom exploit, checkra1n offers an unpatchable jailbreak for iPhones from the iPhone 4S to the iPhone X. This tool is compatible with iOS 12 and later, making it one of the most reliable options for older devices.
  
- **unc0ver**: A popular tool that offers a semi-untethered jailbreak for devices running iOS 11 to iOS 14.3. It’s known for its stability and ease of use, as it provides a one-click jailbreak solution through an app that can be installed on the device.

- **Taurine**: Released by the Odyssey Team, Taurine is designed for iOS 14 and offers fast and stable performance. It uses the **libhooker** tweak injection system, which is optimized for modern iOS devices.

- **Chimera**: Another tool from the Odyssey Team, Chimera was released for iOS 12. It’s notable for using the **Sileo** package manager instead of the traditional Cydia, offering a modern, user-friendly experience.

# Notable Exploits

Jailbreaking is made possible by exploiting vulnerabilities in iOS. Here are some of the most significant exploits used in jailbreaking tools:

- **checkm8**: A bootrom exploit that affects devices from iPhone 4S to iPhone X. Since it is a hardware-level exploit, it is unpatchable via software updates, making it a powerful tool for creating persistent jailbreaks. It is the basis for the **checkra1n** jailbreak.

- **tfp0**: An exploit that grants task-for-pid (tfp0) privileges, giving the attacker full read/write access to the kernel memory. This is a common type of exploit used in modern jailbreaks like **unc0ver** and **Taurine**.

- **v0rtex**: A kernel-level exploit that was used for jailbreaking iOS 10.3.x. It is known for allowing advanced memory manipulation and privilege escalation.

- **SockPuppet**: This exploit is used in jailbreaks for iOS 12. It is a userland exploit that escalates privileges and allows for modifications at the system level.

# Resources & Reference

- **The iPhone Wiki**: A detailed resource on the history, tools, and exploits used in iOS jailbreaking. It is an invaluable source for understanding the technical underpinnings of jailbreaking.
   - [The iPhone Wiki](https://www.theiphonewiki.com/wiki/Main_Page)

- **Reddit's r/jailbreak**: A vibrant community of jailbreak enthusiasts who share news, guides, and troubleshooting tips.
   - [r/jailbreak](https://www.reddit.com/r/jailbreak/)

- **Cydia Substrate**: A framework for developing tweaks and extensions for jailbroken iOS devices.
   - [Cydia Substrate](http://www.cydiasubstrate.com/)

- **Google Project Zero Blog**: Google’s Project Zero often publishes detailed reports on iOS vulnerabilities, offering insights into how these exploits work and are used in jailbreaking.
   - [Google Project Zero](https://googleprojectzero.blogspot.com/)