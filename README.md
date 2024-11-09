**VE816 Computer Docs**

This is the primary documentation to my new project, a computer system built off of a WDC 65816 microprocessor. The full list of features is here:

* A CPU running up to 20mhz
* Access to 16MB of removable RAM and code space
* Hardware seperation for code and data
* Four cartridge slots
* Four external bus slots
* A video card
* PS/2 keyboard and mouse controller
* UART serial connector
* Sound card
* Power-on reset

It is a fairly simple system, designed to mimic systems from the 80/90's. I also use some designs used by common x86 systems, such as my own external bus and segment seperation. The computer is built highly modularly, for ease of assembly. As well as having a 65816 CPU, it includes other chips from the 65xxx chipset, such as the 6522 and 6552.

#### Chipset

This system uses one WDC 65816 as the primary CPU, as well as two WDC 6522's. These two chips are used for managing the external bus channels as well as memory selectors, flags, and more. The primary chip is usually used for managing the regular interrupt lane, described later. The way to talk to the motherboard is through this. The second (secondary) chip is used for handling the other two lanes, called the fast interrupt lane. The primary VIA is connected through to the NMIB pin, which cannot be masked. The secondary one is wired to the IRQB pin. The reason why the regular interrupts cannot be masked is so it can operate as an extension of the CPU. After all, you wouldn't want half the CPU suddenly cut off! The secondary lanes can be used for less vital purposes, such as keyboard and mouse inputs.

As well as those, it has several embedded ROMs. The first one we will discuss is for the BIOS functions. When a cartridge needs to call an important function, such as reading a chunk of memory from the code segment, it calls a BIOS function. It could implement one itself, but that takes space. When calling, it jumps to a location placed in a ROM, storing code. When finished, it simply goes back to where it left off. There is also a 64K RAM chip used for storing the stack, as well as the zero-page for important variables. The zero-page is efficient to access in terms of clock cycles. There is also a decoder for the entire memory map, mostly done through another ROM. It takes the higher part of the address pins, and tells the CPU which chips should be active for certain parts of the memory map. As well as that, another set of logic gates manage whether the CPU should be using the code or data segment for a specific clock cycle.

Those two systems manage exactly which chip should be active and where. As well as those, there are eight slots to insert a 512K RAM chip. A decoder is used to moderate which one is active. Memory is detectable by simply writing a value and reading it back.

#### VIA Pins

The two regular interrupts are INT0, INT1 and fast interrupts are INT2, INT3.

VIA (Versatile Interface Adepter) is used to interface with the motherboard properly. Like stated before, it can control anything from the externel bus lanes to the memory system. You can access these through memory, as stated on the map. Anything and everything about them can be found on their datasheet, the WDC 6522. Each chip is composed of two pins we will use for interrupts, as well as two 8-bit ports. Each bit can be set to either input or output. The pins on the primary VIA are defined here:

*Primary VIA*


| Pin   | Description                                                                                     |
| ------- | ------------------------------------------------------------------------------------------------- |
| PA0   | (output) Read segment, not for writing. 0 for data and 1 for code.                              |
| PA1   | (output) Execution segment, which segment we are executing from. Useful for compiled code.      |
| PA2   | (input) Regular INT0 avalible. High if so, low if not. Turn off after done.                     |
| PA3   | (input) Regular INT1 avalible. High if so, low if not. Turn off after done.                     |
| PA4   | (output) Interrupt selection, multiplex with PB0-7 for the interrupt caller ID. 0=INT0, 1=INT1. |
| PA5   | (output) (pulse) INT0 completed, pulse LOW then high to finish.                                 |
| PA6   | (output) (pulse) INT1 completed, pulse LOW then high to finish.                                 |
| PA7   | (output) Block interrupts. Force smart devices to queue return interrupts, may cause issues.    |
| PB0-7 | (input) Multiplexed PCI interrupt caller ID, aka which device called an interrupt.              |
| CA2   | (input) INT0 In line, pulsed HIGH by a device to call. Also sets INT0 ava. asynchronously.      |
| CB2   | (input) INT1 In line, pulsed HIGH by a device to call. Also sets INT1 ava. asynchronously.      |

