# 6502 system software

This documentation provides necessary insight into software provided in the repository.

## Getting started with provided software

Before discussing the details of the contents, I will show how to use sample programs for smoke testing of the soldered board.

I recommend running first couple of programs in slow clock mode, so with external clock module, with Arduino Mega based bus analyzer.

### Connecting Arduino Mega

First, make sure you have the 6502-monitor program installed on your Arduino Mega board. You will find it in `Arduino/6502-monitor` folder.

Connect pins as per following table:

|Expansion port pins  |Arduino Mega pins|
|---------------------|-----------------|
|D0-D7 (data bus)     |53-39            |
|A00-A15 (address bus)|52-22            |
|CLK (system clock)   |2                |
|R/W (read/write)     |3                |
|GND                  |GND              |
|+5V                  |+5V              |

**PLEASE NOTE: Connections are reversed for convenience. D0 is connected to pin 53, D1 to pin 51, A00 to pin 52 and A15 to pin 22.**

Also, connect clock module as follows:

|Computer connection         |Clock module connection|
|----------------------------|-----------------------|
|CLK input (middle pin on J1)|CLK output             |
|UART +5V                    |+5V                    |
|UART GND                    |GND                    |

It might seem strange that the clock module is powered from UART port, but in fact it doesn't matter which of the power outputs you choose. I use this one, since this one is closes to the clock connector (J1). This is one thing I didn't consider when designing the PCB - I provided GND reference in J1, but the power is missing...

The whole thing will be powered from Arduino via USB - don't worry, the load will not be too high.

### Running the simplest possible program

Now you need to build the first program. Go to `Software/rom/01_nop_fill` folder and run:

```shell
make clean all test
```

You expect output similar to the following:

```text
rm -f ../../build/rom/*.bin \
  ../../build/rom/01_nop_fill/*.o \
  ../../build/rom/01_nop_fill/*.lst \
  ../../build/rom/01_nop_fill/*.s \
  ../../build/rom/*.map \
  ../../build/common/*.o \
  ../../build/common/*.lst \
  ../../build/common/*.s \

ca65 --cpu 65C02 -Dfastclock=1 -I ../../common/include -o ../../build/rom/01_nop_fill/nop_fill.o -l ../../build/rom/01_nop_fill/nop_fill.lst nop_fill.s
cl65 -t none -C ../../common/firmware.ext.cfg -m ../../build/rom/01_nop_fill.ext.map -o ../../build/rom/01_nop_fill.ext.bin ../../build/rom/01_nop_fill/nop_fill.o
hexdump -C ../../build/rom/01_nop_fill.ext.bin
00000000  ea ea ea ea ea ea ea ea  ea ea ea ea ea ea ea ea  |................|
*
00008000
MD5 (../../build/rom/01_nop_fill.ext.bin) = 49d01fd92a6a02370364f8eef2ee2c93
```

If you remember Ben's video - this is the first program he uploads to ROM. Now, run `make install` to upload the binary to your EEPROM - I assume you put the ROM chip in TL866II+ programmer and it is connected to your machine.

```shell
make install
```

You expect the following output:

```text
minipro -p AT28C256 -w ../../build/rom/01_nop_fill.ext.bin
Found TL866II+ 04.2.109 (0x26d)
Erasing... 0.02Sec OK
Protect off...OK
Writing Code...  6.78Sec  OK
Reading Code...  0.49Sec  OK
Verification OK
Protect on...OK
```

Plug the ROM back in your board, connect the Arduino to your PC, toggle clock module to manual mode, reset 6502 computer and start serial monitor. Go a few cycles step by step and you should see something similar to the below in serial monitor:

