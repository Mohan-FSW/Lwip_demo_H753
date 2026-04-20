# STM32H753 Ethernet Demo with lwIP with out FREERTOS

A bare-metal Ethernet demo for the NUCLEO-H753ZI development board using the lwIP 2.1.2 TCP/IP stack. No RTOS involved — the network stack runs in a simple polling loop. The board responds to ping and can be extended for UDP/TCP applications.

---

## What This Does

The firmware initialises the STM32H753 Ethernet MAC in RMII mode, brings up the LAN8742A PHY, and runs the lwIP stack at a fixed static IP address. Once flashed and connected to a PC over Ethernet, the board will respond to ping. The link state is polled every 100 ms so the board recovers automatically if the cable is unplugged and reconnected.

---

## Hardware

- **Board:** NUCLEO-H753ZI
- **MCU:** STM32H753ZIT6 — Cortex-M7, 480 MHz, 2 MB Flash, 1 MB RAM
- **PHY:** LAN8742A on-board, RMII interface, MDIO address 0
- **Connection:** Standard RJ45 Ethernet cable between board and PC (or switch)

### RMII Pin Assignments

| Pin  | Function     |
|------|--------------|
| PA1  | ETH_REF_CLK  |
| PA2  | ETH_MDIO     |
| PA7  | ETH_CRS_DV   |
| PC1  | ETH_MDC      |
| PC4  | ETH_RXD0     |
| PC5  | ETH_RXD1     |
| PB13 | ETH_TXD1     |
| PG11 | ETH_TX_EN    |
| PG13 | ETH_TXD0     |

All Ethernet pins use AF11 at very-high speed. No external pull resistors required.

---

## Network Configuration

The board uses a static IP — there is no DHCP. Configure your PC's Ethernet adapter to be on the same subnet before trying to ping.

| Setting     | Value           |
|-------------|-----------------|
| Board IP    | 192.168.1.100   |
| Subnet mask | 255.255.255.0   |
| Gateway     | not required    |

Set your PC to something like 192.168.1.10 / 255.255.255.0 with no gateway. A direct cable between PC and board works fine — the LAN8742A supports auto-MDIX.

## Clock Tree
HSI (64 MHz)
└─ PLL1 M=4, N=60, P=2
└─ SYSCLK 480 MHz
├─ HCLK 240 MHz (AHB /2)
└─ APBx 120 MHz (APBx /2)

Flash latency is set to 4 wait states as required for 480 MHz operation at VOS0.

---

## Memory Layout

The Cortex-M7 D-cache is enabled. DMA regions must be kept non-cacheable or manually maintained — the MPU is configured accordingly before caches are turned on.

| Region              | Address    | Size   | MPU policy              | Contents                         |
|---------------------|------------|--------|-------------------------|----------------------------------|
| Flash               | 0x08000000 | 2 MB   | —                       | Code, read-only data             |
| RAM_D1 (AXI SRAM)  | 0x24000000 | 512 KB | Default cacheable       | Stack, .bss, .data               |
| RAM_D2 (AHB SRAM)  | 0x30020000 | 128 KB | Write-back, cacheable   | lwIP heap (LWIP_RAM_HEAP_POINTER) |
| RAM_D2 (AHB SRAM)  | 0x30040000 | 32 KB  | Non-cacheable, device   | ETH DMA descriptors + RX pool    |

The 32 KB non-cacheable window at 0x30040000 is the critical region. ETH DMA descriptors and the zero-copy RX buffer pool both live here so the DMA and CPU always see consistent data without software cache maintenance on the receive path.

---

## Building

Open the project in **STM32CubeIDE** (tested with 1.13+) and build with the default `Debug` or `Release` configuration. The project uses GCC (arm-none-eabi) and the generated makefile from CubeIDE.

No external dependencies — everything is included in the repository (HAL drivers, CMSIS, lwIP middleware).

---

## Flashing

With the NUCLEO board connected over USB (ST-LINK):

**From CubeIDE:** Run → Debug (or Run) — the IDE flashes automatically.

**From command line using STM32CubeProgrammer:**

STM32_Programmer_CLI -c port=SWD -w build/Lwip_demo_H753.elf -rst

**Drag and drop:** The NUCLEO enumerates as a USB mass storage device. Drag the `.bin` file onto it.

---

## Testing

1. Flash the firmware and power the board.
2. Connect an Ethernet cable between the board and your PC.
3. Set your PC Ethernet adapter to static IP `192.168.1.10`, mask `255.255.255.0`.
4. Wait about 3 seconds for the PHY to finish initialising (the green link LED on the RJ45 should come on).
5. Open a terminal and ping:

ping 192.168.1.100


You should see replies with sub-millisecond latency on a direct connection. If ping times out, check the link LED first, then verify your PC adapter is on the `192.168.1.x` subnet.

---

## Project Layout

Lwip_demo_H753/
├── Core/
│ ├── Inc/
│ │ ├── main.h
│ │ └── stm32h7xx_hal_conf.h
│ └── Src/
│ ├── main.c entry point, MPU, clocks, main loop
│ ├── stm32h7xx_hal_msp.c peripheral MSP callbacks
│ └── stm32h7xx_it.c interrupt handlers (ETH, TIM6)
├── LWIP/
│ ├── App/
│ │ └── lwip.c network interface init, IP config, process loop
│ └── Target/
│ ├── ethernetif.c HAL to lwIP glue, PHY management, DMA callbacks
│ └── lwipopts.h lwIP compile-time options
├── Drivers/
│ ├── STM32H7xx_HAL_Driver/ ST HAL (unmodified)
│ └── BSP/Components/lan8742/ LAN8742A PHY driver
├── Middlewares/Third_Party/LwIP/ lwIP 2.1.2 (unmodified)
├── STM32H753ZITX_FLASH.ld linker script
└── Lwip_demo_H753.ioc CubeMX project file


---

## Known Issues Fixed

**D-cache coherency on transmit** — The most common STM32H7 Ethernet pitfall. TX buffers are in the cacheable lwIP heap. Without an explicit cache clean before triggering the DMA, the Ethernet controller reads stale data from physical RAM and transmits garbage. Fixed by calling `SCB_CleanDCache_by_Addr()` on each TX buffer immediately before `HAL_ETH_Transmit()`.

**RX pool placement** — The zero-copy RX pool (`memp_memory_RX_POOL_base`) must land in the non-cacheable region at 0x30040200. The GCC section attribute is forward-declared before `LWIP_MEMPOOL_DECLARE` to guarantee correct placement regardless of compiler version.

**HAL transmit error handling** — `HAL_ETH_Transmit()` return value was previously ignored. Errors are now propagated back to lwIP as `ERR_IF`.

