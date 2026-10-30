# DMA SPI

## Overview

This Software Building Block (SBB) demonstrates SPI communication using DMA-supported SPI channels on Renesas RA MCUs. The supplied reference application configures one SPI channel as Master and another as Slave, then transfers data between them using the configured DMAC support. User commands and data are entered through a serial terminal.

This document is limited to the following reference boards:

- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

> **Board scope note:** The supplied `readme.txt` does not list FPB-RA0L1 or FPB-RA2E3 as supported boards for this SBB, and it provides no SPI wiring or configuration for them. They are therefore not documented as supported reference configurations here.

## Use case demonstrated by this SBB

The SBB demonstrates two SPI transfer patterns between a Master and Slave SPI instance:

1. **Write and Read**: the Master transmits data to the Slave using separate write/read operations, after which the Slave transmits the received data back to the Master.
2. **WriteRead**: the Master and Slave simultaneously transmit and receive data using the SPI WriteRead API.

The reference application verifies received data against transmitted data and reports success or an error through the terminal.

The supplied example uses **32-bit SPI transfers**. User data is padded to a 4-byte boundary before the number of SPI transfers is calculated. The README states that each four bytes of user data are transferred in one SPI transmission.

## Hardware setup

The supplied README specifies:

- 1 x supported Renesas RA board
- 1 x appropriate USB cable for programming and debugging
- 3 x jumper wires

Connect the board USB debug port to the host PC and make the following Master/Slave SPI signal connections.

### FPB-RA4E2

| Signal | Connection |
|---|---|
| MISO | P110 (J4:3) to P100 (J4:16) |
| MOSI | P109 (J4:2) to P101 (J4:15) |
| RSPCK | P111 (J4:4) to P102 (J4:14) |

### FPB-RA6E2

| Signal | Connection |
|---|---|
| MISO | P110 (J4:3) to P100 (J4:16) |
| MOSI | P109 (J4:2) to P101 (J4:15) |
| RSPCK | P111 (J4:4) to P102 (J4:14) |

### FPB-RA8E1

| Signal | Connection |
|---|---|
| MISO | P609 (J3:13) to P410 (J2:12) |
| MOSI | P611 (J3:12) to P411 (J2:9) |
| RSPCK | P610 (J3:14) to P412 (J2:10) |

## SBB core code

The reusable SPI-facing SBB layer consists of:

- `spi_ep.c`
- `spi_ep.h`

The core code:

- Opens the configured SPI Master instance.
- Opens the configured SPI Slave instance.
- Provides wrappers for SPI Write, Read, and WriteRead operations.
- Closes both SPI instances.
- Handles Master and Slave callbacks.
- Records transfer-complete or transfer-aborted events for each SPI instance.
- Selects `r_spi_b` or `r_spi` at compile time from BSP peripheral feature macros.

The actual DMA configuration is not implemented directly in `spi_ep.c`; the README describes the SPI channels as DMA-supported and states that SPI data is transmitted and received via the DMAC module. The generated SPI/FSP configuration associated with the Master and Slave instances therefore forms part of the reference setup.

## Reusable SBB code versus demo/reference application

### Reusable SBB code

`spi_ep.c` and `spi_ep.h` provide the reusable SPI wrapper and callback layer. These files expose the primitive operations used by the application:

- initialize the Master and Slave
- write
- read
- simultaneously write and read
- deinitialize
- report transfer completion/abort through event flags

### Demo/reference application code

`dma_spi_example_app.c` and `dma_spi_example_app.h` implement the interactive demonstration. They are responsible for:

- Initializing the UART console.
- Displaying the example and FSP version information.
- Presenting the terminal menu.
- Collecting Master and Slave input data.
- Padding Master input to a 4-byte boundary.
- Selecting 32-bit SPI transfer width.
- Sequencing Master/Slave calls for the Write and Read demonstration.
- Sequencing Master/Slave calls for the simultaneous WriteRead demonstration.
- Waiting for both transfer-completion callbacks.
- Detecting timeout and aborted-transfer conditions.
- Comparing transmitted and received buffers.
- Displaying results and handling errors.

