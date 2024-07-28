 ---
title: Understanding keyloggers
date: 2024-07-20t21:16:19+03:00
draft: true
slug: understanding-keyloggers
---

Lately i've been researching a lot about keyloggers and implementing them in c using system hooks and that led me into a rabbithole that i will share with you today, but before we start i should put a disclaimer that i will over simplify things so we would understand the things needed for our windows keylogger not the whole keyboard input mechanism.

So now thats that out of the way, ever thought how keyboard handles a key input anyways?
because thats what we will be talking about starting with...

## scan codes

our journey starts with scan codes, you can think of a scan code as a unique identifier for the physical key on your keyboard set by the manufacturer however ever since usb happened its mostly been standardized.
scan codes are also layout independent and hardware dependent, but what's that mean?

![azqwerty](azqwerty.jpg)

Whenever you press on a key on your keyboard, a scan code is sent to the os.

 So lets say we pressed a in the qwerty layout and we got the scan code `0x1e` for example. then we switched the layout to azerty in the os settings and pressed the same key physically we will get the same code `0x1e` even though its not interpreted as the same character anymore but its the same physical key on the keyboard independent of the layout.

 Scan codes are rarely used by applications though as they're easier to deal with especially using the win32 api. now after pressing a key the scan code is generated then converted by windows to a virtual key code using the keyboard device driver.

## vk code

Now virtual key codes on the other hand can be looked as as the software as they're higher level and handled by the os itself, in this case we are talking about windows. vk-codes are used by windows to identify keyboard keys independent of the language selected.

Let's go back to our example so we can grasp the difference, pressing a in the qwerty layout (en) will give us `vk_a` yet if we press the same key in azerty (fr) will give us `vk_q` as a different **virtual key code** but the same **scan code**

![diagram1](keylogger1.png)
this is a simplified diagram to the keyboard input life cycle we covered so far. now the next step is to get the vk-codes before it reaches the application and we are going to cover that using hooks.

## messages

As we are going to mostly record inputs that are sent to a running application and most likely it will have a window for a gui then we need to delve more in-depth and understand how those window-based application can receive and process the user keystrokes, so in another words the diagram above was a lie.

Well not all of it but when the keyboard device driver converts the scan code to vk-codes it's actually moved around windows as `messages`. messages are just already defined constants that represent that type of event that just occurred for example `#define wm_char 0x0102`. so for example we will see later on that we can check to see if the user pressed a key by checking if the `wm_char` message is posed to the application's thread message queue.

 Yet messages are not just limited to keystrokes they can be anything from clicks to touch screen-gestures and other user input methods. messages can be also generated from the os for example notifying discord that there is a new mic that just got plugged in or windows will be hibernating or sleeping.

Now that we understand how windows passed the input to running applications we can see the real diagram that i'm refrencing from the [msdn](https://learn.microsoft.com/en-us/windows/win32/inputdev/about-keyboard-input)

![keyboardinput](keyboardinput.png)

Let's start from the begnning:

0. You have discord open and you're typing out a message.
1. A key is pressed on the keyboard.
2. The scan code of the key is sent to the os.
3. Keyboard device driver translates/maps that scan code to a vk-code that is found in the message.
4. The message is posted to the system message queue before the os can send the message to discord's thead queue.
5. The thread message loop is what fetches new message posted to discords thread queue and passes it to windows procedure.
    
    ```C
    MSG msg;
    while(GetMessage(&msg,0,0,0)){ // While there are messages in the queue.

        TranslateMessage(&msg); // If key pressed is a char -> Add WM_CHAR message to the queueu.
        DispatchMessage(&msg); // Send the message to the discord's window procedure to handle it.
    }

    ```

6. The windows procedure is what handles the message sent to discord. So lets say discord's WndProc looks something like this:

    ```C
    // you don't have to define the WM_CHAR 0x0100 macro, it's already defined for you
    switch(msg){
   //
   //
   //
   case WM_CHAR: // or we can use the value 0x0102 directly ¯\_(ツ)_/¯
      //stuff
      break;
    }
    ```



Now that we have a solid idea what exactly we want to intercept, we will focus on how to intercept the message that is going to the applications and running our own window procedure function to log that input.

## Hooks

Hooks are a mechanism that allows us to intercept a certain event as it happens whether that is  for example a key-press or a mouse movement and once it is intercepted we can run our own window procedure and in this case it's called a `hook procedure`.


# Todo
2. MSg loop
3. add video to show case scan code of press and release.
4. RE the keylogger new vs old.
5. Add cover (see dll injection)
6. note that the hook will be on another thread so there will be at least 2 threads in the malware if it wants to communicate with the C2.
7. write bullet points of topics to talk about first so you can get stuff done quicker and have a chain of thought!!

   4. Hooking the user input and how it fits in general idea.
   5. Cowsay
   6. Ppicetify
   