*Secondary VIA, same details as primary*


| Pin   | col3                                                                                       |
| ------- | -------------------------------------------------------------------------------------------- |
| PA0   | (input) Fast INT2 avalible.                                                                |
| PA1   | (input) Fast INT3 avalible.                                                                |
| PA2   | (output) Interrupt selection.                                                              |
| PA3   | (output) Block interrupts. Drop all new requests.                                          |
| PA4-7 | Reserved                                                                                   |
| PB0-7 | (input) Multiplexed fast PCI interrupt caller ID.                                          |
| CA2   | (input) INT2 In line, pulsed HIGH by a device to call. Also sets INT2 ava. asynchronously. |
| CB2   | (input) INT3 In line, pulsed HIGH by a device to call. Also sets INT3 ava. asynchronously. |

*Missing pins are self-explanatory and on the datasheet.*

#### Memory Map

*Data Segment:*


| Range               | Size | Description            |
| --------------------- | ------ | :----------------------- |
| 0x000000 - 0x0000FF | 256B | Zero-page              |
| 0x000100 - 0x00FFFF | 65KB | Stack                  |
| 0x010000 - 0x0100FF | 256B | Primary VIA/PCI device |
| 0x010100 - 0x0101FF | 256B | Secondary VIA          |
| 0x010200 - 0x3FFFFF | 4MB  | Extended Memory        |
| 0x400000 - 0xFFFFFF | 12MB | Cartridges (DATA)      |

**Note: the final 512KB memory chip is cut off by 0x10000 bytes*
*Cartridges*


| Range               | Size | Description |
| --------------------- | ------ | ------------- |
| 0x400000 - 0x6FFFFF | 3MB  | Cartridge A |
| 0x700000 - 0x9FFFFF | 3MB  | Cartridge B |
| 0xA00000 - 0xCFFFFF | 3MB  | Cartridge C |
| 0xD00000 - 0xFFFFFF | 3MB  | Cartridge D |
| *Code Segment*      |      |             |


| Range               | Size | Description        |
| --------------------- | ------ | -------------------- |
| 0x000000 - 0x0000FF | 256B | Zero-page          |
| 0x000100 - 0x00FFDF | 65KB | Firmware/BIOS      |
| 0x00FFE0 - 0x00FFFF | 32B  | IVT (see 65816)    |
| 0x010000 - 0x3FFFFF | 4MB  | Unused (ext. mem.) |
| 0x400000 - 0xFFFFFF | 12MB | Cartridges (CODE)  |

#### Memory managment

The memory system on this system is fairly straightforward. You traditionally have 16MB of space for data, as well as 16MB for code. The code segment is also treated as a sort of "hard-disk" system, as you can store things like constants and things that must be loaded. When using a cartridge, the code and data chips can be managed by that specific piece of hardware. Addresses going into a cartridge are automatically brought down to zero, and pins tell it to send data onto the bus, and from which segment.

During runtime, the CPU will automatically switch to the code segment. This is for things like fetching opcodes and the like. When you need to read data from the code segment, such as a message at boot, you need to flip a bit called the "segment selector". When the bit is high, you are reading from the code segment. When low you are reading from the regular data segment. Since you cannot write to the code segment, all writes during a high byte will default over to the data segment. This is useful when copying data. Variables needed during these can be used in the zero-page. It always goes to the data segment when dealing with zero-page, regardless of mode, and always goes to code when dealing with the IVT, hardcoded by the CPU.

The optimal way to read from data is in chunks, writing them bit by bit starting from some location in the data segment. Since writes default there, this is made easy. Reading from a PCI device while the selector is high makes no difference, as it goes to the PCI regardless. Unexpected behaviour can happen if you try to read while in code mode from an extended location or a VIA. Best to not do that.

