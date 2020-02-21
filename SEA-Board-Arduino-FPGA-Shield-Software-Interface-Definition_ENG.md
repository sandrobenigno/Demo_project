# SEA Board -Arduino FPGA Shield Software Interface Definition



| VER  | DATE       | PROD DESC                                                        |
| ---- | ---------- | ---------------------------------------------------------------- |
| 1.0  | 2019-04-16 | FIRST ED.                                                        |
| 1.1  | 2019-04-23 | Mr. Di added a definition specs for the I2C interface            |
| 1.2  | 2019-09-23 | 1. I2C interface adjusts   2. Pins definitions and descriptions  |

 

## Previous

The SEA-S7 Board（Spartan Edge Accelerator Board） integrates a Spartan7 FPGA whith a ESP32 embeded chip
FPGA features such as parallelism, high performance, low power consumption, and flexible configuration.
WIFI and Bluetooth's ESP32 chip can be designed to meet the requirements of various scenarios.

The following is a schematic diagram of the main communication interfaces of the SAE-S7 on-board FPGA, on-board ESP32 and Arduino main control.
For details, please refer to the board schematic diagram.

<p align="center">
<img src ="./images/Block_Diagram.png">
</p>
<p align = "center">
</p>

SEA-S7 can be used as a single board or as an Arduino extension daughter board.
This article only considers the application scenarios of the Arduino extension daughter board.


## Purpose

In the scenario of an Arduino expansion board, the Arduino master uses the following communication interfaces to access the SEA-S7.

. I2C Interface

. SPI Interface

This article will introduce the above interfaces and provide examples and references for FPGA overlay and ESP32 developers.

## I2C interface specification

|    Name    |  ESP32 Pin  |   Description    |
| ---------- | ----------- | ---------------  |
| ESP_SDA_A4 | ESP32_PIN12 | I2C Data Signal  |
| ESP_SCL_A5 | ESP32_PIN13 | I2C Clock Signal |

 

When connected to a main control board such as Arduino, the on-board ESP32 acts as an I2C slave device.

Device address: 0x52
Write data address: 0xA4, read data address 0xA5


2. I2C interface example

| **Address** | **Status** | **Description**                                                                                 |
| ------------- | ------------ | ------------------------------------------------------------------------------------------------- |
|     0x00      |      RO      | On-board FPGA version. 0: XC7S15 1: XC7S25                                                        |
|     0x01      |      RO      | Configure X number Overlay (16bit number) ** Note: ** The overlay directory in the ESP32 SD card is used to store the FPGA bitstream. The name of the bitstream file should follow the naming rules of ** X_Y_Z.bit **, where X is the bitstream number, Y is the description, and Z is the file creation date. Such as: 01_ColorDetect_20190801.bit |
|     0x02      |      RO      | Query the number of existing overlays (bitstream) (16bit data)                                    |
|     0x03      |      RO      | File name of X overlay (file name is transmitted by designated message)                           |
|     0x04      |      RO      | Query the configuration overlay status (0: configuration completed, 1: configuration in progress) |
|     ...       |     Spare    |                                                                                                   |
|    0x0E       |     W / R    | Overlay with configuration file name xx (file name is transmitted by specified message)           |
|    0x0F       |     W / R    | Configure X Number Overlay (16bit number)                                                         |
|    0x2x       |   Reserved   | For AT command configuration WIFI                                                                 |


Take the example of reading the Overlay file name and configuring the Overlay by the Overlay file name as examples:

Read file name                                                                                                                  

| **Address** | **Instruction** | **Data 16bit** |      | **Address** | **Instruction** |  **Length 8bit**  |    **N Bytes**   |
| ----------- | --------------- | -------------- | ---- | ----------- | --------------- | ----------------- | ---------------- |                           
|    OXA4     |       0x03      |      Number    |      |    0xA5     |      0x21       | File Name Bytes N | File Name String |                                                   

Configure Overlay by file name (for user-defined Overlay)                                                                       

| **Address** | **Instruction** | **Header** |  **Length 8bit**  |    **N bytes**   |                                          
| ----------- | --------------- | ---------- |  ---------------  |  --------------  |                                                             
|    OXA5     |      0x0E       |    0x21    | File Name Bytes N | File Name String |                                                                     


## SPI Interface Specification                                                                                     

