# IKASCC
Konami SCC core for FPGA implementation. Based on schematics extracted from a die shot. © 2024 Sehyeon Kim(Raki) 

## Features
* A **cycle-accurate, die shot based, BSD2 licensed** core.
* Four implementation options available.

## Module instantiation
The steps below show how to instantiate the IKASCC module in Verilog:

1. Download this repository or add it as a submodule to your project.
2. You can use the Verilog snippet below to instantiate the module.

```verilog
//Verilog module instantiation example
IKASCC #(.IMPL_TYPE(1), .RAM_BLOCK(1)) main (
    .i_EMUCLK                   (                           ),
    .i_MCLK_PCEN_n              (                           ),
    .i_RST_n                    (                           ),

    .i_CS_n                     (                           ),
    .i_RD_n                     (                           ),
    .i_WR_n                     (                           ),
    .i_ABLO                     (                           ),
    .i_ABHI                     (                           ),

    .i_DB                       (                           ),
    .o_DB                       (                           ),
    .o_DB_OE                    (                           ),

    .o_ROMCS_n                  (                           ),
    .o_ROMADDR                  (                           ),

    .o_SOUND                    (                           ),

    .o_TEST                     (                           )
);
```
3. Attach your signals to the port. The direction and the polarity of the signals are described in the port names. The section below explains what the signals mean.


**PARAMETERS**
* `IMPL_TYPE`
    * **0** : (sync) SoC implementaton with a fast clock above 10MHz: Wavetable RAM samples data on the rising edge, when the internal write request is valid
    * **1** : (sync) SoC implementaton/standalone module with a slow clock around 3.58MHz: Wavetable RAM samples data on the falling edge, when the internal write request is valid
    * **2** : (async) standalone module with a slow clock around 3.58MHz: All registers sample CPU data on the rising edge of /WR. Therefore, **you must promote the /WR signal** appropriately. A separate delayed /WR is required for writes to the wavetable RAMs, which is achieved by a buffer chain given the synthesis attribute "keep". **You will need to treat this signal as a global clock as well.** As a result, you need three global clocks. SCC clock, /WR and delayed /WR. It's up to you to choose the attribute and determine the length of the delay chain based on your synthesis tool.
* `RAM_BLOCK` If **1**, the core will use BRAMs. If **0**, the core will use DFFs. If `IMPL_TYPE` is **2**, it will use DFFs always.

**INTERNAL PARAMETERS**
* `RAMCTRL_ASYNC` Valid for the `IMPL_TYPE` **0** or **1**. This is important when emulating the characteristics of the original chip. An SCC wavetable player samples data from an embedded RAM on the rising edge of the clock. When the CPU writes data to the embedded RAM, and the DFF picks up the data provided by the CPU. This is a reported glitch of the chip. I made these parameters because I'm not sure about global buffer delay, gate delay and access time of the embedded RAM of TC22SC standard cell ASIC. Moreover, I haven't compared the behavior of the core with the actual chip. It can have a value of **0** or **1**, and there is a detailed description in the file `IKASCC_player_s.v`. You should adjust the timing manually if you want to use the `IMPL_TYPE` **2**.
* `RAM_ASYNC_WRITE_DELAY_CHAIN_LENGTH` defines the delay chain length. Values can be very different depending on the FPGA.


**PORTS**
* `i_EMUCLK` is your system clock.
* `i_MCLK_PCEN_n` is the clock enable(negative logic) for rising edge of the MCLK. If you want to use a 3.58 MHz clock, you can supply 3.58 MHz to `i_EMUCLK` and connect this port to logic 0.
* `i_RST_n` is the synchronous reset on the `IMPL_TYPE` **0** or **1**, asynchronous reset on the the `IMPL_TYPE` **2**.
* `o_DB_OE` is the output enable for FPGA's tri-state I/O driver.
* `o_SOUND` is the sound output. You can sample the sound on the rising edge of MCLK.

## Implementation note
* The official SCC data sheet specifies an access time of 60ns maximum and 35ns minimum. CPU data appears on the embedded RAM data bus about 6ns after the CPU asserts /WR, and the embedded RAM write strobe is enabled 8.25ns later after that. As a result, the minimum guaranteed time for write access seems to be 35ns after the /WR assertion. Since the embedded RAM in the NEC 1.5um CMOS-4R gate array has an access time of 18ns, the access time of the TC22SC(2um) is estimated to be 20-30ns. Also, I assumed that the gate delay of the TC22SC is 1.5ns. If true, the `RAMCTRL_ASYNC` parameter should be set to **1**.