The reference application's entry point is:

```c
void dma_spi_example_app_run(void);
```

The source identifies this as the single function called by `hal_entry.c`.

## Public functions to look at first

### `spi_init()`

```c
fsp_err_t spi_init(void);
```

Opens both generated SPI instances using:

- `g_spi_master_ctrl` / `g_spi_master_cfg`
- `g_spi_slave_ctrl` / `g_spi_slave_cfg`

This is the starting point for understanding the SBB's Master/Slave configuration relationship.

### `spi_write()`

```c
fsp_err_t spi_write(spi_ctrl_t * const p_api_ctrl,
                    void const * p_src,
                    uint32_t const length,
                    spi_bit_width_t const bit_width);
```

Starts an SPI write on the Master or Slave instance supplied by the caller.

### `spi_read()`

```c
fsp_err_t spi_read(spi_ctrl_t * const p_api_ctrl,
                   void * p_dest,
                   uint32_t const length,
                   spi_bit_width_t const bit_width);
```

Starts an SPI read on the selected instance.

### `spi_write_read()`

```c
fsp_err_t spi_write_read(spi_ctrl_t * const p_api_ctrl,
                         void const * p_src,
                         void * p_dest,
                         uint32_t const length,
                         spi_bit_width_t const bit_width);
```

Starts a simultaneous SPI transmit/receive operation.

### `spi_deinit()`

```c
void spi_deinit(void);
```

Closes the Master and Slave instances when open.

### SPI callbacks

```c
void spi_master_callback(spi_callback_args_t * p_args);
void spi_slave_callback(spi_callback_args_t * p_args);
```

Each callback records `SPI_EVENT_TRANSFER_COMPLETE` when a transfer completes. Other received events are recorded as `SPI_EVENT_TRANSFER_ABORTED` by the supplied implementation.

## Reference configuration assumptions

### Master and Slave instances

The SBB assumes generated SPI configurations named:

- `g_spi_master_ctrl`
- `g_spi_master_cfg`
- `g_spi_slave_ctrl`
- `g_spi_slave_cfg`

The README describes DMA-supported SPI **Channel 0** and **Channel 1** configured in Master and Slave modes.

### SPI implementation

`spi_ep.h` selects the driver family at build time:

- `BSP_PERIPHERAL_SPI_B_PRESENT` maps the abstraction to `R_SPI_B_*`.
- `BSP_PERIPHERAL_SPI_PRESENT` maps it to `R_SPI_*`.

### Transfer width

The reference application explicitly performs transfers with:

```c
SPI_BIT_WIDTH_32_BITS
```

It pads Master user input to a multiple of four bytes and divides the padded byte count by four to obtain the number of SPI transfers.

### Input size

The application allocates Master and Slave data buffers as:

```c
static uint32_t g_master_tx_buff[BUFFER_SIZE];
static uint32_t g_master_rx_buff[BUFFER_SIZE];
static uint32_t g_slave_tx_buff[BUFFER_SIZE];
static uint32_t g_slave_rx_buff[BUFFER_SIZE];
```

with `BUFFER_SIZE` defined as 16. The README and application prompt specify that user data must be **less than 64 bytes**, including the end-of-line character according to the README.

### WriteRead input relationship

The original README states that the number of bytes sent for the Slave should be less than the number of bytes sent for the Master when using the WriteRead operation.

### Transfer completion

The reference application waits until both `g_master_event_flag` and `g_slave_event_flag` report `SPI_EVENT_TRANSFER_COMPLETE`. It uses a decrementing software counter starting from `MAX_COUNT` (`0xFFFFFF`) to detect a timeout and also checks for an aborted event.

## What must change when adapting the SBB?

### Pins and physical routing

The Master and Slave SPI signals are physically connected with jumper wires in the reference design. If the SBB is moved to another board or another SPI pin assignment, update the generated pin configuration and hardware wiring for MISO, MOSI, and RSPCK.

