# RH850 CAN FD device Boot

Boot firmware for R7F701684 (RH850/F1KM-S1,1MiB Code Flash).
This project pairs with `Sample_Project_RH850_S1_Boot_loader_CAN_FD_app`.

## Board and layout

| Item | Configuration |
| --- | --- |
| Clock | X1 16MHz, CPU120MHz, CAN communication clock40MHz |
| EVM CAN | CAN1 P10_7/P10_6;CAN4 P0_10/P0_9;500kbit/s nominal,2Mbit/s data,BRS |
| CAN IDs | Standard control700/701;64-byte blocks on raw720,FD64 |
| Boot / App | Boot00000..0FFFF;App10000..FFFFF |
| Metadata | App+100h,256bytes(core128,reserved128FF) |
| App EBASE / INTBP | 10200 /10400;512 normal interrupt entries |
| Flash | App erase32KiB;program256-byte aligned pages |
| RX / tick | FIFO1/4 shared EIINT23,per-port16-slot queues(15usable);OSTM0 EIINT84,1ms |
| Diagnostic UART | UART0 TX P0_2,115200;optional logic-level output |

## EVM ports and buffer allocation

The EVM target is R7F701684 (100 pins,1 MiB Code Flash),X1=16MHz.
Default enabled ports are MCU CAN1 and CAN4. Set each ENABLE_PORT_CAN0..5
flag to0 or1 in `can_port_config.h`;at least one must be1.
Each project owns its local `can_port_config.h`. Apply compatible port and
communication settings to the Boot and App that will be installed together.

| MCU port | TX / RX routing | EVM connector |
| --- | --- | --- |
| CAN1 | P10_7 Alt4 / P10_6 Alt4 | Board CAN0,CN6 |
| CAN4 | P0_10 Alt5 / P0_9 Alt5 | Board CAN1,CN8 |

The board connector labels are not MCU channel numbers.
BLDC V2:S7_6/5 ON;S7_4/2 ON and S7_3/1 OFF.
ASK V3:J20 1-2/3-4 for CAN1,5-6/7-8 for CAN4.
Use the board manual's S4 termination settings;leave S4_3/4 OFF for separate
buses. Connect only the bus under test to the host,then repeat on the other
bus. The Tool bridge-port selector selects a host port,not a device port.

Each enabled MCU channel owns RX FIFO[channel] and a16-slot RAM queue
(15usable),with independent counters and source-channel reply routing.
TX uses buffer0 local to that channel (status index channel*32).
There are no RX Message Buffers or common TX/RX FIFOs.
FD RX allocation is1216bytes per port including hardware message headers;
default two ports use2432bytes,all six7296bytes,within the21504-byte shared
RX/FIFO budget (hardware manual24.10.1.5). Classical uses320bytes per port.
CPU RAM queues are separate from CAN message RAM.
Disabled ports allocate neither queues nor active filters.
CAN0/2/3/5 require board-level routing and compatible external transceivers.

