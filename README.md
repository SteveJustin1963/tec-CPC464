# tec-CPC464

CPC464-TEC-1 bridge 
- run experiments


## nicf82.f
convert to Forth
- configuring pin constants and defining various functions.
- Here's an explanation of the code:

The lines beginning with `2 CONSTANT G`, `3 CONSTANT FORCE_READY`, `4 CONSTANT IS_WRITE`, and so on define constants. Constants in Forth are like variables, but their values cannot be changed once defined. These constants are assigned specific values:
- `G` is assigned the value 2
- `FORCE_READY` is assigned the value 3
- `IS_WRITE` is assigned the value 4
- `D0` to `D7` are assigned the values 8 to 13 and 6 to 7, respectively

Next, the line `VARIABLE buf 256 ALLOT` declares a variable named `buf` and allocates 256 bytes of memory for it.

The `setup` function is defined with the `:` colon definition word. It sets up the initial configuration by:
- Starting the serial communication at a baud rate of 9600 using `9600 SERIAL-BEGIN`
- Configuring the pins:
  - `G` and `IS_WRITE` as input pins using `G PIN-INPUT` and `IS_WRITE PIN-INPUT`
  - `FORCE_READY` as an output pin using `FORCE_READY PIN-OUTPUT`
- Setting `FORCE_READY` pin to a low state using `LOW FORCE_READY DIGITAL-WRITE`
- Calling the `busMode` function with the parameter `OUTPUT` to set the pin modes for D0-D7 as output
- Printing "Ready" to indicate the setup is complete using `." Ready" CR` (CR adds a carriage return to move to the next line)

The `busMode` function is defined to set the pin modes for D0-D7 based on the provided mode. It uses the `DUP` word to duplicate the mode on the stack for each pin and then calls `PIN-MODE` to set the pin mode accordingly.

The line `255 VARIABLE b` declares a variable named `b` and initializes it with the value 255.

The `loop` function is defined to perform the main logic. It checks if pin `G` is high using `G DIGITAL-READ`. If it is high (true), it exits the loop using `EXIT`. Otherwise, it checks if pin `IS_WRITE` is high. If it is high (true), it executes the code inside the `IF` branch:
- Prints "Write was requested" to indicate the write operation using `." Write was requested" CR`
- Sets the pin mode for D0-D7 as input using `INPUT busMode`
- Performs bitwise operations on the pins `PIND` and `PINB` and stores the result in the variable `b` using `PIND B11000000 AND PINB OR b !`
- Prints the value of `b` using `b @ .`
- Prints "Received" on a new line using `." Received" CR`
- Sets the `FORCE_READY` pin to a high state using `HIGH FORCE_READY DIGITAL-WRITE`
- Waits until pin `G` is high (0 is false, so it repeats the loop until `G` is high) using `G DIGITAL-READ 0= UNTIL`
- Sets the `FORCE_READY` pin to a low state using `LOW FORCE_READY DIGITAL-WRITE`

If the `IS_WRITE` pin is low (false), the code inside the `ELSE` branch is executed:
- Prints "Read was requested" to indicate the read operation using `." Read was requested" CR`
- Sets the pin mode for D0-D7 as output using `OUTPUT busMode`
- Performs bitwise operations on the variable


## ref
- https://www.indieretronews.com/2016/02/ddi3-usb-floppy-emulator-for-amstrad.html
- https://www.tindie.com/products/bobsbits/amstrad-cpc464-replica-pcb/
- https://gist.github.com/nicf82/18eced4ebf9648cb47963bdf6f21a345
- 
