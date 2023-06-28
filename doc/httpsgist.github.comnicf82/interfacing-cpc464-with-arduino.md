Interfacing an Amstrad CPC464 with an Arduino
=============================================

This is a brief guide showing I managed to hook up an Arduino Uno to my CPC464 to use it as a peripheral device. I did it 
only to learn about retro electronics and electronics in general, so it may not be the perfect design.

I'll aim this at a hobbyist with a pretty basic knowledge of electronics. Mainly because thats all the knowledge I have, so I 
couldn't aim it at anyone else even if I wanted to! But also hopefully it will serve as a guide to some of the basics
which I have been working on.

I would say you will need to have a basic understanding of logic gates, and also some BASIC and C skills for the 
CPC / Arduino programming, although I'll include the code you'll need to get started.

Acknowledgements
----------------

This design is based almost entirely on [this one](http://codinglab.blogspot.com/2013/01/virtual-msx-disk-drive.html) by Raul, which implements an Arduino disk drive emulator for an MSX, another Z80 based computer.

Thanks to everyone from CPCWiki who has given me help in getting this to work. There is some discussion and context in [this thread](https://www.cpcwiki.eu/forum/amstrad-cpc-hardware/interfacing-cpc-with-a-microcontroller/)

An excellent resource for learning about logic gates is [Ben Eater's youtube channel](https://www.youtube.com/channel/UCS0N5baNlQWJCUrhCEo8WlA) and it's where the majority of my knowledge came from.

Equipment Requirements
----------------------

You will obviously need a CPC464 (or 664, 6128 the peripheral interface will be slightly different though I believe) 
and an Arduino, I used an Uno. 

In addition you will need:

 * An expansion board [like this one](https://github.com/revaldinho/cpc_ram_expansion/wiki/CPC-Expansion-Backplane)
 * A male to female 50pin IDC ribbon cable, I had to get the parts and make one 
   * https://uk.rs-online.com/web/p/products/6741221/ 
   * https://uk.rs-online.com/web/p/products/7193449/ 
   * https://uk.rs-online.com/web/p/products/6741114/
 * A large breadboard and some jumper wires
 * Some male to male Dupont cables to connect the breadboard to the ribbon cable
 * For the address decoder logic
   * 2x NS27LS32N OR chips
   * 1x NS27LS00N NAND chip
   * 1x NS27LS04N Inverter chip
 * 1x SN74LS245N Bus transceiver
 
CPC Expansion port and how it talks to peripherals
--------------------------------------------------

The CPC464 expansion port pin layout is as follows:

![Expansion port](https://gist.githubusercontent.com/nicf82/18eced4ebf9648cb47963bdf6f21a345/raw/cb0c5cf5fde2467f98c6d599684f3ef10ba5615b/ExpansionPortEdge.gif)

    1  Sound     2  GND
    3  A15       4  A14
    5  A13       6  A12
    7  A11       8  A10
    9  A9        10 A8
    11 A7        12 A6
    13 A5        14 A4
    15 A3        16 A2
    17 A1        18 A0
    19 D7        20 D6
    21 D5        22 D4
    23 D3        24 D2
    25 D1        26 D0
    27 VCC       28 *MREQ
    29 *M1       30 *RFSH
    31 *IORQ     32 *RD
    33 *WR       34 *HALT
    35 *INT      36 *NMI
    37 *BUSRQ    38 *BUSAK
    39 READY     40 *BRST
    41 *RSET     42 *ROMEN
    43 ROMDIS    44 *RAMRD
    45 RAMDIS    46 CURSOR
    47 LPEN      48 *EXP
    49 GND       50 CLK4

> Source: http://www.cpcwiki.eu/index.php/Connector:Expansion_port

But the only pins we will be using are GND, A8-A15, D0-D7, VCC, /IORQ, /RD, /WR and READY.

GND and VCC are the 0V and 5V supply pins from the CPC. I'm not sure of the maximum current it can supply, but it had no trouble powering the Arduino.

The A* pins are the address bus pins. We only need the 8 highest ones. The address bus on the CPC is used for both reading and writing from memory and IO devices like the one we are building. This brings us to the...

/IORQ pin. The reason for the / in front is because this pin is _active low_. Its considered active when it is at 0V. When this pin is active, it means that the address on the bus refers to an IO address and not a memory address, so we will need to check this pin when responding to the IO requests.

The /RD and /WR pins (also active low) tell the peripheral if the CPU wants to READ data or WRITE data to it. We will check that at least one of these are active.

The READY pin is also refered to as the /WAIT pin and it's purpose is to allow a peripheral to ask the CPU to temporarily pause while we read or write some data from/to the data bus. I'll call it /WAIT from now on as its handy to think of it as an active low signal.

Finally the D* pins are the data bus, they allow the CPU to send and receive data to memory or peripherals, depending on the conditions set out above.

Address decoder and the /WAIT signal
------------------------------------

One of the most complicated things I struggled with when building this was how the /WAIT signal works. I thought I could just plug the address pins into the Arduino then when I saw the correct address, and IORQ was low, I could just get the Arduino to pull the /WAIT pin low and read/write data on the bus at leisure. 

It didn't work, I think even though the Arduino is 4x faster than the CPC, (16MHz vs 4Mhz), this is still not quick enough to stop the Z80 in its tracks while we use the data bus.

So what we need to do is design a logic circuit which will pull the /WAIT pin LOW (within microseconds), when the correct IO address appears on the address bus and other pins.

First of all though, we need to choose an IO address. That is quite simple once you know that all addresses below &F800 are off limits, as these are used by internal devices. In other words the highest 5 bits of the high address byte must be set.

Referring to [this guide](http://www.cpcwiki.eu/index.php/I/O_Port_Summary) I decided to go with &F9xx, but there are other options - it also depends which other peripherals you have plugged in of course.

Since &F9 in binary is %11111001, the logic to decode the address is shown in this diagram

![Address Decoder](https://gist.githubusercontent.com/nicf82/18eced4ebf9648cb47963bdf6f21a345/raw/cb0c5cf5fde2467f98c6d599684f3ef10ba5615b/Address_Decoder.png)

A8-A15 show the logic for decoding the address, so that the output is LOW when the input is &F9.

The /IORQ and at least one of /RD and /WR must be low to keep the /G signal to the 74HC245 LOW (more on /G later)

FORCE_READY is a signal that we generate on the Arduino, and it gives us a way to signal that we are finished processing the current request. If FORCE_READY or /G are high, then /WAIT goes high, and the CPU will stop waiting. In other words, both FORCE_READY and /G must be low to pause the CPU.

We therefore default FORCE_READY to LOW in the Arduino code, and that means whenever the conditions are met that signify a read or write to our device, then /WAIT also immediatley drops LOW.

> The logic above could be simplified by replacing the top NAND/NOT gates following /RD and /WR with a single AND if you have one available. I didn't and it wouldn't reduce the chip count in any case.

So now we understand how we will be notified when there is data available, how do we read/write data?

The 74HC245 transceiver and the /G signal
-----------------------------------------

The transceiver chip is used to give us access to the data bus when we need to read or write from/to it, but not interfere with it when we dont need it. It does this using the /G signal. When /G is low, then the transceiver's A* pins (the CPC side in our case) are connected to it's B* pins (our arduino GPIO pins). 

As described above, /G only goes low under the correct circumstances i.e when the CPC is reading or writing on IO port &F9xx. At any other time, the pins are in a state of High Impedence, this means that they are neither biased toward HIGH or LOW, and basically to us this means the A* pins can be pulled HIGH or LOW by any other devices that need them.

The next interesting pin on the 74HC245 is the DIR pin. This tells the chip which way data should flow. We will connect this to the /RD signal, so if the CPC is reading data, it will be allowed to flow from B to A, and if not, it must be writing, and so data will flow from A to B. 

Arduino connections
-------------------

The Arduino needs access to a few signals from the CPC, and as discussed earlier it needs to be able to output the FORCE_READY signal of its own.

It needs 8 pins connected to pins B0-B7 on the 74HC245, these provide access to the CPC's data bus. I connected B0-B5 to pins 8-13 and B6 & B7 to pins 6 & 7. I would have prefered to use pins 0-7 but this would not leave serial pins 0 & 1 free for debugging.

It also needs access to the /G signal, which is used to check if an IO operation is active on port &F9xx. I connected this to pin 2.

We will connect the /RD pin to the Arduino so it can check which kind of IO operation is being requested. I connected this to pin 4.

Finally the FORCE_READY signal will be output from pin 3.

Assembly on the breadboard
--------------------------

I think this is easiest described with a diagram:

![Breadboard Diagram](https://gist.githubusercontent.com/nicf82/18eced4ebf9648cb47963bdf6f21a345/raw/e93a9a1d8ee3af08a8384f3c23b086e3c70c7311/Breadboard_Diagram.png)

> Note: The chip on the left is the 74HC245, but the tool I used to draw the diagram didn't have this chip, so I joined 2 others together to represent it.

> Note: The random headers dotted around the diagram just represent wires being plugged into the rebon cable attached to the CPC464's expansion port. The wires are labeled indicating there they should go.

Plugging it in
--------------

The best way I have found to plug the wires into the CPC's expansion cable is just with male to male DuPont wires directly into the 50 pin female IDC socket. There are breakout boards available or you could make one. One thing to note is that when I plugged my 50 pin ribbon cable in to the expansion board, the pin with the arrow was not pin 1 as you would expect. At least on my expansion board's layout, pin 1 was on the left.

The setup I have shown here does not show a power wire from the breadboard to the Arduino's Vin pin to power it. You can power the Arduino like this, but to flash it you'll need to power it from USB, in which case don't connect VCC to the Arduino's Vin pin.

>I have noticed a problem when the Arduino is externally powered, in that the CPC wont switch on, if you disconnect the Arduino GND from the breadboard then power it on and re-connect it works. I'm working on a solution for this. Ground loop issue?

Arduino Code
------------

This simple program will just store a byte from the data bus when when a WRITE is requested and write that same byte back when a READ is requested.

The only thing that may need explanation is the FORCE_READY signal. It starts off low so that when /G drops low it pulls /WAIT low. When the code is finished with the data on the bus, it sets FORCE_READY high, which will push /WAIT high again - because it shares an OR gate with /G. It then enters a tight loop on /G waiting for the Z80 to execute the next instruction after the IO request we have just serviced. Once that happens /G will be high and it's safe to pull FORCE_READY low again, ready for the next IO request.

    #define G           2
    #define FORCE_READY 3
    #define IS_WRITE    4

    #define D0   8
    #define D1   9
    #define D2   10
    #define D3   11
    #define D4   12
    #define D5   13
    #define D6   6
    #define D7   7

    char buf[256];

    void setup() {
      Serial.begin(9600);
      
      pinMode(G, INPUT);
      pinMode(IS_WRITE, INPUT);
      
      pinMode(FORCE_READY, OUTPUT);
      digitalWrite(FORCE_READY, LOW);   //Allow wait pin to be pulled low by /G

      busMode(OUTPUT);
      
      Serial.println("Ready");
    }

    void busMode(int mode) {
      pinMode(D0, mode);
      pinMode(D1, mode);
      pinMode(D2, mode);
      pinMode(D3, mode);
      pinMode(D4, mode);
      pinMode(D5, mode);
      pinMode(D6, mode);
      pinMode(D7, mode);
    }

    byte b = 255;

    void loop() {
        
      //If no I/O is not being with this device, return
      if(digitalRead(G)) return;

      if(digitalRead(IS_WRITE)) {
        Serial.println("Write was requested");

        busMode(INPUT);
        
        b = PINB | ( PIND & B11000000 );

        sprintf(buf, "Received: %d", b);
        Serial.println(buf);
        
        digitalWrite(FORCE_READY, HIGH);  //Cancel the hardware wait, /WAIT == G + FORCE_READY
        while(!digitalRead(G));           //Wait till IO is done (probably instantly)
        digitalWrite(FORCE_READY, LOW);   //Allow wait pin to be pulled low by /G again
        
      } else {
        
        Serial.println("Read was requested");

        busMode(OUTPUT);
        
        PORTB |= (b & B00111111);
        PORTD |= (b & B11000000);
      
        digitalWrite(FORCE_READY, HIGH);  //Cancel the hardware wait, /WAIT == G + FORCE_READY
        while(!digitalRead(G));           //Wait till IO is done (probably instantly)
        digitalWrite(FORCE_READY, LOW);   //Allow wait pin to be pulled low by /G again
      
        busMode(INPUT);
      }
    }

CPC Code
--------

This is almost too simple to include, but running:

    OUT &F900, 128

Will store 128 on the Arduino, then to read it back, you can run:

    PRINT INP(&F900)

You should get the value you set echo'd back to you!

Things to try
-------------

I have managed to get the Arduino to work with an analog Joystick (not with any games yet, just moving cursor round the screen). My next plan is to use the Arduino to drive an LCD display, and then design a circuit to drive the LCD direct from the CPC.