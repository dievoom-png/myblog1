---
title: Understanding Keyloggers
date: 2024-07-20T21:16:19+03:00
draft: true
slug: understanding-keyloggers
---

Lately i've been researching a lot about keyloggers and implementing them in C using system hooks and that led me into a rabbithole that i will share with you today, but before we start i should put a disclaimer that i will over simplify things so we would understand the things needed for our Windows keylogger not the whole keyboard input mechanism.

So now thats that out of the way, ever thought how keyboard handles a key input anyways?
because thats what we will be talking about starting with...

## Scan codes

Our journey starts with scan codes, you can think of a scan code as a unique identifier for the physical key on your keyboard set by the manufacturer however ever since USB happened its mostly been standardized.
Scan codes are also layout independent and hardware dependent, but what's that mean?

![AZQWERTY](AZQWERTY.jpg)

Whenever you press on a key on your keyboard, a scan code is sent to the OS.

 So lets say we pressed A in the QWERTY layout and we got the scan code `0x1E` for example. then we switched the layout to AZERTY in the OS settings and pressed the same key physically we will get the same code `0x1E` even though its not interpreted as the same character anymore but its the same physical key on the keyboard independent of the layout.

 Scan codes are rarely used by applications though as they're easier to deal with especially using the Win32 API. Now after pressing a key the scan code is generated then converted by windows to a virtual key code using the Keyboard Device Driver.

## VK code

Now Virtual key codes on the other hand can be looked as as the software as they're higher level and handled by the OS itself, in this case we are talking about windows. Vk-Codes are used by windows to identify keyboard keys Independent of the language selected.

Let's go back to our example so we can grasp the difference, pressing A in the QWERTY layout (en) will give us `VK_A` yet if we press the same key in AZERTY (fr) will give us `VK_Q` as a different **virtual key code** but the same **scan code**

![diagram1](keylogger1.png)
This is a simplified diagram to the keyboard input life cycle we covered so far. Now the next step is to get the VK-Codes before it reaches the application and we are going to cover that using hooks.

## Messages and Message Loop

So the diagram above is a lie. Well not all of it but when the keyboard device driver converts the scan code to Vk-Codes it's actually moved around windows as Messages and those messages contain


## Hooks

Hooks are a mechanism that allows us to intercept a certain event as it happens whether that is  for example a key-press or a mouse movement.



#  Todo

1. add HID
2. MSg loop
3. add video to show case scan code of press and release.
   1. The scancode for key release (`break') is obtained from it by setting the high order bit (adding 0x80 = 128). Thus, Esc press produces scancode 01, Esc release scancode 81 (hex).
4. RE the keylogger new vs old.
5. 
6
