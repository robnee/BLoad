Introduction

The BLoad development platform is a combination of open source software, hardware and a firmware framework that dramatically speeds firmware development. These work together to facilitate a fast and efficient development cycle. It turns a PIC 16F819 into a system similar to higher priced stamp-like microcontrollers. The tool chain, utilities are all free and open.

The efficiency of the normal firmware development cycle of change, build, load and test is typically hampered by the load process. Loading new firmware requires reprogramming the microcontroller outside the target circuit. While solutions for In Circuit Serial Programming exist there are often limitations and design rules that may interfere with the target system. Using a bootloader the BLoad Platform allows the microcontroller firmware to be downloaded via a serial port. The microcontroller is self programming and the serial port can be used to debugging and other system functions in the target design including user driven field upgrades.

The goal of this system is a fast, robust and easy to use development platform. This article will attempt to explain how you can use it to develop or modify your own firmware for flight electronics.
Components

There are three main components to the platform: a firmware framework, a baseline hardware platform and a bootloader driver running on a host system. The host system described here is a Linux PC but the system is open and can be ported to other platforms such as Windows or OS X provided a serial port is available. USB Serial ports will typically work too. The host system is used to control the entire design cycle including writing the firmware code, compiling it and downloading it to the target microcontroller. By controlling the platform from the host the developer is freed from repeatedly swapping chips in and out of the target circuit to test changes. The host interfaces with the microcontroller using a simple circuit that provides a serial interface with reset control. This hardware can be built into the target design or implemented on a prototyping board. I have encapsulated this circuity into a small module called the BLoad Interface Adapter (shown above) and into a 24 pin stamp-like module. The firmware is developed using a framework or template that provides the bootloader and serial port services that the system needs. The firmware is developed in PIC Micro Assembler and makes liber use of macros and has a solid base of utility routines for things like math, button interface, beepers, i2c, servo driver to name a few.
Firmware Framework

In order for firmware programs to work with the BLoad Platform to work they must adhere to a framework. The platformThere are some restrictions this entails but for the most part they are minor and some of the resources consumed produce benefits outside the straight bootloader functions such as a serial port for debugging or other communications. Here's a quick list of reserved resources:

    The top 256 words of program memory are reserved for the bootloader, low-level serial routines and flash I/O routines.
    Bank 2 SRAM is reserved for the bootloader transmit buffer
    RB0 and the INT interrupt is reservered for serial port RxD
    RB1 is reservered for serial port TxD
    Location 0x04 must contain an ISR for INTF that vectors to the ser_in routine in the bootloader
    MCLR must be enabled
    internal weak pullups must be disabled. Use external pullups or counteract with a strong pulldown on RB0
    Must use an internal or external 4MHz oscillator

I'd like to go through an example program but first let's discuss the toolchain for building the source code. It consists entirely of open source or free software. All are available for download on the web. I prefer a Linux-based tool chain and will describe it here but you can use Windows substitues if you like.

To build the source code you will need an assembler. I use GPASM that is part of the GPUTILS package. Prebuilt packages are available for Linux and Windows and I'd bet someone has ported the code to OS X and other UNIX platforms. Since GPASM is just an open source clone of MPASM, the Microchip assembler for PIC microcontrollers, you can, of course, use that as well. The Microchip IDE is very nice but I am an old-school command-line type guy and old habits die hard. Links to both these packages are below.
GPUTILS from SourceForge
MPLAB From Microchip

