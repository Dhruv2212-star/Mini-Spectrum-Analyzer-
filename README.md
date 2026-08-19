
**Ai ussage**- used to find correct parts for the build and help choose the correct MCU for this project & debugging the firmware and  STM32 **ONLY**.  

**Project Description**- The Mini Spectrum Analyzer is a compact electronic device designed to detect and display radio frequency (RF) signals over a selected frequency range. 

It uses an ADF4351 frequency synthesizer to generate a programmable local oscillator and an ADL5801 mixer to convert the received RF signal into an intermediate frequency.  

The AD8313 logarithmic detector measures the signal strength, while the STM32G031K8T6 microcontroller processes the data and displays the spectrum on an SPI TFT display.  

Power input- The device is powered through a USB-C connector and includes onboard power regulation, protection circuitry, user control buttons, status indicators for easy debugging.  

*Designed in Kicad 10.0.3 as 4 layered PCB with independent power planes such as GND & +3v3*

**Note to reviewer**- the Format of the Readme is kept in tabular manner to keep things organized and easy to read. instead of long para i have used tables which i created in excel and pasted it into git file;. if you do have any further question regarding to my project, feel free to reach out to me on slack where i can explain it to you if you want.  
**[slack id- U0AFT499S6S,  Magic tree/Dhruv Patel]**
------------------------------------------------------------------------------------------------------------------------------------------
# Firmware Flashing Guide  
Don't flash the ZIP directly, The ZIP contains the ADF4351 driver source, not a ready-to-flash .bin  
You first need an STM32 project that compiles it, Create the STM32 project In STM32CubeIDE, create a project for: STM32G031K8T6  
Then configure the pins to match our PCB 
The ADF4351 uses a 3-wire serial interface and accepts a programmable 35 MHz–4.4 GHz output range.  
Now *Configure SPI1  
In CubeMX the following:  
SPI1 → Master  
8-bit  
MSB first  
CPOL = Low  
CPHA = 1st Edge  
Software NSS  
Configure PA4, PA5 and PA7 for the SPI interface, with PA4 ultimately serving as the ADF4351 LE control  
Add the driver. From the ZIP, copy: adf4351.h, adf4351.c  

into your CubeIDE project:  
Core/Inc/adf4351.h  
Core/Src/adf4351.c  
Then copy the initialization from: Maine.c to main.c  
now, Set your actual reference  
Our PCB uses:  
Y1 = 10 MHz TCXO  
hz = 10000000UL  
The ADF4351's reference/PFD configuration is important because it directly affects the generated frequency.  
->Build the project  
In CubeIDE:  
Project → Build Project  

 If it has:  
0 errors  
0 warnings related to the driver, ONLY then proceed 

*Connect your ST-LIN  
Use your J4 SWD header:  
J4-1 > SWDIO  
J4-2 > SWCLK   
J4-3 > NRST  
J4-4 > +3V3 / VTREF  
J4-5 > GND  
First test: don't test the spectrum analyzer yet  

Connect the ST-LINK to your PC and open STM32CubeProgrammer. ST officially supports programming STM32 devices through ST-LINK/SWD, and CubeProgrammer can program and verify the MCU flash  
Click:  
ST-LINK > SWD > Connect  
If it detects your STM32G031, stop there and tell me that it connected successfully.  
**Then we'll flash it**  
Compile > generate .elf > flash > reset > verify ADF4351 lock > check RF output  

so the driver is configured for:  

| STM32 U5     | ADF4351 U3   |
| ------------ | ------------ |
| PA4 — pin 11 | LE — pin 3   |
| PA5 — pin 12 | CLK — pin 1  |
| PA7 — pin 14 | DATA — pin 2 |
| PB0 — pin 15 | CE — pin 4   |
| PA1 — pin 8  | LD — pin 25  |


------------------------------------------------------------------------------------------------------------------------------------------
# PCB component's Role in the circuit 
 # Block 1 — USB-C Power Input & Protection
 <img width="278" height="270" alt="image" src="https://github.com/user-attachments/assets/de027333-ba95-42fd-aa08-6013625df111" />  

 | Ref | Component        | Role in the Circuit    |                                                                                     
| --- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| J1  | USB-C Receptacle | Receives 5 V power from a USB-C source and serves as the main power input to the spectrum analyzer.                                          |
| F1  | Polyfuse         | Protects the board from excessive current by disconnecting the supply during a fault and automatically resetting after the fault is removed. |
| D1  | TVS/ESD Diode    | Protects the USB power line against electrostatic discharge and voltage spikes that could damage sensitive components.                       |
| R1  | 5.1 kΩ           | USB-C CC1 pull-down resistor that identifies the board as a power sink.                                                                      |
| R2  | 5.1 kΩ           | USB-C CC2 pull-down resistor that enables correct cable orientation detection.                                                               |
| C1  | 10 µF            | Reduces low-frequency voltage ripple and provides energy storage during sudden load changes.                                                 |
| C2  | 100 nF           | Filters high-frequency noise on the incoming USB power line.                                                                                 |

