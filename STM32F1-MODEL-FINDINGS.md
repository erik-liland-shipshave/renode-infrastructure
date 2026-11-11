# STM32F1 model findings (Renode 1.17.0)

Defects found while booting unmodified STM32CubeF1 HAL firmware (an STM32F103CB bootloader and application) on
`platforms/cpus/stm32f103.repl`. Each one was confirmed with a register-level repro on Renode 1.17.0
(`1.17.0+20260907gitf1dd1b4af`, Linux portable build). Each was also checked against `master` of renode and
renode-infrastructure as of 2026-11-11: the source files involved there match 1.17.0.

Fix branches, each one commit on top of upstream `master`:

| # | Model / platform | Defect | Fix branch | Upstream |
| --- | --- | --- | --- | --- |
| 1 | `STM32G0DMA` | Any read of CCRx disables the channel; HTIF/TCIF are not sticky | [renode-infrastructure `stm32g0dma-ccr-read-and-sticky-flags`](https://github.com/erik-liland-shipshave/renode-infrastructure/tree/stm32g0dma-ccr-read-and-sticky-flags) | Related to renode/renode#964 |
| 2 | `STM32SPI` | F1/F4/L0/L1 get a 4-byte RX FIFO instead of a single receive buffer | [renode-infrastructure `stm32spi-single-rx-buffer-on-f1-f4-l0-l1`](https://github.com/erik-liland-shipshave/renode-infrastructure/tree/stm32spi-single-rx-buffer-on-f1-f4-l0-l1) | None found |
| 3 | `STM32F1GPIOPort` | Inputs with pull-up read 0; BSRR/BRR/ODR writes to input pins are dropped | [renode-infrastructure `stm32f1gpioport-input-pull-up-down`](https://github.com/erik-liland-shipshave/renode-infrastructure/tree/stm32f1gpioport-input-pull-up-down) | None found |
| 4 | `STM32F1GPIOPort` | CRL/CRH read 0x00000000 after reset instead of 0x44444444 | [renode-infrastructure `stm32f1gpioport-crl-crh-reset-value`](https://github.com/erik-liland-shipshave/renode-infrastructure/tree/stm32f1gpioport-crl-crh-reset-value) | None found (same class as renode/renode#1036) |
| 5 | `stm32f103.repl` | No peripheral bit-band alias; RCC_CR is a constant tag and RCC_CFGR ignores writes, so `SystemClock_Config` hangs | [renode `stm32f103-bitband-and-rcc`](https://github.com/erik-liland-shipshave/renode/tree/stm32f103-bitband-and-rcc) | renode/renode#652 (symptom), #461 (closed, bit-band half) |

Branches 3 and 4 touch the same file but different lines, and they merge cleanly in either order.

The fix branches have not been built into Renode and run through Renode's own test suite. Each fix was verified
by loading a renamed copy of the modified model into Renode 1.17.0 at run time (`include @Model.cs`) and running
the repros below against it. The `stm32f103.repl` change was also run with `scripts/single-node/stm32f103.resc`
(the Zephyr button demo), whose UART output is unchanged.

All repros are Renode monitor scripts that need no firmware binary.

---

## 1. STM32G0DMA: a CCR read disables the channel; HT/TC flags are not sticky

`platforms/cpus/stm32f103.repl` uses `STM32G0DMA` for both DMA controllers.

**Root cause.** The EN field of CCRx is defined with `valueProviderCallback: _ => false`.
`PeripheralRegister.ReadInner()` stores what a value provider returns back into the register's underlying value,
so every read of CCRx clears the stored EN bit. From then on, every request logs
`DMA peripheral request on channel N ignored - channel is disabled`. renode/renode#964 reports the read-back
value; the stopped channel is the consequence it does not mention.

The STM32CubeF1 `HAL_DMA_IRQHandler` begins with `source_it = hdma->Instance->CCR`, so a circular
peripheral-to-memory channel (`HAL_UART_Receive_DMA`, `HAL_UARTEx_ReceiveToIdle_DMA`) stops at its first
half-transfer interrupt.

`DoTransfer` also recomputes the flags on every transfer, where hardware sets them once and only software
clears them (RM0008 13.3.7):

- `HalfTransfer = dataCount <= originalDataCount / 2` sets HTIF again on every transfer of the second half, right
  after software has cleared it.
- `TransferComplete = dataCount == 0` clears TCIF on the first transfer after a circular wrap.

**Repro** (USART1 RX on DMA1 channel 5, 8-byte circular buffer):

```
mach create
machine LoadPlatformDescription @platforms/cpus/stm32f103.repl
sysbus WriteDoubleWord 0x40020060 0x40013804   # CPAR5 = USART1_DR
sysbus WriteDoubleWord 0x40020064 0x20001000   # CMAR5
sysbus WriteDoubleWord 0x4002005C 8            # CNDTR5
sysbus WriteDoubleWord 0x40020058 0xA7         # CCR5 = MINC|CIRC|HTIE|TCIE|EN
sysbus WriteDoubleWord 0x40013814 0x40         # USART1_CR3.DMAR
sysbus WriteDoubleWord 0x4001380C 0x2004       # USART1_CR1.UE|RE
sysbus.usart1 WriteChar 0x20
sysbus.usart1 WriteChar 0x21
sysbus.usart1 WriteChar 0x22
sysbus.usart1 WriteChar 0x23
sysbus ReadDoubleWord 0x40020000               # ISR: 0x00050000 (HTIF5|GIF5)
sysbus ReadDoubleWord 0x40020058               # CCR5: 0x000000A6 - EN reads 0, and this read disables the channel
sysbus WriteDoubleWord 0x40020004 0x40000      # CHTIF5
sysbus.usart1 WriteChar 0x24                   # "request on channel 5 ignored - channel is disabled"
sysbus.usart1 WriteChar 0x25
sysbus.usart1 WriteChar 0x26
sysbus.usart1 WriteChar 0x27
sysbus ReadDoubleWord 0x4002005C               # CNDTR5: 0x4, expected 0x8 (wrapped)
sysbus ReadDoubleWord 0x20001004               # 0x00000000, expected 0x27262524
```

Without the CCR5 read, the same setup shows the flag problem:

| After | ISR on 1.17.0 | Expected |
| --- | --- | --- |
| 4 bytes, CHTIF5, 1 more byte | `0x00050000` | `0x00000000` |
| 8 bytes (wrap) | `0x00070000` | `0x00030000` (HTIF5 already cleared in this lap) |
| 1 more byte, nothing cleared | `0x00000000` | `0x00030000` |

**Fix.** EN is a plain flag. Only its 0 -> 1 transition starts a memory-to-memory or memory-to-peripheral transfer,
so a read-modify-write of CCR on a running channel does not start another one. HTIF is set by the transfer that
crosses the half-way point and TCIF by the one that reaches zero, and both are cleared only through IFCR. With the
fix, the repro wraps correctly (CNDTR5 = 8, buffer word = 0x27262524) and the flags match the expected column.

Side note: `stm32f103.repl` connects `usart1.DMARequest -> dma1@4 | dma1@5`, so every received byte also requests
on the TX channel 4, which logs "ignored" while it is disabled. That is log noise, not this defect.

---

## 2. STM32SPI: F1/F4/L0/L1 get a 4-byte receive FIFO

**Root cause.** `bufferCapacity` defaulted to 4 for every series. STM32F1, F4, L0 and L1 have no RX FIFO, only a
single receive buffer: a byte that arrives while RXNE is set is lost and OVR is raised (RM0008 25.3.10). Only F0,
F7, G0 and L5 have a 32-bit RX FIFO. `platforms/cpus/stm32l071.repl` already sets `bufferCapacity: 1` by hand,
but `platforms/cpus/stm32f4.repl` uses the default.

After a write-only transfer, up to four stale bytes stay queued. The HAL removes them with
`__HAL_SPI_CLEAR_OVRFLAG` (one DR read, then one SR read), which takes out only one. The next full-duplex transfer
then returns a stale byte instead of the device's response. Disabling SPE does not empty the queue either, so the
stale bytes survive a re-initialisation, such as a bootloader handing over to an application.

**Repro:**

```
mach create
machine LoadPlatformDescriptionFromString "spi1: SPI.STM32SPI @ sysbus 0x40013000 { series: STM32Series.F1 }; dev: Mocks.DummySPISlave @ spi1"
sysbus.spi1.dev EnqueueValue 0xA1
sysbus.spi1.dev EnqueueValue 0xA2
sysbus.spi1.dev EnqueueValue 0xA3
sysbus.spi1.dev EnqueueValue 0xA4
sysbus.spi1.dev EnqueueValue 0x5A
sysbus WriteDoubleWord 0x40013000 0x344        # SSM|SSI|SPE|MSTR
sysbus WriteDoubleWord 0x4001300C 0x01         # HAL_SPI_Transmit of 4 bytes: DR never read
sysbus WriteDoubleWord 0x4001300C 0x02
sysbus WriteDoubleWord 0x4001300C 0x03
sysbus WriteDoubleWord 0x4001300C 0x04
sysbus ReadDoubleWord 0x4001300C               # __HAL_SPI_CLEAR_OVRFLAG: DR ...
sysbus ReadDoubleWord 0x40013008               # ... then SR: 0x00000003 (RXNE still set)
sysbus WriteDoubleWord 0x4001300C 0xFF         # 1-byte HAL_SPI_TransmitReceive
sysbus ReadDoubleWord 0x4001300C               # 0x000000A2, expected 0x0000005A
```

**Fix.** `bufferCapacity` is now `int?`, and when it is not given it defaults by series: 1 for F1, F4, L0 and L1,
and 4 for the others. An explicit value still overrides it. With the fix, the clearing SR read returns `0x42` (OVR
set, RXNE clear, as on hardware), and DR returns `0x5A`. With `series: STM32Series.F7` the behaviour is unchanged.

Not checked: Renode's own tests that use `stm32f4.repl` SPIs, which now get the single buffer.

Workaround without the fix: add `bufferCapacity: 1` to the SPI in the `.repl`.

---

## 3. STM32F1GPIOPort: input pull-up/pull-down not modelled

**Root cause.** On the F1, an input with `CNF = 10` is pulled up when its ODR bit is 1 and pulled down when it is
0 (RM0008 9.1.1, Table 20). `HAL_GPIO_Init` selects the pull by writing the pin to BSRR (pull-up) or BRR
(pull-down). The model's `SetBitsFromMask` drops BSRR/BRR writes to input pins with the warning
`Trying to set the state of the input pin #N`, and ODR reads return the connection states rather than a register.
An unconnected input configured with pull-up therefore reads 0, and firmware waiting for a pulled-up line (an
active-low button, an open-drain ready line) sees it asserted from reset.

**Repro:**

```
mach create
machine LoadPlatformDescription @platforms/cpus/stm32f103.repl
sysbus WriteDoubleWord 0x40010800 0x44444448   # GPIOA_CRL: PA0 CNF=10 MODE=00 (input, pull-up/down)
sysbus WriteDoubleWord 0x40010810 0x1          # BSRR: ODR0=1 -> pull-up, as HAL_GPIO_Init does
sysbus ReadDoubleWord 0x40010808               # IDR: 0x00000000, expected 0x00000001
sysbus ReadDoubleWord 0x4001080C               # ODR: 0x00000000, expected 0x00000001
```

**Fix.**

- ODR is a register of its own, written by ODR, BSRR and BRR for every pin.
- A pulled input takes its level from its ODR bit until a signal arrives through `OnGPIO`. The level shows in IDR
  and on its outgoing connection, so EXTI sees it too.
- After an `OnGPIO` signal, the pin follows that signal. A GPIO connection cannot be released, so once driven, a
  pin stays driven.
- Output pins behave as before, except that an ODR write now also updates IDR, as BSRR already did.
- An alternate function now drives a pin only in an output mode. In input mode `CNF = 10` means pull-up/pull-down,
  not alternate-function push-pull, so before this fix a timer AF could drive a pulled input.

Verified on a standalone port:

- pull-up reads 1 and pull-down reads 0;
- flipping ODR flips both pulls;
- an external `OnGPIO` overrides the pull;
- output pins via BSRR and ODR, and CRL read-back, behave as expected.

---

## 4. STM32F1GPIOPort: CRL/CRH read 0x00000000 after reset

**Root cause.** The CRL/CRH registers are created with the reset value `0x44444444`. Their fields, however, read
from the per-pin arrays, which `Reset()` sets to `PinMode.Input` (0) and `OutputMode.ResetValue =
GeneralPurposePushPull` (0). So CNF reads 00 (analog) instead of 01 (floating input). A read-modify-write of one
pin, as `HAL_GPIO_Init` does, writes analog mode back for the other seven pins of the register.

**Repro:**

```
mach create
machine LoadPlatformDescription @platforms/cpus/stm32f103.repl
sysbus ReadDoubleWord 0x40010800               # GPIOA_CRL: 0x00000000, expected 0x44444444
sysbus ReadDoubleWord 0x40010804               # GPIOA_CRH: 0x00000000, expected 0x44444444
```

**Fix.** `OutputMode.ResetValue = GeneralPurposeOpenDrain` (CNF = 01, a floating input in input mode). CRL/CRH
then read `0x44444444` after construction and after `machine Reset`, and configuring a pin still works.

---

## 5. stm32f103.repl: SystemClock_Config never finishes

renode/renode#652 reports this symptom. The STM32CubeF1 clock setup (HSE + PLL) is blocked by two independent gaps:

1. **No bit-band region.** The F1 HAL switches oscillators through the peripheral bit-band alias:
   `__HAL_RCC_PLL_DISABLE()` writes 0 to `0x42420060` (RCC_CR.PLLON), and HSEON, HSION, LSION, CSSON and PWR_CR.DBP
   are switched the same way. Renode logs `WriteDoubleWord to non existing peripheral at 0x42420060`.
2. **RCC is a constant tag.** `RCC_CR` is `Tag <0x40021000, 0x40021003> "RCC_CR" 0x0A020083`, so PLLRDY always
   reads 1 and `HAL_RCC_OscConfig` times out waiting for PLLRDY == 0 after disabling the PLL. `RCC_CFGR` comes from
   the SVD and ignores writes, so `HAL_RCC_ClockConfig` never sees SWS = PLL either.

Adding the bit-band alias alone is not enough: the tag still pins PLLRDY to 1.

**Repro:**

```
mach create
machine LoadPlatformDescription @platforms/cpus/stm32f103.repl
sysbus WriteDoubleWord 0x42420060 0            # __HAL_RCC_PLL_DISABLE(): "non existing peripheral"
sysbus ReadDoubleWord 0x40021000               # 0x0A020083 - PLLRDY still set
sysbus WriteDoubleWord 0x40021004 0x2          # CFGR.SW = PLL
sysbus ReadDoubleWord 0x40021004               # 0x00000000 - SWS never follows
```

**Fix.** The `stm32f103.repl` change has two parts:

- `bitbandPeripherals: Miscellaneous.BitBanding @ sysbus <0x42000000, +0x2000000>` with
  `peripheralBase: 0x40000000`.
- The RCC_CR tag is replaced by `rcc: Python.PythonPeripheral` running the new `scripts/pydev/stm32f1-rcc.py`. It
  is a register file in which HSIRDY, HSERDY, PLLRDY, LSERDY and LSIRDY follow their enable bits, SWS follows SW,
  and RMVF clears the reset flags. The reset values are RCC_CR = 0x83 and RCC_CSR = 0x0C000000.

Verified:

- HSEON and PLLON via bit-band set their ready flags, and PLL off clears PLLRDY;
- SWS follows SW, LSION via bit-band sets LSIRDY, and RMVF works;
- the Zephyr button demo prints identical output before and after.

The SRAM bit-band alias (0x22000000) is not added: `stm32f103.repl` maps `sram` over 0x20000000-0x2FFFFFFF.

Other F1 blocks are still missing from `stm32f103.repl`:

- PWR, BKP and RTC (RTC_CRL.RTOFF and RSF must read set);
- the flash interface at 0x40022000 (unlock, page/mass erase, EOP), reported in renode/renode#227;
- SPI1/SPI2.

---

## Not reproduced

| Claim | Result on 1.17.0 |
| --- | --- |
| `STM32F1_I2C` never NACKs an address no device answers | Not reproduced. SR1.AF is set for an unregistered address. The missing part is renode/renode#1037: a *registered* child cannot NACK. |
| `STM32F1_I2C` ignores CR1.ACK/POS, so the HAL's 2-byte (POS + BTF) and N-byte (ACK cleared at N-3) master receive never complete | Not reproduced. The polled HAL sequences for 1, 2, 3 and 6 bytes complete with correct data against `Mocks.DummyI2CSlave`. ACK and POS are tagged, but the model's own sequencing copes. Interrupt-mode HAL and real CPU timing were not tested. An earlier hang may have been the 1.16.1 model (changed in renode-infrastructure `033ee445`) or a child model that expects the whole read in one `Read(count)` call, while `STM32F1_I2C` calls `Read(1)` per byte. |
| `STM32_UART` raises DMA requests with one `Blink()` per byte | Not a defect. A DMA model must serve one transfer per rising edge, which `STM32G0DMA` does. |
