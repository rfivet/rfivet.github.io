# Appendix A: Factory written content

	$ stm32flash -r - -S 0x1FFFF7a0:96 COM6 2>/dev/null | hexdump -C
	00000000  ff ff ff ff 31 00 10 00  ff ff ff ff 14 80 20 00  |....1......... .|
	00000010  13 57 33 52 31 37 31 20  ce 06 f5 05 f0 ff ff ff  |.W3R171 ........|
	00000020  ff ff 27 05 ff ff ff ff  fc ff ff ff 10 00 ff ff  |................|
	00000030  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
	00000040  ff ff ff ff ff ff ff ff  f3 ff ff ff ff ff ff ff  |................|
	00000050  ff ff ff ff ff ff ff ff  68 97 64 9b 3c c3 3f c0  |........h.d.<.?.|
	00000060

| Location   | Content         | Size | Format                   | Reference   |
|------------| :-------------: | :--: | ------------------------ | ----------- |
| 0x1FFFF7A0 | Protocol 3 Ver. | 2    | xFFFF => unsupported     |             |
| 0x1FFFF7A2 | Protocol 2 Ver. | 2    | xFFFF => unsupported     |             |
| 0x1FFFF7A4 | Protocol 1 Ver. | 2    | x31 => 3.1               | AN2606 4.2  |
| 0x1FFFF7A6 | Bootloader ID   | 2    | x10 => 1 USART, 1st ver. | AN2606 4.2  |
| 0x1FFFF7AC | UID.X           | 2    | bit signed, x8014 => -20 |             |
| 0x1FFFF7AE | UID.Y           | 2    |                          |             |
| 0x1FFFF7B0 | UID.WAF_NUM     | 1    | unsigned                 |             |
| 0x1FFFF7B1 | UID.LOT_NUM     | 7    | ASCII                    |             |
| 0x1FFFF7B8 | TS_CAL1         | 2    |                          | DS9773 3.10 |
| 0x1FFFF7BA | VREFINT_CAL     | 2    |                          | DS9773 3.10 |
| 0x1FFFF7C2 | TS_CAL2         | 2    |                          |             |
| 0x1FFFF7CC | Flash size (KB) | 2    | x10 => 16 KB             | RM0360 27.1 |


| BootID | X     | Y   | Wafer | Lot       | TS_CAL1 | VREFINT_CAL | TS_CAL2 | Flash | TBD     |
| :----: | ----: |-----|-------|-----------| :-----: | :---------: | :-----: | :---: | :------ |
| x10    | x8014 | x20 | x13   | ‘W3R171 ‘ | x6CE    | x5F5        | x527    | 16    | hd<?    |
| x10    | x8011 | x49 | x0E   | ‘W3U795 ‘ | x6D1    | x5F0        | x523    | 16    | hbF?    |
| x10    | x8015 | x21 | x13   | ‘W4A195 ‘ | x6DA    | x5EE        | x52A    | 16    | h^D?    |
| x10    | x1C   | x3A | x12   | ‘W4A820 ‘ | x6B9    | x5F6        | x511    | 16    | hR;?    |
| x10    | x09   | x4D | x0B   | ‘W4C593 ‘ | x6E6    | x5F4        | x53C    | 16    | haG?    |
| x10    | x1D   | x41 | x12   | ‘W4R342 ‘ | x6E4    | x5F1        | x535    | 16    | hZJ?    |
| x10    | x10   | x17 | x05   | ‘WAA390 ‘ | x6E9    | x5F8        | x523    | 16    | hY=?    |
| x10    | x800A | x41 | x08   | ‘QMY687 ‘ | x6E6    | x5F1        | x53E    | 16    | hZE?    |
| x21    | x4D   | x47 | x0C   | ‘QMT476 ‘ | x6D8    | x5F3        | x52D    | 64    | hlR\xBF |
| x21    | x36   | x47 | x0A   | ‘QRW813 ‘ | x6DF    | x5F3        | x539    | 64    | hhF\xFF |
| x21    | x41   | x1E | x12   | ‘SNG712 ‘ | x6EB    | x5F0        | x526    | 64    | hjS\xFF |

___
© 2020-2021 Renaud Fivet
