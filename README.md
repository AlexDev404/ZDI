# ZDI
eZ80 ZDI adapter for debugging/programming

From: http://hsl.wz.cz:8080/zdi.htm

Long time ago I used eZ80 processors in my job. Once I got an idea to prepare [CP/M system](http://hsl.wz.cz:8080/cpmez80.htm) on top of FAT16/32 driver running on this CPU. After very long time I wanted to return a bit to this project but I didn't have access to debugger adapter (USB SmartCable) anymore. Fortunately, Zilog provides description for ZDI debugger interface in datasheets (PS0066, PS0130, PS0153 and PS0192), so it seemed to me as good point to start work on my open-sourced ZDI adapter.

First version was based on ATmega8, which was had soon small memory for all functions added. Fortunately, it was easy to replace it with ATmega328 and continue working... Now, I have working almost everything I wanted. So here follows short description...

ZDI adapter communicates with PC through serial port. It also provides serial port signals to user application. Serial port signal RTS is used to switch between ZDI function and serial port connected to user application. Bootloader utility [MCUload](http://hsl.wz.cz:8080/MCUload.htm) has support for this switch.

ZDI adapter, when switched to ZDI mode, controls both ZCL and ZDA signal. These signals are software controlled, so limited clock speed can be achieved. I have tested it with eZ80F92 running at 18.432MHz. It is possible that slower or faster system would require changed ZCL speed.

Currently, eZ80 system is connected on these pins:

- PC0 = /RESET
- PC1 = ZCL
- PC2 = ZDA
- PD2 = RxD (input from application)
- PD5 = TxD (output to application)

PC is connected through serial port signals:

- PD0 = RxD (input from PC)
- PD1 = TxD (output to PC)
- PD3 = DTR (/RST signal from PC)
- PD4 = RTS (BOOT signal from PC)

When ZDI adapter is switched to user application mode (RTS signal is 1), serial communication from PC is copied to application pins and also back from pplication to PC. So both PC and application signals have to be on the same port - here PORTD because this port has UART function. After some experience with ZDI adapter I would now suggest to move also signal /RESET from PC0 to free pin PD6 or PD7 to allow easier signal copying to control user application RESET signal.

ZDI adapter has simple text command interface. Here is list of available commands:

- **Basic commands**
- HELP
- VER
- **Low-level commands**
- ZDIR
- ZDIW
- **Run-control commands**
- RST
- RUN
- STOP
- BRK
- STEP
- EXEC
- **State-control commands**
- ADL
- MADL
- EXX
- **Debugging commands**
- REG
- IN
- OUT
- MAP
- MEMR
- MEMW
- DIS
- STACK
- SPS
- SPL
- **Special commands**
- MODE

All values are entered as hexadecimal. Eg. value '10' means 0x10 or 16 (decimal); 100 means 256 (decimal)... There is no way to use another base then hexadecimal.

#### HELP

Prints list of available commands.

#### VER

Prints firmware version and connected eZ80 chip version.

```
>ver
ZDI v0.9.282
built Mar  6 2016,18:35:14
Found: eZ80F92/93 rev AA
>
```

#### ZDIR, ZDIW

These two commands are basics of all communication. Both commands have address as first parameter. ZDI registers can only be read or written. Full list of registers is in processor datasheet, eg. PS0153 for eZ80F92/93 on Zilog's web.

```
ZDIR <addr> [<count>]`
`ZDIR <addr> <byte> [<byte> [<byte> [...]]]
```

For example, ZDI_STAT register can be read:

```
>zdir 3
ZDI[3]=0
>
```

eZ80F92 identification bytes can be read:

```
>zdir 0 3
ZDI[3]=7 0 AA
>
```

Values 7 and 0 are identification of eZ80F92 chip, AA is revision of the chip.

Directly control run/stop mode:

```
>zdiw 10 80
Break!
AF =000002	BC =000010	HL'=000208	IY =000020
SP =00F8BA
000038:	RST	$38
>
```

#### RUN, STOP

These commands are used to stop or run CPU execution. They also control breakpoints if used.

#### RST

This command simply resets CPU. The result depends whether CPU is running or stopped. If CPU is running, RST makes just single reset pulse. If CPU is stopped, it has to send STOP command in the right time, so series of RST pulses is used to find the right time automatically. After right timing is found, it is reused for other RST commands (until ZDI adapter gets reset).

#### BRK

Breakpoint syntax is more complex:

```
BRK[0..3] [<addr> [1]]
```

This command allows to print all breakpoints used, when entered without any parameter. All four breakpoints can printed/controlled separately entering breakpoint number. When removing breakpoint, - is used in place of address. If breakpoint is added without breakpoint number, free breakpoint will be searched. Breakpoint can be set for whole page (256 bytes) - additional parameter has to be used.



#### STEP

Single stepping through code can be made entering command STEP or just by pressing Enter on beginning of empty line. Changed registers are printed between steps.

```
>stop
Break!
AF =004200	BC =00003E	DE =0000B4	HL =B7E300
IX =B7FF1B	IY =000000	SP =B7FF1B	SP'=000000
00054E:	LD	A,($B7E3AC)
AF =00421C
000552:	LD	HL,$B7E3AD
HL =B7E3AD
000556:	CP	(HL)
000557:	JR	Z,$054E
00054E:	LD	A,($B7E3AC)
>
```

#### EXEC

This command allows to execute any opcode up to 5 bytes long. Eg. opcodes for `LD A,0x56`:

```
>exec 3e 56
AF =004256
>
```

#### ADL, MADL, EXX

These commands can be used to print or modify state of these special CPU bits and registers.

```
ADL [<value>]`
`MADL [<value>]`
`EXX
```

#### REG

This command prints all CPU registers (except I and R registers). In case of ADL mode (ADL set to 1), 24-bit values are printed and AF register has MBASE value in upper byte. Also, SP has value of SPL. When ADL is reset to 0, only 16-bit values are printed and upper byte is always 0. SP has value of SPS. Registers can be printed or changed entering it's names: AF, BC, DE, HL, SP, PC, IX, IY and alternate register SET AF', BC', DE', HL'. When value follows register name (eg. `HL' 123456`), register value is changed. Without value, current register value is printed.

#### IN, OUT

These commands allow peripheral control. Eg. `IN 9A` will read portB, `OUT 9E 41` will write data to portC, `OUT F5 B6 49` will unlock FLASH_KEY register to acces some Flash functions.

#### MAP

MAP will print memory mapping based on many registers: CSx_*, RAM_*, FLASH_*. Here is example of memory mapping when running in external RAM:

```
>map
Flash: disabled.
RAM: B7
CS0: disabled.
CS1: MEM 00-07 1ws
CS2: disabled.
CS3: IO 10-10 0ws
>
```

#### MEMR, MEMW

If memory needs to be read or written, these commands can be used.

```
MEMR [<addr> [<cnt> [<len>]]]`
`MEMW <addr> <byte> [<byte> [...]]
```

MEMR can read from current PC address or from entered address. Data read are printed in form of Intel-HEX records. Default byte count as well as line length is 32 bytes, i.e. 20 (in hexadecimal). When byte count is bigger than line length, more lines are printed.

MEMW can be used to easily write some data to memory. So many parameters are entered, so many bytes are written to memory.

#### DIS

This command provides disassembly either from current PC or from entered address, default one line only, but range can be also entered:

```
DIS [<addr_from> [<addr_to>]]
```

#### STACK, SPS, SPL

To print current stack, one of these commands can be used. Stack always prints current stack based on current ADL mode, while SPL or SPS print specific stack: either 24-bits or 16-bits.

By default, all these commands print 5 stack items, but the count can be changed by entering parameter. To print 10 (decimal) stack entries, `STACK A` can be entered.

#### MODE

This command is used internally by MCUload to switch between programming and verification mode. After selecting this mode, regular Intel-HEX lines are sent to ZDI. Data are written to CPU memory immediately. If received line has correct checksum, and in case of verification all data is verified, '*' is returned. Otherwise error code is returned back. Download utility has to wait always for '*'. If reception timeouts, there is an programming/verification error.

## Ideas for improvement

After long time spent developing ZDI adapter and using it I would now do some parts another way...

#### ZCL frequency

Original USB SmartCable probably uses FPGA or CPLD as ZDI interface. There is a table in doc, which ZCL frequency should be used for which system frequency. ATmega makes all ZDI communication i software, for internal 8MHz oscillator, ZCL frequency is about 1.3MHz. When it was below 1MHz, it didn't work and I had to find a way to optimize ZDI communication utility.

Nowadays, I wouldn't use FPGA or CLPD but high performance MCU - probably ARM Cortex-M3/M4 running on high enough frequency - to achieve 4MHz ZCL frequency for my system's clock 18.432MHz, or even 8MHz for top speed 50MHz eZ80 CPUs.

For multibyte transfers, it takes about 8us to send byte, so effective ZCL speed is 1MHz. But probably this parameter is not important for reliable communication, because when transfering Intel-HEX, bytes are sent whenever arrive - at 57.6kBd it is every 350us - so effective ZCL speed in this case would be much lower. But it works so I assume this as non-essential...

#### /RESET signal for eZ80

As suggested above, it would be convenient to have both /RESET output and /RST input pins on the same port to easily control this signal while switched in application mode.

#### Built-in disassembler

Disassembler is not yet fully finished. There are still some opcode combinations which are not correctly supported.

#### Strange behaviour

There are few things which are not mentioned in doc and I don't know if they are caused by misunderstanding of doc or are features (or bugs) of CPU.

Reading from memory using ZDI_RD_MEM always ends with PC being incremented by one more than read bytes. Probably, the read cycle is made at the end of previous byte transferred (i case of first byte at the end of address written). Ususally, PC is saved anyway before read is made. Other possibility is to read one byte less and the last byte read from ZDI_RD_L register.

When executing opcode, previous opcode is executed when writing the last opcode byte. My guess is that it is influenced by low ZCL clock. Fortunately, there is an workaround to execute wanted opcode and then to execute NOP, i.e. 00 opcode.

Another strange thing happens when erasing flash. This is done by executing OUT instruction. As soon as this instruction is being executed, CPU holds internally WAIT signal active and ZDI_ACTIVE bit is inactive for this time. So it is good idea to wait for this bit active after any opcode is executed, because I don't know if there are another operations leading to WAIT signal active for longer time...