```text
10:13:51.707 -> 000000   1110101011110010   11101010   eaf2  r ea
10:13:52.315 -> 000001   1111111111111111   11101010   ffff  r ea
10:13:52.776 -> 000002   1110101011110010   11101010   eaf2  r ea
10:13:53.201 -> 000003   0000000111111101   10111101   01fd  r bd
10:13:53.585 -> 000004   0000000111111100   10111001   01fc  r b9
10:13:54.074 -> 000005   0000000111111011   10000001   01fb  r 81
10:13:54.576 -> 000006   1111111111111100   11101010   fffc  r ea
10:13:55.562 -> 000007   1111111111111101   11101010   fffd  r ea
10:13:56.020 -> 000008   1110101011101010   11101010   eaea  r ea
10:13:56.511 -> 000009   1110101011101011   11101010   eaeb  r ea
10:13:56.982 -> 000010   1110101011101011   11101010   eaeb  r ea
10:13:57.337 -> 000011   1110101011101100   11101010   eaec  r ea
10:13:57.659 -> 000012   1110101011101100   11101010   eaec  r ea
10:13:57.940 -> 000013   1110101011101101   11101010   eaed  r ea
10:13:58.253 -> 000014   1110101011101101   11101010   eaed  r ea
10:13:58.658 -> 000015   1110101011101110   11101010   eaee  r ea
```

First six lines might be different, but starting from line 7 you should see identical output. If this works, it means your CPU and ROM are working just fine, your clock module and Arduino Mega bus analyzer are attached correctly. Congratulations!

If you have any problems during build, installation or execution - check instructions above again, maybe you missed something.

### More complex programs

After having ran the first one, try executing the following programs:

* `Software/rom/02_nop_fffc` - this one will jump to the beginning of the accessible ROM address space and will confirm your address decoder functions correctly,
* `Software/rom/03_first_code` - this is another program taken directly from Ben's videos - it will store value 0x42 in the address 0x6000. Please note: this will have no effect whatsoever.

Now, if the both above work as expected (when checked using bus analyzer), you can try connecting peripherals to your computer. If you want to follow Ben's videos, keep reading this section, otherwise, skip to [next one](#initiate-warp-speed).

First, let's play with some LEDs. Using the connectors in bottom left corner of the PCB, connect 8 LEDs to VIA2 PORTB lines PB0-PB7, and then, using current limiting resistors of 220Ohm, connect these to ground (also from the VIA2 PORTB connector).

Having these connected, upload `Software/rom/04_blink_s` or `Software/rom/05_knight_rider` to your ROM. After powering on, you should see LEDs blinking in a way similar to what Ben did in his videos. If it works correctly, you can move on to connecting LCD. For now, use it in 8-bit mode with slow clock - just as in Ben's videos.

To do this, connect LCD to breadboard (not to the dedicated LCD port on PCB), and then connect each line as listed below:

|VIA2 Pin|LCD Pin|
|--------|-------|
|PA5     |RS     |
|PA6     |R/W    |
|PA7     |E      |
|PB0     |DB0    |
|PB1     |DB1    |
|PB2     |DB2    |
|PB3     |DB3    |
|PB4     |DB4    |
|PB5     |DB5    |
|PB6     |DB6    |
|PB7     |DB7    |

Also, connect A and VDD connectors to +5V, K and VSS to GND and connect V0 to a middle pin of 10KOhm potentiometer plugged between GND and +5V.

Now upload program `Software/rom/06_lcd_test` to ROM - when executed, it should display "Merry Christmas!" message on the LCD.

If it works correctly, follow with `Software/rom/07_mem_test` and `Software/rom/08_stack_test` - these will test if RAM works as expected: first one will copy data from ROM to RAM, while the second will use stack routines. Congratulations, you have working CPU, ROM, RAM, VIA and address decoder!

### Initiate warp speed

Now it's time to go a bit faster and test the more complex features. Please note: you could keep using the analyzer and external clock with these programs, you just have to remember to build them with `FASTCLOCK=0` flag. More details can be found in [building software section](#building-software).

For now let's assume we move to 1MHz clock. To do it, put jumper on two leftmost pins of clock connector (J1). Disconnect external clock and bus analyzer - first one is not needed, second one will not work with high frequencies anyway. Connect your LCD to onboard LCD port and upload `Software/rom/13_4bit_lcd` to your ROM. Upon boot you expect to see message "Hello 4-bit! Chars!" on your screen. If it works, it means that the primary VIA works just fine and clock is OK.