Let's take a look at a simple program using the framework and go through it. It's called blink and all it does is blink an LED connected to port RA1. It may look like a lot of code to blink an LED, and it is, but it also includes the framework. Once you understand how it works it will become more of a help than a hinderance.
`
   1 ;----------------------------------------------------------------------
   2 ;
   3 ; File: blink.asm
   4 ;	Blink an LED on RA1
   5 ;
   6 ;	Supports bootloader interface with DTR/MCLR control
   7 ;
   8 ;	Pin		Func	Desc
   9 ;	18		RA1		LED
  10 ;	4		MCLR	DTR (4)
  11 ;	5		VSS		GND (5)
  12 ;	6		RxD		TxD (3)
  13 ;	7		TxD		RxD (2)
  14 ;	14		VDD		+5V
  15 ;
  16 ; Author:
  17 ;	Robert F. Nee
  18 ;	robnee@robnee.com
  19 ;
  20 ; Revisions:
  21 ;	last delta 02/16/07 21:46:39
  22 ;
  23 ; v1.0 04/13/2007
  24 ;
  25 ;----------------------------------------------------------------------
  26 
  27 ;----------------------------------------------------------------------
  28 ; Processor-dependant includes and defines
  29 
  30 		errorlevel -302
  31 
  32 		radix	dec
  33 
  34 		; Select the target processor
  35 		#include "p16f819.inc"
  36 
  37 		; CPU configuration
  38 		__config  _INTRC_IO & _WDT_OFF & _BODEN_OFF & _PWRTE_ON & _MCLR_ON & _LVP_OFF
  39 
  40 ;----------------------------------------------------------------------
  41 
  42 #ifdef GPASM
  43 #include "macros.inc"
  44 #else
  45 #include "..\include\macros.inc"
  46 #endif
  47 
  48 ;----------------------------------------------------------------------
  49 ; Pin Definitions
  50 
  51 trisb_mask equ b'00000001'
  52 #define serin_pin   PORTB, 0
  53 #define serout_pin  PORTB, 1
  54 
  55 trisa_mask equ b'00000000'
  56 #define led_pin     PORTA, 1
  57 
  58 ;----------------------------------------------------------------------
  59 ; Constants
  60 
  61 BAUD_RATE	equ		9600
  62 
  63 ;----------------------------------------------------------------------
  64 ; Variables
  65 
  66 		cblock 0x7E
  67 w_temp
  68 status_temp
  69 		endc
  70 
  71 		cblock 0x20
  72 i
  73 		endc
  74 
  75 ;----------------------------------------------------------------------
  76 ; Define the initial contents of the EEPROM.
  77 
  78 		org	0x2100
  79 
  80 ;----------------------------------------------------------------------
  81 ; Power-on reset startup location
  82 
  83 		org		0x0
  84 		nop					; For ICD
  85 		goto	init
  86 
  87 ;----------------------------------------------------------------------
  88 ; Interrupt service routine
  89 
  90 iserv	org	0x04
  91 
  92 		; Save STATUS and W
  93 		int_save
  94 
  95 		ifbs 	INTCON, INTF	; Serial Input
  96 		 call	ser_in
  97 		 bcf	INTCON, INTF
  98 		endif_
  99 
 100 		; Other interrupt checking code here
 101 
 102 		; Restore STATUS and W
 103 		int_restore
 104 
 105 		retfie
 106 
 107 ;----------------------------------------------------------------------
 108 ; Initialization
 109 
 110 init    ; turn off comparators and A/D
 111 		clrf    CCP1CON
 112 		clrf	ADCON0
 113 
 114 		clrf	PORTA
 115 		clrf	PORTB
 116 
 117 		bank1
 118 
 119 		; Configure PORTA for digital I/O
 120 		movlw   b'00000110'
 121 		clrf   ADCON1
 122 
 123 		; Initialize the IO port direction
 124 		movlw   trisa_mask
 125 		movwf   TRISA
 126 		movlw   trisb_mask
 127 		movwf   TRISB
 128 
 129 		; Select 4MHz internal oscillator and wait for clock to stablize
 130 		movlw   b'01100000'
 131 		movwf   OSCCON
 132 		loop
 133 		 btfss  OSCCON, IOFS
 134 		endloop
 135 
 136 		; Check for bootloader
 137 		call	boot_detect
 138 
 139 		bank0
 140 
 141 ;----------------------------------------------------------------------
 142 ; Main loop
 143 
 144 main
 145 		loop
 146 		 bsf	led_pin
 147 		 call	pause_1s
 148 		 bcf	led_pin
 149 		 call	pause_1s
 150 		endloop
 151 
 152 ;----------------------------------------------------------------------
 153 
 154 #ifdef GPASM
 155 #include "pause.asm"
 156 #include "bootload.asm"
 157 #else
 158 #include "..\include\pause.asm"
 159 #include "..\include\bootload.asm"
 160 #endif
 161 
 162 ;----------------------------------------------------------------------
 163 
 164 		end
 165 
`
The first 50 lines are mostly boilerplate. The first section is a header with hookup, version, author and revision information. The next block contains compiler directives for setting the default radix for numerical constants, suppressing certain unnecessary compiler messages and including the header file for the PIC 16f819 processor. The header provices manifest constant for register locations, bit definitions etc. The __config directive specifies the values of the fuse bits. It is important to turn MCLR on and LVP (low voltage programming) off. The next block includes a collection of macros that provides some high-level constructs to the programmer such as loops, structured if and some convenience macros. If GPASM is being used the assumption is that macros.inc is in a separate directory and made available via the -I directive. MPLAB on the PC does not have this capability so the path to an adjacent include directory is specified explicitly.