|   **Name**   |  **FPGA pins**   |  **Description**                                    |                                                               
| ------------ | ---------------- | ----------------------------------------------------|                 
| FPGA_AR_SCK  |        H13       | SPI interface synchronous clock from Arduino master |                                          
| FPGA_AR_MOSI |        M5        | SPI Interface Master Output Slave Input             |                                                      
| FPGA_AR_MISO |        L5        | SPI Interface Master Input Slave Output             |                                                      
| FPGA_AR_D10  |        B2        | SPI Chip Select Signal                              |                                                                      
| FPGA_AR_OE2  |        M3        | 1: Enable Arduino to communicate with FPGA SPI  0: Disable Arduino to communicate with FPGA SPI     |

As an SPI slave device controlled by the Arduino, the FPGA can send control instructions and acquire data to the on-board FPGA through this interface.

1. SPI mode                        
Mode 0: CPOL = 0, CPHA = 0      
Transmission length: 8 bits     

2. SPI communication protocol definition-instruction                                                                                                                

Arduino instruction format that FPGA Overlay can receive. The instruction starts at 0xAA and ends with a 1-byte checksum:                                           

|  **Frame header (1 Byte)**  |  **Function word (1 Byte)**  |  **Parameter length (1 Byte)**  |  **Parameter (N Bytes)**  |  **Checksum (1 Byte)**  |              
| --------------------------- | ---------------------------- | ------------------------------- | ------------------------- | ----------------------- |
|            0XAA             |                              |                                 |                           |                         |                                                                                                                                                    

**Remarks:** The checksum indicates the frame header, function word, parameter length, and the parameters are accumulated in bytes to take the lower eight digits.

3. SPI Communication Protocol Definition-Data

The data format sent by FPGA Overlay to Arduino is defined as follows. The data starts with 0x55 and the checksum of 1 byte ends:

| **Frame header (1 Byte)** |  **Function word (1 Byte)** | **Data length (1 Byte)** | **Data (N Bytes)** | **Checksum (1 Byte)** |
| ------------------------- | --------------------------- | ------------------------ | ------------------ | --------------------- |
|          0X55             |                             |                          |                    |                       |

**Remarks:** The checksum indicates the frame header, function word, data length, and data are accumulated in bytes and the lower eight digits are taken.

4. SPI communication protocol example

The following protocol provides the Arduino master control interface for the on-board ADC and DAC.

**Instruction format:**

|            Name            | Frame Header (1byte) | Function Word (1byte) | Parameter Length (1byte) |    Parameter N bytes   | Checksum |   Description     |
| -------------------------- | -------------------- | --------------------- | ------------------------ | ---------------------- |--------- |-------------------|
|         Enable DAC         |         0xAA         |          0x01         |           0x00           |          None          |   0xAB   | Enable DAC Output |
|          Stop DAC          |         0xAA         |          0x02         |           0x00           |          None          |   0xAC   | Stop DAC Output  |
|     Configure DAC data     |        0xAA          |          0x03         |           0x02           |                        |    -     | Configure DAC output Byte1: bit3 ~ bit0 as the upper four bits Byte2: bit7 ~ bit0 as the lower eight bits |
| Start / Stop AD Conversion |        0XAA          |          0x04         |           0x01           |     Byte0 = 0 or 1     |    -     | 0: Start AD Conversion 1: Stop AD Conversion                          |
|     Set Sampling Rate      |        0XAA          |          0x05         |           0x01           |    Byte0: 0x01 ~ 0x06  |    -     | 0x01: 1MHz 0x02: 2MHz 0x03: 5MHz 0x04: 8MHz 0x05: 10MHz 0x06: 15MHz   |
|     Set sampling times     |        0XAA          |          0x06         |           0x01           |    Byte0: 0x01 ~ 0x07  |    -     | 0x01: Single 0x02: 1k 0x03: 2k 0x04: 5k 0x05: 10k 0x06: 20k 0x07: 40k |
|    Get Sampling Data       |        0XAA          |          0x07         |           0x01           |    0x01 ~ 0x07         |    -     |                                                                       |

**Data reading:**

|          Name        | Frame header (1byte) | Function word (1byte) | Parameter length (1byte) | Parameter N bytes | Checksum | Description |
| -------------------- | -------------------- | --------------------- | ------------------------ | ----------------- | -------- | ----------- |
| ADC data acquisition |        0x55          |          0x07         |       0x01- 0x07         |         -         |          |             |