- [BLDC V2 manual](https://www.renesas.com/en/document/mat/rh850f1km-s1-bldc-starter-kit-v2-users-manual)
- [ASK V3 manual](https://www.renesas.com/en/document/mat/rh850f1km-s1-starter-kit-v3-user-manual-hardware)

## Source ownership

| Source | Purpose |
| --- | --- |
| `image_metadata_config.h` | Neutral format, MCU/Flash contract, IRQ ABI, Boot identity |
| `hal/image/image_metadata.c/.h` | Independent portable C90 metadata validation |
| `hal/image/image_metadata_profile.c` | Expected profile and compiled App manifest |
| `can_boot_config.h` | CAN protocol, IDs, mode/rate masks |
| `boot_app_handoff.h` | RAM cookie,AA/55 owner,ENTER/update requests |
| `hal/image/device_image.c/.h` | Image CRC,entry/EBASE/512-vector bounds and INFO |
| `hal/isp/boot_engine.c/.h` | Staging,version policy,blocks,replay,ABORT/END/VERIFY |
| `hal/flash/device_flash.c`, `flash_geometry.c` | FCL and strict App-only geometry |
| `hal/device/device_read.c`, `device_cpu.asm` | RAM-only unknown-Flash read and CPU handoff |
| `hal/can/device_can.c` | Device protocol wrapper;replies on the received bus |
| `hal/can/rh850_can_ports.c/.h`, `can_port_config.h` | ISP compatibility API,port selection,source routing and completed-reply gate |
| `hal/can/can_service.c/.h`, `rs_canfd*` | Shared v4 frame queues,TX events,bounded RX service and validated register driver |
| `hal/can/can_config.c`, `can_board.c` | Exact ISP filters and R7F701684 EVM clock/pin/INTC adapter |
| `tools/device_build.ps1` | C-header generation,CS+ preparation,actual-MOT/map guards and packaging |

One bus claims metadata staging with RESET0A. Other buses may discover/read
INFO but cannot change that session or supply its raw blocks. A pre-START
30-second idle lease releases staging RAM. After START,ownership remains until
owner ABORT or reset;there is no concurrent Flash update or automatic takeover.

### CAN service and protocol ownership

The service and register-driver sources under `hal/can` match the
standalone MultiCHv4 core. Board/configuration adapters retain this EVM's
R7F701684/16MHz/CAN1+CAN4 settings and exact control/raw-data filters.
The standalone demo console,accept-all rules,startup and vectors are not used.

`CanPorts_Read()` services foreground TX/status work,then reads complete RAM
frames round-robin;only EIINT23 drains hardware RX FIFOs. Replies use the frame's
source channel. `CanPorts_Send()` waits for its matching `CAN_TX_DONE` event;
queue admission alone cannot trigger END/ENTER reset. TX timeout requests one
abort and waits for confirmation. Unconfirmed TX latches that bus's TX fault;
no automatic recovery or protocol replay follows. Other buses remain independent.

The v4 foreground APIs provide channel stop/recover and tag cancellation for
custom application code,not new Tool/wire commands. Use them only after settling
the protocol transaction and saving diagnostics. They do not release the Boot
update owner or bypass its Flash/metadata/CRC/version rules. Hardware error flags
remain readable;status queries and CLEAR_STATS do not clear them.

START validates the sealed metadata and installed-App version floor before
any erase. Erase occurs on the first accepted CAN block of each32KiB region.
Four64-byte blocks form one256-byte Flash page. ACK means session acceptance;
a partially filled page remains in RAM. Replaying the last block never writes
Flash again. END requires the full image CRC and vector contract before reset.
The Flash execution chain runs from FCL RAM with interrupts masked.

Valid App plus ENTER permits30seconds CAN inactivity before launch,only before
an accepted START. Invalid App stays in Boot. ABORT does not restore erased data.
Normal updates enforce same/newer only when the valid installed App enables
version control. FORCE_START bypasses ordering for that transaction only.

## Verification and guide

Clean CC-RH builds,actual linked metadata/vector/RAM guards and actual C90
engine/production-PC models are provided. Hardware CAN,FCL,ECC/reset/launch and
power interruption tests are NOT RUN. The board transceiver/timing needs
physical verification. Compiler license and alignment warnings are recorded
in the engineering build log rather than suppressed.

- [Engineering and operation guide](https://released.github.io/slide_boot_loader_for_RL78/#rh850-can-device)
- [Tool operation](https://released.github.io/slide_simple_renesas_isp_tool/#tool-operation)
- [Hardware manual](https://www.renesas.com/en/document/mah/rh850f1km-rh850f1kh-users-manual-hardware)
- [Flash hardware interface](https://www.renesas.com/en/document/mah/rh850f1k-users-manual-flash-hardware-interface)

## User code entry

`custom_func.c` provides `hardware_init()` for initialization and `loop()` for
foreground service. Add application initialization and foreground work through
these callbacks, retaining the existing communication service calls.
The selected main/startup adapter owns entry and handoff. Generated peripheral
files remain under `src/`; retain the project's MTPJ source selection when using
Code Generator or Smart Configurator, then rebuild to validate the image and vectors.

## Project layout and standalone build

This project includes its own source files, configuration headers and build tools.
A sibling Boot, App or Transfer checkout is not required to compile it.
Keep the project directory name aligned with its `.mtpj` filename; remove a ZIP download suffix when necessary.
Build with CS+ Rebuild, or run the local PowerShell entry:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\build.ps1
```

| Location | Responsibility |
| --- | --- |
| `custom_func.c` | Project initialization in `hardware_init()` and recurring work in `loop()`; runtime orchestration stays here |
| `hal/can` | CAN service, controller driver and CAN adapter, when used by this project |
| `hal/device` | Device, clock, peripheral and CPU handoff APIs |
| `hal/image` | Metadata parser/profile, image validation and software CRC |
| `hal/flash` | Flash access/geometry APIs, when required by the selected image |
| `hal/isp` | Update/Transfer protocol and transport processing APIs |
| Root configuration headers | Local MCU/Flash, handoff and communication contract |
| `tools` and root PowerShell helpers | Local metadata/IRQ generation and linked-image checks |
| `src` | Code Generator or Smart Configurator peripheral sources |

Only the sources registered in this project's CS+ configuration are linked.
Generated main/vector implementations must not become a second active entry.
Retain the `custom_func.c` callbacks and the selected startup adapter after code generation.

Boot and App each carry a local contract. Keep their MCU, Flash layout, image format,
IRQ ABI, handoff and communication settings compatible when installing a pair.
App releases are edited in `app_image_config.h` and `app_irq_handlers.h` in the App project.
Rebuild the selected App to regenerate its metadata, vector table, update BIN and timing BINs.
The App build does not read or rewrite a Boot checkout.

## Diagnostic text output

The project-local `retarget_config.h` selects the formatter at compile time:

```c
#define TINY_PRINTF_FULL 1 /* 1: Full C99-format mode; 0: Current mode. */
```

Rebuild after changing the mode. Full is the default and supports C99 integer,
string/character, pointer, `f/F/e/E/g/G/a/A`, `n`, `hh/h/l/ll/j/z/t/L`, flags,
numeric/dynamic width and precision. Current retains the existing format subset,
finite rounding and attempted-character counts. Both retain streamed precision
and the project's finite UART0 wait; ISP data uses its independent interface.

```c
#include "retarget.h"
/* Foreground, after UART0 initialization. Use types matching each format. */
tiny_printf("counter=%lu status=0x%08lX\r\n", counter, status);
tiny_printf("voltage=%*.*f size=%zu\r\n", 12, 3, voltage, size);
```

Full uses fixed C locale and nearest/ties-to-even rounding. Its exact decimal
conversion does not delegate to the vendor `printf`. Double/long-double precision
follows the selected CC-RH `-Xdbl_size` setting (both types must match); extra
printed decimals do not add floating-point precision. `%lc`/`%ls` accept C-locale
ASCII wide characters; unrepresentable characters return a negative result.
Ordinary `%s` copies encoded bytes. `%p` uses hexadecimal `0x...`, including
`0x0` for null; pointer notation is implementation-defined by the C standard.

Full returns the count accepted by UART, or a negative result on format/count/
encoding errors or UART timeout. This is not receiver/wire acknowledgement.
`SendChar()` returns the accepted unsigned byte or -1 on its existing wait bound.
Current mode ignores the hook error and retains attempted-output counts.
Format in foreground code, not in an ISR or Flash P/E path.

Format examples and the mode table are described in the
[RH850 Log guide](https://released.github.io/slide_boot_loader_for_RH850/app-development.html#rh850-runtime-log).