Line 50 begins the pin definitions section. The mask values specify pin directions (1 is input 0 is output). The #defines create symbolic names for the pins. serin_pin and seroaut_pin are use by the serial I/O routines and are required. In addition the trisb_mask bits 0 and 1 must be set to 1 and 0 respectively in order to set up the proper pin direction for the serial port pins. The led_pin is part of the "user" portion of the program. You can add your own here.

The next two sections declare user constants and variables. The BAUD_RATE must be defined for the serial I/O library. The variables sections associates variable names with memory locations. Two variables w_temp and status_temp are declared for use by the ISR and are located at the top of the first bank of memory on the PIC 16F819. A second cblock for user memory locations begin at 0x20. This example declares a byte variable name i that actually isnt used. It's is included for demonstation purposes. The final section before the actual program code begins defines the contents of the EEPROM data memory. On the PIC 16F819 there is 256 bytes of available entirely for use by the user program. By convention the address for the start of data EEPROM memory is 0x2100.

Program memory location 0x0 is the start of program memory and the power-on reset startup location. The first location is a nop instruction to reserve the location for the Microchip ICD 2 debugger. The interrupt service routine (ISR) must begin at location 0x04 so the next instruction simply jumps over the routine to the initialization code. The ISR must first save the values of the W and STATUS registers as they may be in use at the time the interrupt is triggered. The macros (from the macro.inc include file) int_save and int_restore handle this for us and are mandatory. Once the registers are saved the ISR checks the INTF flag to determine if a serial receive interrupt occurred. If so a call is made to ser_in to handle the serial byte. The ser_in routine is part of the bootloader. If there are any other interrupt requests that should be handled they should be placed next. The end of the ISR calls the int_restore routine to restore W and STATUS immediately followed by a retfie instruction.

The init routine beginning on line 110 sets up the on-chip resources such as pin direction, whether the analog to digital converters and comparators are enabled and configures the internal oscillator to run at 4MHz. You are free to use an precision external oscillator but there are timing sensitive routines that expect a 4MHz clock so if you use a different frequency you would have to tweak them. These include the pause, serial I/O and servo routines to name a few. The last thing the initialization code does is make a call to boot_detect routine. This routine looks for a break condition on the serial port. If it doesn't see one after 250ms it gives up and returns. If it does, however, the bootloader is started and awaits programming commands. This allows the chip to behave normally and will only invoke the bootloader if the chip is reset and immediately sent a break. In case you haven't figured it out this is exactly what the bootloader driver does.

Next up is the main body of the program. It's exceeding simple. Inside the main loop the LED is turned on for a second and then off for a second. The whole thing repeats forever. This isn't much but it is sufficient for testing. The final block is more include statements for the pause routines used by the main program and the bootloader itself. Again, how they get included is dependant on the compiler used. It is best to place these includes at the end of your program.
Baseline Hardware

The BLoad platform requires a small amount of external hardware to operate. This hardware provides the serial port interface and reset circuit. In addition to this physical hardware there are some configuration requirements in order for everything to operate properly. This hardware is usually implemented in the prototype and final design using a few inexpensive discrete parts. In addtion, there are various options such as plug-in modules to speed up prototyping and help the system fit existing designs.
can be simpl

AYUCR BLoad Schematic

AYUCR BLoad Schematic

The first part of the external hardware is the reset circuit. Pin 4 on a PIC 16F819 is denoted /MCLR and is an active-low reset pin. When the pin is brought momentarily low the processor resets. This is interfaced to the DTR pin of the host serial port through a 2N2222 NPN transistor so that when the DTR line is raised by the host the processor resets. A pull-up resistor on the PIC side holds /MCLR high so the pin doesn't float and sporadically reset. The bootloader driver needs to reset the PIC as part of the sequence to reflash the firmware or to simply reset the processors during debugging.

