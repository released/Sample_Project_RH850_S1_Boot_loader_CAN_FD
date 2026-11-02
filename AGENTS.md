# RH850 MCU CAN FD device

Read this project README before changes. Preserve Classical
and existing RL78/host projects. User scope is offline builds/models only.

- C90,block-start declarations,English source comments.
- common/can_v4 mirrors standalone v4 service/driver,not its board/startup/demo.
  ISP config/board/wrapper retain EVM684/16MHz/CAN1+CAN4 and exact filters.
  CanPorts_Read services TX/status but never polls IRQ-owned RX hardware.
  Reply success requires matching DONE;unconfirmed TX faults only that bus.
  No automatic restart/replay/error-flag clear or new control wire commands.
  Preserve protocol owner across any explicit service-level channel control.
- EVM R7F701684/100pins/1MiB/X1=16MHz,CPU120/CAN40. Boot64KiB,
  App capacity960KiB;Flash geometry38blocks. ECC startsFEDE0000.
- can_port_config.h numericENABLE_PORT_CAN0..5 defaultsCAN1/CAN4.
  Per-port FIFO[channel]/RAM queue/TX0,counters and source-bus replies.
  Shared RX IRQ23,bounded round-robin;no main-loop RX polling fallback.
  Keep the port core/config identical to both Boot projects and standalone
  Classical Transfer. Use unique filters/FIFOs and the checked RAM budget.
- RESET0A claims one staging bus;other buses have discovery/INFO only.
  Release on ownerABORT/reset or pre-START30s idle;preserve ever_started.
- Neutral metadata/profile/parser,CAN config and handoff stay separate.
  The neutral parser and C-header reader mirror the Tool golden helpers.
- App uses six-field app_image_config.h and independent app_irq_handlers.h;
  CS+ PowerShell pre/post hooks generate and verify actual linked outputs.
- 64KiB Boot/App10000/metadata10100/EBASE10200/INTBP10400/code11000;
  full512 entries,EIRET,AA/55 owner,at least512-byte Boot spare.
- FD500K/2M/BRS,64-byte raw720 andFF tail. Preserve complete mode/ID/DLC/length
  checks and actual IRQ-only RAM queue. No main-loop hardware RX polling.
- Erase32KiB on first accepted64B block,aggregate256B programming page,never
  reprogram on replay. ACK is session acceptance,not64B durability.
- FCL RAM-only P/E with DI;call-free copied RAM reader saves/restores VCI ECC.
  Guard all RAM,debugger,cookie and FCL copy bounds in actual map/assembly.
- Keep valid-installed-App-only version floor and transaction force0E.
  No persistent override or automatic repeat erase/RUN.
  rebuild or rewrite Boot images. Verify actual512 pointers after IRQ edits.
- Sync RH850 guide/Web/PDF and actual build/model evidence. Do not claim
  physical CAN2M/IRQ/FCL/ECC/reset/power acceptance from software models.

## User code entry

Use project-local custom_func.c hardware_init()/loop() callbacks. Preserve the
selected main/startup adapter and role-specific initialization/handoff ordering.
Keep generated main sources from becoming a second active entry. Preparation
templates and SC checks must preserve these bindings. Expand handwritten control
flow with braces and one statement per line; do not introduce transfer_main.c
or transfer_fd_main.c.

## Local source ownership

Each Boot/App/Transfer owns all sources, headers and build tools needed to compile.
No sibling checkout, absolute source path or parent-directory include is permitted.
Use hal/can, hal/device, hal/image, hal/flash and hal/isp by function; do not add
common or version-number directories. Root headers remain local editable contracts.
Project orchestration belongs in custom_func.c hardware_init()/loop(), not a
separate runtime entry source. Preserve role-specific ordering and SC bindings.
Keep local Boot/App contracts compatible; App-only builds cannot modify Boot.

## Diagnostic console

Use the existing project-local retarget.c/.h tiny_printf directly for foreground
text and full original format support. retarget.c retains C99/long long and
float support; other handwritten modules retain C90 style. Do not strip formats.
SendChar is its sole bounded UART0 output hook.
Keep protocol data separate; no printf/formatting in ISR or Flash P/E paths.
Do not reintroduce Device_Log/Device_LogHex/UartChar or manual numeric printers.