For the supported FPB reference configurations, use the wiring documented in the Hardware setup section.

### Master/Slave and SPI channel configuration

The reusable source relies on the generated `g_spi_master_*` and `g_spi_slave_*` configurations. Any changed board configuration must preserve compatible Master and Slave roles and whatever SPI/DMAC configuration is required by the intended design.

### DMA configuration

The README explicitly describes the example as using SPI through DMAC-supported channels, but the supplied handwritten files do not expose or define the generated DMAC transfer configurations. If the generated FSP configuration is changed, verify the DMA support associated with both SPI instances in that project.

### Transfer width and buffers

The reference application assumes 32-bit transfers and aligns Master data to four-byte boundaries. Changing the SPI bit width requires corresponding changes to the reference application's length, padding, buffer interpretation, and comparison logic.

### SPI clocks and timing

The supplied source and README do not state the SPI clock frequency, clock divider, clock polarity, clock phase, or DMA timing configuration for FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1. Those properties must be verified against the generated Master/Slave SPI configuration if the board or clock configuration is changed.

### Timeout behavior

`MAX_COUNT` is a software loop count, not a time unit. The supplied source does not translate `0xFFFFFF` into a guaranteed elapsed time. If deterministic timeout behavior is required, the integrating application must define it according to its execution environment.

### Memory and task priorities

The supplied files do not specify an RTOS, task priorities, stack/heap requirements, or measured memory usage. The reference application executes in a foreground loop and observes callback-updated event flags. Any tasking and memory requirements must be determined by the integrating application.

### Console middleware

The demo uses `console/uart_console.h`, `printf()`, and standard input functions. These are reference-application dependencies rather than requirements of the primitive SPI wrapper APIs in `spi_ep.*`.

## Dependencies

### Renesas FSP SPI HAL

Depending on BSP capabilities, the SBB maps its operations to either:

- `R_SPI_B_Open`, `Read`, `Write`, `WriteRead`, and `Close`; or
- `R_SPI_Open`, `Read`, `Write`, `WriteRead`, and `Close`.

### DMAC

The supplied README states that the SPI channels have DMA support and that SPI data is transmitted and received through the DMAC module. The supplied source files do not directly call DMAC APIs or enumerate the generated DMAC objects, so detailed DMA configuration must be obtained from the corresponding generated FSP project configuration.

### FSP and generated configuration

The source depends on:

- `hal_data.h`
- `g_spi_master_ctrl`
- `g_spi_master_cfg`
- `g_spi_slave_ctrl`
- `g_spi_slave_cfg`
- `R_FSP_VersionGet()` in the reference application

### Console and C runtime

The reference application uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- `printf()`
- `fgets()`
- `atoi()`
- `strlen()`
- `memcmp()`
- `memset()`
- `strcspn()`

### Hardware resources

For the documented boards, the reference implementation requires:

- Two DMA-supported SPI configurations, one Master and one Slave.
- MISO, MOSI, and RSPCK signals connected between the two SPI interfaces.
- Three jumper wires.
- USB/debug connection.
- A serial terminal for the interactive demonstration.

## Serial terminal configuration

The supplied README specifies:

| Setting | Value |
|---|---|
| Port | Port provided by J-Link On-Board |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term, it also specifies:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

The README states that operation is not guaranteed for unsupported input forms such as floating-point or special-character input.

## Demonstrated behavior and performance

### Separate Write and Read operation

For menu option `1`, the reference application:

1. Reads Master transmit data from the terminal.
2. Pads its length to a four-byte boundary when needed.
3. Starts a Slave read.
4. Starts a Master write.
5. Waits for Master and Slave transfer completion.
6. Uses the Slave's received data as its transmit data.
7. Starts a Slave write.
8. Starts a Master read.
9. Waits for completion again.
10. Compares the Master's original transmit data against its received data using `memcmp()`.
11. Prints successful operation when the buffers match.