# Block 2 — 5 V to 3.3 V Power Regulation
<img width="388" height="184" alt="image" src="https://github.com/user-attachments/assets/553fb35f-b87a-4fd6-93a2-74f583cecbcf" />  

| Ref | Component       | Role in the Circuit                                                                         |
| --- | --------------- | ------------------------------------------------------------------------------------------- |
| U1  | 3.3 V Regulator | Converts the incoming 5 V supply into a stable 3.3 V rail for the digital and RF circuitry. |
| L1  | Ferrite Bead    | Suppresses high-frequency switching noise between the regulator and the clean 3.3 V rail.   |
| C3  | 10 µF           | Stabilizes the regulator input and output while improving transient response.               |
| C4  | 100 nF          | High-frequency bypass capacitor for the regulator output.                                   |
| C5  | 100 nF          | Additional local decoupling capacitor to maintain a clean supply.                           |

# Block 3 — RF Input & Signal Conditioning
<img width="285" height="232" alt="image" src="https://github.com/user-attachments/assets/09b9d0d7-c975-4a9a-ba9f-98dd038968d7" />  

| Ref | Component   | Role in the Circuit                                                       |
| --- | ----------- | ------------------------------------------------------------------------- |
| R6  | 10 Ω        | Provides input damping and limits RF transients entering the signal path. |
| R7  | 10 Ω        | Works with R6 to improve signal integrity and reduce ringing.             |
| C6  | 100 pF      | AC-couples the RF signal while blocking DC from reaching the mixer stage. |
| D2  | PESD5V0X1UL | Protects the RF input against electrostatic discharge events.             |

# Block 4 — RF Detector & Block 5 — Frequency Generation, Mixer & IF Stage  
<img width="524" height="344" alt="image" src="https://github.com/user-attachments/assets/8ed2acdf-cf41-4b15-80a0-d072d54b76de" />

| Ref    | Component             | Role in the Circuit                                                                   |
| ------ | --------------------- | ------------------------------------------------------------------------------------- |
| U2     | AD8313                | Converts the RF signal level into a proportional DC voltage that the MCU can measure. |
| C7–C10 | Decoupling Capacitors | Filter supply noise and maintain stable operation of the RF detector.                 |


| REf    | Component                        | Role in the Circuit                                                                                                  |
| ------- | -------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| U3      | ADF4351                          | Generates a programmable local oscillator signal used to sweep the analyzer across different frequencies.            |
| U4      | ADL5801                          | Mixes the incoming RF signal with the local oscillator to produce an intermediate-frequency signal.                  |
| Y1      | 10 MHz TCXO                      | Provides a highly stable reference clock for accurate frequency generation.                                          |
| T1      | ADT4-1T+ Transformer             | Converts the differential IF signal into the required form while providing impedance matching.                       |
| R11–R15 | Bias / PLL Resistors             | Set PLL operating conditions, bias RF outputs, and configure the frequency synthesizer.                              |
| C11–C22 | Coupling / Decoupling Capacitors | Provide AC coupling, supply filtering, RF bypassing, and PLL loop-filter functions required for stable RF operation. |

# Block 6 — MCU, Display & Programming & Block 7 — User Interface  
<img width="408" height="485" alt="image" src="https://github.com/user-attachments/assets/6e32f34b-3bc0-4904-a657-66ae1c01e7a0" />

| Ref | Component             | Role in the Circuit                                                                                      |
| --- | --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| U5  | STM32G031K8T6         | Controls the analyzer, programs the synthesizer, samples detector data, processes measurements, and updates the display. |
| J3  | ST7789 Display Module | Displays the measured spectrum, menus, and analyzer information to the user.                                             |
| J4  | SWD Header            | Allows firmware programming and debugging of the microcontroller.                                                        |
| R16 | 10 kΩ                 | Pull-up resistor for the MCU reset pin, ensuring reliable startup.                                                       |
| C24 | 100 nF                | High-frequency bypass capacitor for the MCU supply.                                                                      |
| C25 | 4.7 µF                | Bulk decoupling capacitor that stabilizes the MCU power rail during rapid current changes.                               |

| Ref | Component   | Role in the Circuit                                      |
| --- | ----------- | -------------------------------------------------------- |
| SW1 | Push Button | Increases frequency or navigates upward through menus.   |
| SW2 | Push Button | Decreases frequency or navigates downward through menus. |
| SW3 | Push Button | Selects menu options and confirms user actions.          |
| R17 | 10 kΩ       | Pull-up resistor for SW1 input.                          |
| R18 | 10 kΩ       | Pull-up resistor for SW2 input.                          |
| R19 | 10 kΩ       | Pull-up resistor for SW3 input.                          |
<img width="433" height="573" alt="screenshot of PCB" src="https://github.com/user-attachments/assets/864d23e9-2eb7-4606-8995-59e0458ccd69" />