WHen the VDA pin on the CPU is high, it is trying to access a valid data address. WHen high, we should usually select the data segment. When VPA is high instead, it is trying to fetch an opcode for the next instruction. If both are high, it is attempting to reach something in data such as the stack. It is best to give it a data segment. If none are high, it is doing something internally and it does not matter which one is selected.

If it has a valid data address but we are trying to read from the code segment, instead select the code segment. If we are writing, then still select the data segment. If the execution segment bit is high, then we need to select the data segment when we fetch any opcodes. Otherwise keep that at the code segment.

The lowest 65K pin is a sepecial case. Since we cannot pass the SEL bit due to the address limit, we must combine the first pin output to decide which chip. Addresses are preserved however. The SEL bit (segment select, not from VIA) is computed earlier, and if that is low we are selecting the code segment. Therefore we must select the BIOS chip at all times. When SEL is high, we need to read from the Zero-page/stack chip located in the same spot. The special case is reading the IVT for the CPU. Regardless of which segment we want, the BIOS chip must always be selected. To do this, if A5-A15 are all high, we disable the stack chip and enable the BIOS chip. This is because only inputs A8-A23 are passed into the ROM.

When using either VIA or the extended memory region, there is no code located there, so it defaults to the chip regardless of the segment. The segment is only defined in the low 65K and on cartridges. There are three ways to run code on this computer, either through a BIOS-only system, using a cartridge with code and data chips and boot into that, or upload code through a UART system, setting the execution bit and jumping there.

The execution bit is tricky. Since if you reset it the segment would instantly jump out of the code, it will wait exactly eight clock cycles before the change takes effect. This is so that the CPU has time to jump to that region of memory. For example if you load into extended memory and start from there, since it defaults to memory it begins regardless. After those eight cycles it will have fully switched. Going back follows the same prodecure. The best way is to simply pass it through an 8-bit shift-register to get the appropriate delay time.

*Address decoder pins (active high)*


| Pin | Description                                      |
| ----- | -------------------------------------------------- |
| 0   | Lowest 65K. Combine with SEL to find which chip. |
| 1   | Primary VIA Selected.                            |
| 2   | Secondary VIA Selected.                          |
| 3   | Extended memory Selected.                        |
| 4   | Cartridge A Selected.                            |
| 5   | Cartridge B Selected.                            |
| 6   | Cartridge C Selected.                            |
| 7   | Cartridge D Selected.                            |

*Cartridge connector*


| Pin  | Description            |
| ------ | ------------------------ |
| 0-7  | Data Bus               |
| 8-29 | Address Bus            |
| 30   | Cartridge Enable       |
| 31   | Segment Selector       |
| 32   | (R/WB) 0=WRITE, 1=READ |
| 33   | (PHI2) System Clock    |
| 34   | VCC                    |
| 35   | GND                    |

*Pin description*

The data bus is the typical CPU data bus leaving and entering the processor. The address bus is 4MB wide, however only 3MB of that space is usable. The cartridge enable pin is used to select which of the four cartridges should be transmiting or receiving data, and the segment selector is meant to select either the code (rom) or data (ram) chips within the cartridge body. RWB is simply the RWB pin coming off the CPU, which goes everywhere. PHI2 is the system clock which allows everything to be synchronized writing to the memory of the cartridge can be done by checking the RWB pin and the rising edge of the PHI2 pin. This also goes for general RAM chips like stack or extended memory. This gives a 36-pin cartridge connector.

#### PCI/External Bus

*Regular lanes*

This is the secret sauce that brings it all together. The PCI bus is the main part of the CPU which allows for its modularity. The first 16 PCI devices are taken up by the primary VIA, and sort of act like test devices the remaining 240 possible devices are avalible for use. Note that one plugged in piece of hardware may identify as multiple devices if needed.