Next one to test will be serial connection, so upload `Software/rom/15_serial_irq`. Now, depending on whether you soldered on FT230X chip or not, connect your board using USB cable to PC (using either MicroUSB or USB-B port), or use external USB->UART connector. Connect to your board using `picocom` with baud rate of 19200.

```shell
picocom -b 19200 /dev/tty.usbserial-HANF88HD
```

When you get "Terminal ready" message, press any key - you should get a response of "Hello IRQ>". This means that two things are working correctly: interrupt handling and serial communication. Congratulations, you are almost ready to go.

### Keyboard connection

Even if you don't intend to use keyboard just yet, you still need to upload the controller sketch to ATtiny4313. Recommended way of doing that is to use onboard AVR-ISP connector and some kind of AVR programmer. I used USBASP programmer and it works lovely directly from Arduino IDE. The sketch to upload is in `Arduino/keyboard-4313` folder.

After successful sketch upload, flash your rom with `Software/rom/19_keyboard_test`. Connect your PS/2 keyboard to the port and try pressing some keys - you should see messages on the LCD with confirmation.

### Using the bootloader

Currently only the minimal bootloader is provided, but it should be sufficient for software development without constant need to reflash the EEPROM. To use it, build ROM image in `Software/rom/minimal_bootloader` folder and flash it to EEPROM. To test this functionality, you have to build example loadable programs in `Software/load/01_blink_test` and `Software/load/02_hello_world`.

**PLEASE NOTE:** Both the bootloader and sample programs will be built automatically when invoking `make all` directly in `Software` folder.

Upon boot you will be prompted to connect to the PC via serial connection and press Enter key - either in termina window if keyboard is not connected, or on the keyboard otherwise. Connection details will be displayed on the LCD:

* 19200 baud,
* 8-bit, no parity, 1 stop bit,
* CTS/RTS hardware flow control.

