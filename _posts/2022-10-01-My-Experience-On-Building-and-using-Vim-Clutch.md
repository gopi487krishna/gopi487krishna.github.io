---
layout: post
title: My Experience on Building and Using Vim Clutch
author: Gopi Krishna Menon
tags: Saturday-Posts
---

Few weeks back, I was scrolling through [r/Neovim](https://www.reddit.com/r/neovim/) on reddit and noticed a post that mentioned about `vim clutch`. I got intrigued by the idea of using
a foot pedal to switch modes in vim and decided to give this project a try.

So before starting, lets understand what vim clutch is and discuss the motivation behind building such a device.

## What is VIM Clutch?

vim clutch is a footpedal that allows you to go to to insert mode when the pedal is pressed and to normal mode when the pedal is released. 
It is primarily designed as  a means of convenience and to improve the text editing speed of users using vim/neovim.


The idea for this device was proposed by  [Aleksandr Levchuk](https://github.com/alevchuk/vim-clutch) 


![OriginalClutch](/assets/posts_img/2022-10-01-VImClutchExp/original-clutch.jpg)

To give you guys a comparison, you know clutches in cars right? They basically serve as the mechanism for changing gears in a car 
(They are used for connecting/disconnecting the engine from transmission). Similar to that, vim clutch is used for switching from
one mode to another in vim/neovim.

## Motivation behind VIM Clutch

- Pressing `ESC` causes your hand to move away from homerow resulting in slight discomfort.
- To improve your typing speed and reduce the strain on ring finger

## Why am I doing it?

- From the past few months, I have become a hardcore neovim user and have faced this problem of pressing Escape
  especially on a full sized keyboard
- Because its a fun project and would help me learn something new

## My Modifications

- Pressing the pedal all the time to remain in `Insert` mode is really a pain. So in my version
the pedal acts a button where pressing the pedal results in switching to normal mode from any mode
- In future I plan to use various combinations of presses to trigger different modes and possibly make my life easier.

Now that the you have understood the definition and motivation behind this project, lets get to the real stuff and start
building this device.

**Note** : The instructions given here were performed on a Gentoo. If you wish to do Arduino development on Gentoo head over to
[https://wiki.gentoo.org/wiki/Arduino](https://wiki.gentoo.org/wiki/Arduino) to complete the prerequisites.


## Components Required

- [Arduino Board](https://amzn.eu/d/hOcHe6V)
- [Electronic Foot Pedal](https://amzn.eu/d/3CVp30r)
- [Jumper wires](https://amzn.eu/d/3zbHn8s)

## Building the hardware

Its pretty simple, since our footpedal is acting as the switch,
- Connect the live wire from the footpedal into the 5V rail on your Arduino Board.
- Connect the ground wire from the footpedal into the ground pin of your Arduino Board.
- Connect the data wire from the footpedal into the 8th digital pin of your Arduino board.

## Flashing/Compiling the program to the Arduino
- Clone the repository from github using this command
```bash
git clone --recurse-submodules -j8  git@github.com:gopi487krishna/nvim-clutch.git
```
- `cd` into the newly cloned directory
- Open the file `nvim_clutch_sketch.ino` in Arduino IDE and upload it to the Arduino Board

### What the sketch does?

```cpp
const int footpedal_pin = 8; // Digital Pin 8 to Read the Input from Pedal
int footpedal_previous_state=0;

void setup(){
  pinMode(footpedal_pin, INPUT); // We need to recieve data from  footpedal
  Serial.begin(9600);
}

void loop(){
  int footpedal_current_state = digitalRead(footpedal_pin); // Check if pressed or released
  if(footpedal_current_state == 1 && footpedal_previous_state != footpedal_current_state){
    Serial.println(1);
    
  }   
  footpedal_previous_state = footpedal_current_state;
  delay(20);
}
```
- Well to give a brief explaination, this code sets the 8th Digital pin as Digital Input and prints 
`1` into the serial port if the button was pressed. 
- The if condition makes sure that `1` is only sent when the button was pressed. Holding the pedal
in pressed state will not send multiple `1` or `ON` state to the program.

## Program to send the keyboard event

### Some Important Points to Note
- Here I had a choice to either write a kernel driver or a userspace program to handle the communication.
I decided to go with a userspace program since its simple serial communication.

- To read the data from the serial port, I decided to use a library called [CppLinuxSerial](https://github.com/gbmhunter/CppLinuxSerial.git)
It hides all the complexity of performing serial communication and gives a simple interface to work with serial ports.

- To send the keyboard sequence, I use a library called `libxdo`. Its a simple library for simulating keyboard input and mouse activity.
And as the clever ones among you guessed it, yes it requires Xorg server to work. My usecase was within a X environment and therefore I decided
to go through this route. You can write a kernel driver to send keyboard sequence directly without requiring an Xorg server.

## Steps to compile and install the nclutch program
- Navigate into the github repo that you clone in the previous step.
- Create a new folder called build
- `cd` into that folder and enter
```bash
cmake -DBUILD_TESTS=OFF ..
```
- Now run make to build the program.
- Run make install to install the program into your system

<script id="asciicast-4BjBsjfJVasiHHm3znUR4pYQb" src="https://asciinema.org/a/4BjBsjfJVasiHHm3znUR4pYQb.js" async></script>
### Some Important Points to Note
- In order to understand what the program does, please take a look inside nclutch directory. The code is self explanatory in 
nature and quite easy to follow

```cpp
#include <CppLinuxSerial/SerialPort.hpp>
#include <iostream>
#include <xdo.h>

int main(int argc, char *argv[]) {

  std::string filename;

  if (argc > 1) {
    filename = std::string(argv[1]);
  } else {
    std::cerr << "Error : Please provide the path to the device\n";
    return EXIT_FAILURE;
  }

  /*Screen context*/
  xdo_t *x = xdo_new(":0.0");

  if (x == nullptr) {
    std::cerr << "Error : Failed to create new instance of xdo\n ";
    return EXIT_FAILURE;
  }

  mn::CppLinuxSerial::SerialPort serialport(
      filename, mn::CppLinuxSerial::BaudRate::B_9600,
      mn::CppLinuxSerial::NumDataBits::EIGHT, mn::CppLinuxSerial::Parity::NONE,
      mn::CppLinuxSerial::NumStopBits::ONE);

  /*Block on read and wait until we get atleast 1 byte*/
  serialport.SetTimeout(-1);

  /*Open the file for communication*/
  serialport.Open();

  std::string readData; // Dummy var for reading a byte

  while (true) {
    serialport.Read(readData);

    /*Send escape key to the focused window*/
    xdo_send_keysequence_window(x, CURRENTWINDOW, "Escape", 0);
  }
  serialport.Close();
  delete x;

  return EXIT_SUCCESS;
}
```
- When running `nclutch` you are required to pass the device path as the argument. Run 
```bash
sudo dmesg | grep cdc 
```
to get the device name

<script id="asciicast-NdbhdXjytSvxFgaruK3cjAazA" src="https://asciinema.org/a/NdbhdXjytSvxFgaruK3cjAazA.js" async></script>

## Post Installation
- After building nclutch program, you can create a simple service script to start the program automatically
after the system boots
- In my case, I spawn it as a daemon within my window manager script. Please refer to the documentation of your
window manager to understand how to spawn a daemon after startup

<script id="asciicast-RQRDdizZDok9glLLeFYxVPrUX" src="https://asciinema.org/a/RQRDdizZDok9glLLeFYxVPrUX.js" async></script>


## How is the experience so far?
![My Clutch](/assets/posts_img/2022-10-01-VImClutchExp/my_clutch.jpeg)
- I use Vim Clutch with my desktop and tbh its an improved experience.
- I have not become the fastest typist in town but the strain on my ring finger
has certainly reduced


## Special Thanks !

A special thanks to my friend [Nikhil Kumar](https://github.com/NikhilKumar1999) for helping me complete this 
project. This was my very first attempt at working with hardware stuff
and his excellent knowledge in embedded electronics helped me get started
easily. Also his deep insights on how this project can be improved further
(such as using low cost board, introducing new combinations) is really
interesting and I am planning to implement them further down the road.

