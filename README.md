I needed to contol a bunch of motors and stuff for a project, and Grbl / G code seemed like a good way to do it. However, grbl doesn't have everything I want, so I decided to add it myself. This is for the old standby of an Arduino Mega and RAMPS 1.4 board.

One of those is servos. Grbl is really good at controlling stepper motors, but I'd like to also add on some regualr hobby servos for not as presise stuff. First, I need to generate the PWM signals that the servos expect. One way to do this is by using hardware PWM, and the hardware timers, on their respective pins. But, grbl already uses some hardware timers, and some of the special pins are already hardwired to other stuff on the RAMPS board. I decided to go the other way and do something like (I think) the Arduino Servo library does. A table of pulse timings for each servo channel is used to set a hardware timer (TIMER5 which is unused in Grbl on the Mega), the timer triggers an interrupt, and the timer interrupt ISR is used to change a regular pin and set up the timer again using the next entry in the table. The longest pulse my servos respond to is about 2500us, so for a regular period of 20ms that the servo expects its signal, that gives me enough room to run up to 8 servos. 

Ideally, these would be added as extra machine axes so they could be used along the steppers inside a regular gcode move, but I couldn't be bothered to figure out how to hack grbl for yet more axes, so I implemented some extra M codes instead. Within the codes I added, I reused some of the letters, but these don't affect the matching letters for the stepper axes:

* M96 - sets the servo position immediately when issued. Works like G0.
  * Usage: **M95 An Bn Cn Dn Wn Xn Yn Zn** 
    * where A B C D E W X Y Z refer to the corresponding servo (do not include any servo you don't want to change), and n is the servo position (from 0 to 499 - actual angle this refers to on your servo depends on the type of servo)  
* M97 - Starts servo moving smoothly from current position to new position. Works like G1. Non-blocking.
  * Usage: **M95 An Bn Cn Dn Wn Xn Yn Zn Ts** 
    * where A B C D E W X Y Z refer to the corresponding servo, n is the servo position, and T is the time of the move, in seconds. This timing uses the time base of the servo signal period, and is therefore completely isolated from the rest of Grbl's stepper control functions. It more or less does the servo move in the background.
* M98 - works exactly the same as M97, but instead of starting when the command is issued, it starts with the next G command issued. This makes it possible to (roughly) cordinate motion with a move in the stepper axes (but you need to figure out the correct timings yourself).

***

![GitHub Logo](https://github.com/fra589/grbl-Mega-5X/blob/edge/doc/images/Mega-5X-logo.svg)

***

Grbl is a no-compromise, high performance, low cost alternative to parallel-port-based motion control for CNC milling. This version of Grbl runs on an Arduino Mega2560 only.

The controller is written in highly optimized C utilizing every clever feature of the AVR-chips to achieve precise timing and asynchronous operation. It is able to maintain up to 30kHz of stable, jitter free control pulses.

It accepts standards-compliant g-code and has been tested with the output of several CAM tools with no problems. Arcs, circles and helical motion are fully supported, as well as, all other primary g-code commands. Macro functions, variables, and most canned cycles are not supported, but we think GUIs can do a much better job at translating them into straight g-code anyhow.

Grbl includes full acceleration management with look ahead. That means the controller will look up to 24 motions into the future and plan its velocities ahead to deliver smooth acceleration and jerk-free cornering.

* [Licensing](https://github.com/fra589/grbl-Mega-5X/blob/edge/COPYING): Grbl is free software, released under the GPLv3 license.

* For more information and help, check out our **[Wiki pages!](https://github.com/gnea/grbl/wiki)** If you find that the information is out-dated, please to help us keep it updated by editing it or notifying our community! Thanks!

* Lead Developer: Gauthier Brière (France) aka @fra589

* Built on the wonderful Grbl v1.1f (2017) firmware originally written by Simen Svale Skogsrud (Norway) and maintained by Sungeun "Sonny" Jeon, Ph.D. (USA) aka @chamnit

***

##Update Summary for v1.1
- **IMPORTANT:** Your EEPROM will be wiped and restored with new settings. This is due to the addition of two new spindle speed '$' settings.

- **Real-time Overrides** : Alters the machine running state immediately with feed, rapid, spindle speed, spindle stop, and coolant toggle controls. This awesome new feature is common only on industrial machines, often used to optimize speeds and feeds while a job is running. Most hobby CNC's try to mimic this behavior, but usually have large amounts of lag. Grbl executes overrides in realtime and within tens of milliseconds.

- **Jogging Mode** : The new jogging commands are independent of the g-code parser, so that the parser state doesn't get altered and cause a potential crash if not restored properly. Documentation is included on how this works and how it can be used to control your machine via a joystick or rotary dial with a low-latency, satisfying response.

- **Laser Mode** : The new "laser" mode will cause Grbl to move continuously through consecutive G1, G2, and G3 commands with spindle speed changes. When "laser" mode is disabled, Grbl will instead come to a stop to ensure a spindle comes up to speed properly. Spindle speed overrides also work with laser mode so you can tweak the laser power, if you need to during the job. Switch between "laser" mode and "normal" mode via a `$` setting.

	- **Dynamic Laser Power Scaling with Speed** : If your machine has low accelerations, Grbl will automagically scale the laser power based on how fast Grbl is traveling, so you won't have burnt corners when your CNC has to make a turn! Enabled by the `M4` spindle CCW command when laser mode is enabled!

- **Sleep Mode** : Grbl may now be put to "sleep" via a `$SLP` command. This will disable everything, including the stepper drivers. Nice to have when you are leaving your machine unattended and want to power down everything automatically. Only a reset exits the sleep state.

- **Significant Interface Improvements**: Tweaked to increase overall performance, include lots more real-time data, and to simplify maintaining and writing GUIs. Based on direct feedback from multiple GUI developers and bench performance testing. _NOTE: GUIs need to specifically update their code to be compatible with v1.1 and later._

	- **New Status Reports**: To account for the additional override data, status reports have been tweaked to cram more data into it, while still being smaller than before. Documentation is included, outlining how it has been changed. 
	- **Improved Error/Alarm Feedback** : All Grbl error and alarm messages have been changed to providing a code. Each code is associated with a specific problem, so users will know exactly what is wrong without having to guess. Documentation and an easy to parse CSV is included in the repo.
	- **Extended-ASCII realtime commands** : All overrides and future real-time commands are defined in the extended-ASCII character space. Unfortunately not easily type-able on a keyboard, but helps prevent accidental commands from a g-code file having these characters and gives lots of space for future expansion.
	- **Message Prefixes** : Every message type from Grbl has a unique prefix to help GUIs immediately determine what the message is and parse it accordingly without having to know context. The prior interface had several instances of GUIs having to figure out the meaning of a message, which made everything more complicated than it needed to be.

- New OEM specific features, such as safety door parking, single configuration file build option, EEPROM restrictions and restoring controls, and storing product data information.
 
- New safety door parking motion as a compile-option. Grbl will retract, disable the spindle/coolant, and park near Z max. When resumed, it will perform these task in reverse order and continue the program. Highly configurable, even to add more than one parking motion. See config.h for details.

- New '$' Grbl settings for max and min spindle rpm. Allows for tweaking the PWM output to more closely match true spindle rpm. When max rpm is set to zero or less than min rpm, the PWM pin D11 will act like a simple enable on/off output.

- Updated G28 and G30 behavior from NIST to LinuxCNC g-code description. In short, if a intermediate motion is specified, only the axes specified will move to the stored coordinates, not all axes as before.

- Lots of minor bug fixes and refactoring to make the code more efficient and flexible.


```
List of Supported G-Codes in Grbl v1.1:
  - Non-Modal Commands: G4, G10L2, G10L20, G28, G30, G28.1, G30.1, G53, G92, G92.1
  - Motion Modes: G0, G1, G2, G3, G38.2, G38.3, G38.4, G38.5, G80
  - Feed Rate Modes: G93, G94
  - Unit Modes: G20, G21
  - Distance Modes: G90, G91
  - Arc IJK Distance Modes: G91.1
  - Plane Select Modes: G17, G18, G19
  - Tool Length Offset Modes: G43.1, G49
  - Cutter Compensation Modes: G40
  - Coordinate System Modes: G54, G55, G56, G57, G58, G59
  - Control Modes: G61
  - Program Flow: M0, M1, M2, M30*
  - Coolant Control: M7*, M8, M9
  - Spindle Control: M3, M4, M5
  - Valid Non-Command Words: F, I, J, K, L, N, P, R, S, T, X, Y, Z, A, B, C
```

-------------
Grbl-Mega-5X is an open-source project and fueled by the free-time of our intrepid administrators and altruistic users. If you'd like to donate, all proceeds will be used to help fund supporting hardware and testing equipment. Thank you!

[![Donate](https://www.paypalobjects.com/en_US/i/btn/btn_donate_LG.gif)](https://paypal.me/pools/c/842hNSm2It)