The remainder of the circuit is a simple serial port. It does not use the typical level-shifting chip like a MAX232 in order to keep things as simple as possible. It works because while the RS232 standard calls for +12V/-12V signaling in practice +5V/0V will work as well. The only thing we need to do is drop the incomming voltage using a 2.2K ohm resistor and use a weak pull-down to make sure the line is held "high." I say high because in RS232 signalling a '1' bit is represented by -12V and a '0' by 12V. This scheme does require inverting the bits in software but this is handled by the serial I/O routines in the bootloader. The TxD line is connected directly. This scheme works as long as the PIC is powered by at least a 5V supply. Even though the PIC can work down to 2V the signaling drops with the supply voltage. At some point the host will fail to see valid RS232 levels and communication will be lost. One could deal with this using a real RS252 level shifter but that is outside the scope of this discussion

Chances are you will leave the RS232 serial port in your target design but you may not want or need the reset circuit. You cannot leave it disconnected as a floating pin will tend to randomly fluctuate and reset your circuit when you don't want it to. You have 3 options: implement the reset circuit as shown, replace the reset circuit with a single 4.7K ohm pull-up resistor or configure the /MCLR pin as a normal I/O pin. The last option is handled by specifying _MCLR_OFF instead of _MCLR_ON in the config word in your source code (line 38 above) or when programming. If you don't implement the reset circuit you must add a way in your source code to start the bootloader if you want to be able to reflash the firmware in your target design. The normal reset/break method won't be available. This is discussed in some more depth in the article on the AYUCR Bootloader.

In addition to the physical hardware there are a few configuration requirements. As mentioned above the /MCLR pin must be enabled in order to allow the chip to be remotely reset. This is done via the configuration word for the processor. You will typically specify _MCLR_ON as part of the __config compiler directive which sets the config word in the resultant .HEX file. You can also override the setting in most programmer interfaces but this is not recommended. The PIC 16F819 has an option for weak internal pullups on the PORTB pins. This should be left disabled. If you enable it it will interfere with the break signal detection on the serial port. Alternatively you can replace the 100K ohm pulldown with a stronger 10K ohm pulldown but, again, I recommend discrete, external pull-ups where necessary.

Baseline Hardware

Baseline Hardware laid out on a protoboard

The photo above shows the baseline hardware implemented on a protoboard. It's not as complicated as it looks but it can get in the way. You can certainly rearrange it to suit your needs. I have developed a module that I call the Bload Interface Adapter that combines all of the hardware in addition to a power indicator in a small plug-in module. It makes wiring fast and fool-proof. I use it for all my work.

I have also been working on a 24-pin module that combines the baseline hardware and an 18 pin socket for the PIC 16F819. It shares a similar pinout as a Basic Stamp

BLoad Module

BLoad Module
Host Configuration

This BLoad Development Platform uses a host computer to edit, build and download firmware to the target microcontroller. How the firmware is edited and built into a .HEX file suitable for downloading can be accomplished in different ways. I prefer the open source GPASM assembler which is available for a variety of platforms. The key to the platform is the bootloader driver that controls the target microcontroller and can reset it, reprogram it and aid in debugging. This section will describe how to set it up and get it running under Linux.

The bload.pl bootloader driver is developed in Perl so naturally you must have the perl package installed to run it. Any 5.8.x version of later should work and chances are that you already have this installed anyway.

The bootloader driver uses the Device::SerialPort package for serial port communications. This package was chosen because it mimics the Win32::SerialPort package on Windows and should facilitate easy porting of the bootloader driver to Windows and something like ActiveState Perl. Typically this package must be installed in order for the script to work. Follow the instructions that come with the package. In order to communicate with the target your host PC must also have a serial port. A real, true serial port is prefered and typically is easier to get working. Many newer, legacy-free, PCs don't come with serial ports anymort however. In this case you'll either need to use a PCI card or a USB to serial adapter. These should work fine but configuring them is outside the scope of this article.

One problem I ran into on Linux is permissioning. The serial ports (/dev/ttyS0) on my Mandriva Linux are protected and I would get 'access denied' type errors accessing them. I tried changing the permissions of the ports but they would revert to protected on reboot. The solution was to add my userid to the uucp group which does have permission to use the ports. You may need to do something similar on your distro of Linux
Cabling

The BLoad driver requires a 4 wire cable to communicate with and program the PIC 16F819 bootloader. In addition to the usual ground, transmit and receive lines a reset line is also required. The serial port DTR line (pin 4) is used to implement the reset line. This cable is simple to make with a few simple parts.

BLoad Cable

BLoad Cable
Parts List Female 9 pin D-Sub connector
9 pin D-Sub Hood with hardware
4 conductor cable. Standard phone cable workes well
4 pin female Waldom or other 0.100" spacing connector
4 pin male right-angle waldom or other 0.100" spacing connector
Crimp pins