The purpose of interrupts is simple: to let the CPU know that it has completed a command (regular) or to send a piece of information to the CPU (fast). Usually regular devices have a smaller CPU that can queue up a return, so that it can wait for the line to become open. In order for a regular interrupt to be fired, the device must wait for the specific line's avalibility bit to go low. It is a good idea for devices to not share lines, but if necessary it can be done. Once that interrupt line avalibility bit is off, it means that the CPU has acknowledges the interrupt device ID and has began. The device can then pulse the interrupt line high and then low. This will set the avalibility bit, as well as send an interrupt to the primary VIA using either CA2 or CB2.

If there are two interrupts handling at once, they can become nested like functions. Take care to preserve the state of the CPU. For example, if you want to send an instruction to draw a cricle on a graphics card asynchronously, you need to see when the gpu is finished. Then you will receive a finishing instruction, combined with some timing, before sending the next. When an interrupt is sent, the data bus will first change to the device ID that is required, before pulsing. That updates the registerm, which is then multiplexed into PB2. If the device is already selected for another operation, then it waits, as if there was another interrupt avalible.

The regular lanes go into the NMIB pin on the CPU. That tells the CPU what type of interrupt it is. If the interrupt block pin is high, no interrupts can be fired, and it appears as if all lanes are not avalible to send. That can cause a backup of needed interrupts and cause issues. A general way of doing it is a "one in, one out", meaning you know what you should receive.

A good practice when reading interrupts is to check both lanes, in case two were fired at once.

*Fast lanes*

In the fast interrupt lanes, things work a little differently. In these lanes, wildcard interrupts can be sent by something like a keyboard, mouse or a UART. You don't necessarily know when these will be fired, so they are less important. When the block bit is set, all interrupt are dropped instead of queued by a smart device, typically. They can however be turned *into* a smart device and used as such, their handling is just slightly different.

*Operating with PCI*

Reading/writing to PCI is far simpler than an interrupt. When you need to do one, you can simply use it as a memory address. For example, if you get an interrupt from a UART chip, you could read x bytes from the device ID, each subsequent read is the next byte. Writing is much the same, and it works pretty standard. A lot of trust is left to the specific devices, as if they don't behave it causes issues.

*PCI connector*


| Pin  | Description            |
| ------ | ------------------------ |
| 0-7  | PCI Device ID/Address  |
| 8-15 | PCI Data Bus           |
| 16   | (R/WB) 0=WRITE, 1=READ |
| 17   | (PHI2) System Clock    |
| 18   | PCI Enable             |
| 19   | INT0 Avalible          |
| 20   | INT1 Avalible          |
| 21   | INT2 Avalible          |
| 22   | INT3 Avalible          |
| 23   | INT0 Request           |
| 24   | INT1 Request           |
| 25   | INT2 Request           |
| 26   | INT3 Request           |
| 27   | VCC                    |
| 28   | GND                    |
| 29   | NC                     |

Similarly to the cartridge conenctor design, the PCI has a data and address bus. The Address bus acts as a device ID, and each device is responsible for determing if it is being called upon. The data bus is a standard data bus, though it first goes through an intermediary bus. This is so that it can send the ID to the correct register upon an interrupt, and it can be manually selected to send the data towards the CPU. RWB tells the device whether it is being read from or written to. Combined with the system clock you can synchronize write actions. The Enable pin is to simply tell the PCI bus that a device is being selected. INT0-3 avalible is broadcasting which interrupt lanes are avalible to be occupied. A smart device may choose to use either INT0 or INT1, however using INT2 or INT3 can be risky. This is because the CPU may be blocking those. The block pin is not transmitted directly, instead all avalible pins go high *only to the PCI connectors*, not the VIA pins. This is so that the CPU has time to process it all.

INT0-3 Are the main request lines, which are all pulled low. When a lane comes avalible and it's time to send an interrupt, the device will first put it's device ID on the PCI data bus, before pulsing the desired interruopt lane high and then low.