In MacOS/Linux you can use `picocom` for this operation, under Windows I have successfully used [ExtraPuTTy](https://www.extraputty.com/).

After connection is established you need to press enter as prompted (either on PS/2 keyboard or terminal window) and you will be prompted to initiate file transfer. In `picocom` this requires that your send command is set to `sz -X` (see `make terminal` target in `Software/common/makefile`) and you initiate transfer with Ctrl+A followed by Ctrl+S. Enter load file path (i.e. `Software/build/load/01_blink_test.load.bin`) and press enter. If the transfer fails, try again. `picocom` seems to fail every now and then, while ExtraPuTTy hardly ever has any issues.

In ExtraPuTTy open "Files Transfer" menu item, then "Xmodem" and "Send". Point to loadable module (i.e. `Software/build/load/02_hello_world.load.bin`) and click "Open" button.

Program should load and be automatically executed. Congratulations, you got yourself working bootloader!

**MORE INFO COMING SOON**

## Building software

General rule is simple: `make` should be sufficient for all the build/installation. If you want to use this software under Windows, you have basically two options: Windows Subsystem for Linux (works, with the exception of minipro upload operation) or some sort of MinGW/Cygwin. If you have better idea on how to do it, and can provide Pull Request - I will be more than happy to have something better.

Beside `make` the following tools are used:

* `cc65` compiler, available [here](https://github.com/cc65/cc65),
* `minipro` software to upload ROM image to EEPROM, available [here](https://gitlab.com/DavidGriffith/minipro/),
* `hexdump` for testing the contents of the binaries,
* `md5` to generate binary checksum,
* `rm` and `mkdir` for folder/file manipulation,
* `picocom` for serial communication with the computer,
* `sz` for loadable module upload via `picocom`,
* `python` to run small utility written in Python to trim loadable modules and add start vector,
* `x6502` to run computer emulator (enables execution of generated binaries on PC), available [here](https://github.com/dbuchwald/x6502) - **please note: this one is optional, still in development, and lagging behind the actual computer.** Simple programs can be executed, but anything more complex than LCD operation (including ACIA/keyboard) don't work just yet.

The following `make` targets are to be used for building software:

* `all` - build the project,
* `clean` - delete all temporary files,
* `test` - dump the contents and checksum of generated binary file,
* `install` - upload generated binary to AT28C256 chip using `minipro` tool,
* `terminal` - connect to the 6502 computer using serial port, please note - currently uses my own device ID as visible under MacOS and most likely needs to be adapted to your build/OS,
* `emu` - run the generated binary in system emulator. Again: suitable for simple programs, more complex ones are not yet supported. So far I needed it only for simple debugging and that's why it is so limited.

Beside the targets, there are two very important build flags:

* `ADDRESS_MODE` - with acceptable values `basic` and `ext` (the latter being default if omitted) that drives target addressing model. To build for Ben Eater's machine, use `basic` mode; for my build, use `ext` mode. If you want to support your own model, create additional configuration file, as explained in common sources section below,
* `FASTCLOCK` - with acceptable values `0` and `1` (the latter being default if omitted) enables build time selection of runtime clock variant. Basically, certain operations (like LCD initialization) require delays when working at 1MHz clock speed, and the delays are implemented as dead loops. Each 1ms delay translates to 1000 clock cycles. Now, if you want to run the code with external clock module or in emulator, it will take literally hours to clock through single delay loop. To mitigate this issue and prevent necessity to change code each time, this flag was added. **Please note: to enable detection of possible stack related issues in code using delay operations, these routines are not fully disabled, jump to subroutine still happens, it just results in immediate return.**

Build examples:

```shell
make ADDRESS_MODE=basic FASTCLOCK=0 clean all test install
```

This will build sources with Ben's addressing scheme (16K RAM, 32K ROM, VIA at 0x6000), with support for slow clocking - any delay routines will be skipped. First, all the binaries will be removed, then built from scratch, hexdump of the resulting binary will be displayed and the binary uploaded to the EEPROM, assuming it's connected via minipro-compatible programmer.

```shell
make FASTCLOCK=1 all test
```

This command will rebuild only modified modules with support for my own addressing scheme (32K RAM, 24K ROM, VIA at 0x9000) and suitable for 1MHz execution - all delays will be enabled.

## Detailed description of modules in `Software` folder

There are quite many programs in the `Software` folder, making the navigation a bit difficult. This section should support you in navigating provided software library.

### ROM images in `rom` folder

In the `rom` folder you will find the following ROM images:

* `01_nop_fill` - simplest possible program, composed of 32K of NOP (0xea) instructions. The source itself seems empty, because default fill is defined in firmware configuration files (`common/firmware.basic.cfg` and `common/firmware.ext.cfg`),
* `02_nop_fffc` - extension of the above program by adding `VECTORS` segment, containing start address for 6502. Address of the `init` label depends on the firmware configuration used,
* `03_first_code` - very simple program that actually executes some code, but there is no effect to be observed,
* `04_blink_s` - first example of a program interfacing with external world, using VIA2 to drive LEDs, as in Ben's videos,
* `05_knight_rider` - modification of the previous one to achieve classic effect,
* `06_lcd_test` - modified version of Ben Eater's first LCD program. Modification involves using loops, but runs without RAM, only ROM is used. This program will work only on slow clock (not 1MHz), and will not work with onboard LCD connector. To execute this one, you need to connect LCD via breadboard to VIA2 connectors on the PCB. When compiled for Ben Eater's build (with `ADDRESS_MODE = basic`) it will work out of the box,
* `07_mem_test` - modification of the previous one, testing RAM module usage - message contents are copied first from ROM to RAM and only then displayed on the LCD,
* `08_stack_test` - modification of the previous one, but this time stack is utilized for JSR/RTS operation showcase,
* `09_serial_test` - simplest possible ACIA/serial testing program, using blocking send/receive operation to send simple message in response to each input on serial terminal,
* `10_blink_c` - modification of `04_blink_s`, but mixing low-level ASM code for hardware handling and C code for "business logic", shows how to use software stack to write code in C,
* `11_int_test` - illustration how to use VIA1 clock timer interrupt - basically displays text on LCD attached to VIA2 while changing LED (connected to VIA2 PA0) state each 50 cycles. Obviously needs to be executed in slow clock mode,
* `12_handshake_test` - very simple program that shows how to use CA1/CA2 hardware handshake operation with keyboard controller, will print on the LCD screen (connected to onboard LCD connector in 4-bit mode!) keys pressed on the attached PS/2 keyboard. Requires 1MHz clock for smooth operation, and is not compatible with Ben's build,
* `13_4bit_lcd` - testing program for 4-bit LCD interface, hence not compatible with Ben's build,
* `14_irq_test` - small program to run with slow clock showing operation of the VIA1 timer interrupt,
* `15_serial_irq` - interrupt-driven serial communication (both RX and TX), sends static message in response to each input from serial terminal,
* `16_delay_test` - testing program for improved 4-bit LCD library for onboard port, using functions like line wrap and vertical screen scrolling, not compatible with Ben's build because of the 4-bit interface,
* `17_blink_test` - another blink program, but this one uses common library functions to drive onboard LCD. Can be adapted to work with Ben's build, you just need to connect the LCD to PB0,
* `18_core_program` - test program used to verify operation of aggregated system init operation, uses onboard LCD to present contents of RX/TX buffer pointers (used in debugging of serial connection),
* `19_keyboard_test` - more complex program presenting integration with onboard keyboard controller, with IRQ driven data transmission, hardware state change detection and pretty interface on the onboard LCD port,
* `20_convert_test` - small testing program to test hex conversion function, aimed at x6502 emulator execution,
* `21_serial_load_test` - attempt to implement testing program for high serial load, counting incoming characters,
* `22_modem_test` - barebone modem testing application, sort of bootloader without user interface,
* `23_blink_test` - copy of `load/01_blink_test` to show how simple `makefile` change can be used to build the same source either as ROM image or bootloader-compatible loadable module,
* `minimal_bootloader` - simplest possible bootloader application that can be used to simplify software development thanks to making ROM flashing unnecessary for each code change.

The following table summarizes compatibility of each program with different versions of the 6502 computers:

|Program                  |Ben Eater's build execution notes|This build execution notes                                         |
|-------------------------|---------------------------------|-------------------------------------------------------------------|
|`rom/01_nop_fill`        |Works out of the box             |Works out of the box, slow clock and bus analyzer recommended      |
|`rom/02_nop_fffc`        |Build with ADDRESS_MODE=basic    |Works out of the box, slow clock and bus analyzer recommended      |
|`rom/03_first_code`      |Build with ADDRESS_MODE=basic    |Works out of the box, slow clock and bus analyzer recommended      |
|`rom/04_blink_s`         |Build with ADDRESS_MODE=basic    |Works out of the box, attach LEDs to VIA2, needs slow clock        |
|`rom/05_knight_rider`    |Build with ADDRESS_MODE=basic    |Works out of the box, attach LEDs to VIA2, needs slow clock        |
|`rom/06_lcd_test`        |Build with ADDRESS_MODE=basic    |Works out of the box, attach LCD to VIA2, needs slow clock         |
|`rom/07_mem_test`        |Build with ADDRESS_MODE=basic    |Works out of the box, attach LCD to VIA2, needs slow clock         |
|`rom/08_stack_test`      |Build with ADDRESS_MODE=basic    |Works out of the box, attach LCD to VIA2, needs slow clock         |
|`rom/09_serial_test`     |ACIA chip needs to be added      |Works out of the box with R6551, WDC65C51 needs slow clock         |
|`rom/10_blink_c`         |Build with ADDRESS_MODE=basic    |Works out of the box, attach LEDs to VIA2, needs slow clock        |
|`rom/11_int_test`        |Build with ADDRESS_MODE=basic    |Works out of the box, attach LCD and LED to VIA2, needs slow clock |
|`rom/12_handshake_test`  |Not supported                    |Works out of the box, 1MHz clock recommended, onboard LCD port     |
|`rom/13_4bit_lcd`        |Not supported                    |Works out of the box, onboard LCD port                             |
|`rom/14_irq_test`        |Build with ADDRESS_MODE=basic    |Works out of the box, slow clock and bus analyzer recommended      |
|`rom/15_serial_irq`      |ACIA chip needs to be added      |Works out of the box with R6551, 1MHz recommended                  |
|`rom/16_delay_test`      |Not supported                    |Works out of the box, 1MHz clock recommended, onboard LCD port     |
|`rom/17_blink_test`      |ADDRESS_MODE=basic, LED on PB0   |Works out of the box                                               |
|`rom/18_core_program`    |Not supported                    |Works out of the box                                               |
|`rom/19_keyboard_test`   |Not supported                    |Works out of the box                                               |
|`rom/20_convert_test`    |Build with ADDRESS_MODE=basic    |Works out of the box, slow clock and bus analyzer recommended      |
|`rom/21_serial_load_test`|Not supported                    |Works out of the box                                               |
|`rom/22_modem_test`      |ACIA chip needs to be added      |Works out of the box                                               |
|`rom/23_blink_test`      |ADDRESS_MODE=basic, LED on PB0   |Works out of the box                                               |
|`rom/minimal_bootloader` |Not supported                    |Works out of the box                                               |

### Loadable programs in `load` folder

All the programs in the `load` folder are to be uploaded to the 6502 computer over serial port with XMODEM protocol and require ROM to be flashed with software capable of receiving them. Currently this is `rom/22_modem_test` and `rom/minimal_bootloader`. Following list describes them in more detail:

* `load/01_blink_test` - simple program that blinks onboard LED 10 times, provided to illustrate loadable module build process. This one has been copied to `rom/23_blink_test` to illustrate differences between the two models,
* `load/02_hello_world` - "Hello World" example in a loadable module version,
* `load/03_string_test` - small program written to test string handling library functions,
* `load/04_blink_large` - presents different model of linking loadable code (with included common modules).

As for software compatibility - all the loadable modules require bootloader, and this one, in turn, requires ACIA for operation, so by design these are not compatible with vanilla Ben Eater's build.

### Loadable modules explained

There is one thing important to consider when working with loadable modules. The idea behind them is to have the possibility to run the same code from RAM and ROM, preferably preserving the former if possible and reducing the loadable file size. The idea is to be able to execute common functions stored in ROM from the code running in RAM.

There are two examples in the code repository demonstrating alternative approaches. Naive one, that assumes that all the code to be executed is included in the loadable module is available in `load/04_blink_large` folder. When built, you will notice it contains all the functions required: `_blink_init`, `_blink_led` and `_delay_ms` compiled and bundled. The only difference between this program and `rom/23_blink_test` is the value of `BUILD_TYPE=` flag in `makefile` resulting in different addressing model being used for code storage.

Now, the second example, provided in `load/01_blink_test` is much more interesting. The binary file is smaller (admittably, not by much, due to inclusion of the loadlib vector array!), but none of the code of the common functions (`_blink_init`, `_blink_led` and `_delay_ms`) is actually bundled in the binary. All these references are provided by stub functions, defined in `common/source/loadlib.s` - each of these functions really contains jump to vector defined in dedicated vector range in ROM (0xf800-0xfff9). This indirection layer enables updates to ROM without needing to recompile all the loadable modules. The only requirement is to keep the order of the calls intact and adding new functions to `common/source/syscalls.s` at the end so to keep previously defined addresses unchanged.

Defining new shared function requires the following:

* implementation of the code in `common/source` folder,
* implementation of the interface include in `common/include` folder,
* adding this new function to `common/source/syscalls.s` module,
* adding stub function to `common/source/loadlib.s` module,
* adding new objects to `rom/minimal_bootloader/makefile` and `rom/22_modem_test/makefile`.

The list above should help you understand how this code reusability has been achieved.