Step 1: Cut off about 1" of the outer jacket at each end of the length of cable. The cable requires four conductors. Strip about 3/16" of insulation from each wire of each wire.

Step 2: Using needle nose pliers secure a crimp pin to each wire at one end of the cable. The tabs on the crimp pins are designed with a split in them. One part of each tab secures the insulation and the other part make contact with the exposed wire. Do not over crimp or you may deform the wire and weaken the connection. If you purchased a cable with precrimped connections then you can skip this step.

Step 3: Slide the crimp pins into the three pin connector in the order shown in the picture below. Each crimp pin has a little, raised catch that locks into the slots on the bottom of the connector. Pin 1 is on the right in the photo.

4 Pin Connector

Wiring Order for 4 pin connector

Step 4: On the other end of the cable solder the wires to the solder cups on the back side of the 9 pin connector as follows. The Pin numbers are marked on the connector
Wire Color 	Pin Name 	Pin Number
Yellow 	RxD 	2
Green 	TxD 	3
Red 	DTR 	4
Black 	Gnd 	5

DSub wiring

Wiring the 9 pin D-sub connector

Step 6: Snap the hood over the connector and secure it with the screws and nuts provided.

Step 7: You will also need a gender changer to plug the cable into a proto board. Bend a right-angle connector so that the pins are straight. Trim the ends to be about 3/8" in length otherwise the connector will bottom out in the proto board.

Gender changer

Gender changer made from a bent right-angle connector

That's it. If you were careful with the wiring the cable should work fine. Problems are almost always due to a miswired cable. Check your connections carefully.

Cable Plugged In

Cable plugged into a proto board using a gender changer
Using the Bootloader Driver

With everything connected and working it's time to test the bootloader driver. What does that mean?

    Host configured with a working serial port
    Cable connecting serial port to target PIC 16F819 via baseline hardware
    Targat PIC pre-programmed with BLoad framework firmware (such as blink.hex)
    Target platform powered up 

The help screen you get if you run the bload.pl script with no arguments is below

[rnee@Server blink]$ bload.pl
usage: bload.pl [-p=port] [-b=rate] [-q] [-f] [-w] [-t] <filename.hex>
   -p  followed by =port.  default is /dev/ttyS0
   -b  Followed by =rate.  default is 9600
   -q  Quiet mode.  Don't dump hex files
   -f  Fast mode.  Do minimal verification
   -w  Write Only mode.  Fastest but does no verification
   -t  Start terminal on exit to capture debug output
       Version bload.pl @(#)bload.pl    1.23 03/15/07

The -p option allows you to specify the serial port name to use. The default is /dev/ttyS0

The -b option allows you to specify the baud rate. The default is 9600 baud

The -q option suppresses the display of the contents of the hexfile it is loading and the firmware it reads from the target.

The -f option speeds up the download operation by only reading the portion of the existing firmware necessary to compare the integrity of the bootloader itself

The -w option speeds up the download operation by skipping the firmware verify step.

The -t option starts up a simple debugging terminal after the firmware is reflashed in order to send and receive debugging messages.

The last arg is mandatory and is the .hex file to upload.

If there are problems communicating, reading, writing or verifying the firmware you will receive an error message. They should help you debug the problem.
Troubleshooting

If you get a message that there is a bootloader region mismatch it means the bootloader in the target does not match the bootloader in the .hex file you are trying to download. This could end up causing a problem because if the locations move then the chip could stop communicating and would need to be reflashed using an external programmer. The bootloader driver will warn you of this

If you get a message that you can't open the serial port check that the name of the port is correct. The default is /dev/ttyS0. If your serial port has a different name then use the -p option to specify the name. Also make sure you have permissing to access to serial port. If not you may need to permisson your account by looking at the group that owns the serial port and adding yourself to that group. For example assuming you are user rnee. Log in as root and do something like:

[root@Server bload]# ls -l /dev/ttyS*
crw-rw----  1 root uucp 4, 64 May 24 20:29 /dev/ttyS0
crw-rw----  1 root uucp 4, 65 May 17 20:57 /dev/ttyS1
crw-rw----  1 root uucp 4, 66 May 17 20:57 /dev/ttyS2

[root@Server bload]# usermod -G uucp rnee

If you find the process of flashing the target too slow then experiment with using the -f -w options to speed the process up. 
