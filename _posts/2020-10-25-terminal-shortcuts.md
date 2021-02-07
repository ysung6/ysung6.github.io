---
title:  "Terminal shortcuts"
last_modified_at: 
categories: 
  - Bash
tags:
  - linux, os
toc: true
toc_label: "Getting Started"
---

Sending signals is often used when we try to kill processes through the terminal. you can use the kill command, but you can also use shortcuts that are not commands that are typed and entered.

## Control + D(^D) vs Control + C(^C)

Control + C is often used to stop a running process and Control + D can also be used to kill programs like the python shell. What is the difference?

### Control + C

Control + C sends the Interrupt Signal (SIGINT). 
You may notice that python shells do not exit when receiving this signal. It is programmed to repond to it, so it returns a KeyboardInterrupt instead of quitting the shell. However, when it interrupts other programs like a locally running Django Server, the program is interrupted and stopped

### Control + D

Control + D sends the EOF Signal. This signals that there will be no more input as it is really the "end". So most programs that expect an input, like the python shell or even terminal, would close because they have received a signal that there will be no more inputs. On the otherhand our Django server will simply ignore this, because it is not waiting for such inputs but instead http requests.

## Other signals

### Control + Z 

Control + Z sends a running task to the background.
If you have a long running process, you can send this to the background, and then type other commands. This is handy when used with bash commands like jobs, fg, and bg

### Control + R 
 
Control + R enables you to search through past commands that you have used.

### Control + L

same as the clear command in linux.