### Simultaneous WriteRead operation

For menu option `2`, the reference application:

1. Reads separate Master and Slave transmit data.
2. Pads the Master data length to a four-byte boundary.
3. Starts Slave WriteRead using the Master transfer length.
4. Starts Master WriteRead using the same length.
5. Waits for both transfer-completion events.
6. Compares Slave-transmitted data with Master-received data.
7. Compares Master-transmitted data with Slave-received data.
8. Reports successful operation when the comparisons pass.

### Error behavior

The reference application detects:

- SPI transfer timeout.
- SPI transfer aborted.
- Transmit/receive data mismatch.
- Errors returned by SBB SPI calls.

Its error handler prints the error, flushes the UART console, calls `spi_deinit()`, and enters a breakpoint.

### Performance measurements

The supplied materials provide **no measured SPI throughput, DMAC throughput, CPU utilization, memory footprint, transfer latency, interrupt latency, or timing benchmark**.

The 32-bit transfer width, less-than-64-byte demo input limit, and `MAX_COUNT` wait-loop value are implementation/configuration properties, not measured performance figures.

The SBB should therefore be treated as a functional SPI/DMAC Master-Slave transfer reference rather than a performance benchmark.

## File ownership and modification guidance

| File | Classification | Guidance |
|---|---|---|
| `spi_ep.c` | Reusable SBB implementation | Modify when intentionally changing the reusable SPI wrapper, Master/Slave initialization, callbacks, or driver integration. |
| `spi_ep.h` | Reusable SBB interface and SPI abstraction | Modify carefully when changing public APIs, control constants, or `r_spi`/`r_spi_b` mappings. |
| `dma_spi_example_app.c` | Demo/reference application | Appropriate location for changing the terminal menu, demonstration buffers, transfer sequencing, data checking, timeout policy, or presentation. |
| `dma_spi_example_app.h` | Demo/reference application interface | Contains the reference application's entry point, version, banner, menu, and informational/error strings. |
| `readme.txt` | Existing example documentation | Provides the supported-board list, physical wiring, input limits, transfer-width note, WriteRead constraint, and terminal settings. |

The sources include `hal_data.h` and rely on generated SPI and DMA-related FSP configuration. The supplied files do not enumerate those generated files, so no individual unprovided generated file is classified here.

## Adopting a newer SBB release after local modifications

If `spi_ep.c` or `spi_ep.h` are modified locally, those modifications create differences in the reusable SBB layer that may overlap changes made by a later SBB release.

To reduce upgrade friction:

- Keep terminal menus, input handling, sample buffer behavior, and demo-only validation in `dma_spi_example_app.*` where possible.
- Represent board pin, SPI instance, Master/Slave, and DMA configuration through the generated FSP project configuration where appropriate rather than hard-coding board-specific behavior into `spi_ep.*`.
- Keep necessary SBB source modifications small and record why they are required.
- Keep generated FSP configuration distinct from hand-maintained SBB source.
- When adopting a newer SBB release, compare the new `spi_ep.c` and `spi_ep.h` against local modifications before carrying those changes forward.
- Revalidate SPI Master/Slave configuration, DMA association, pin routing, 32-bit transfer assumptions, callback events, completion handling, and data-integrity checks after an update.

The supplied files define no automated migration or compatibility mechanism for future SBB releases. Compatibility therefore needs to be validated by the integrating project when a newer version is adopted.

## Recommended reading order

1. **`spi_ep.h`** to understand the reusable API, SPI driver mappings, commands, flags, and transfer-related constants.
2. **`spi_ep.c`** to see Master/Slave initialization, primitive SPI operations, cleanup, and callbacks.
3. **`dma_spi_example_app.c`** to understand the two demonstrated transfer sequences, 32-bit data handling, event waiting, and verification logic.
4. **`dma_spi_example_app.h`** to see the application entry point and demo menu/information strings.
5. **`readme.txt`** for the original supported-board wiring and terminal requirements.
