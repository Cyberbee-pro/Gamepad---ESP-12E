# ESP-12E (ESP8266) Multiplexed SPI Wireless Gamepad

A low-level bare-metal hardware project implementing a custom wireless gamepad using a resource-constrained ESP-12E module paired with an external **MCP3204 12-Bit SPI Analog-to-Digital Converter (ADC)**. 

Because the ESP8266 lacks native hardware Bluetooth or multi-channel ADCs, this build establishes direct register-level SPI communications to acquire data and serializes states over ultra-low-overhead Wi-Fi UDP datagrams.

## System Architecture

* **External SPI Multiplexing:** Offloads analog tracking to a dedicated MCP3204 IC over the high-speed hardware SPI (HSPI) bus, guaranteeing crisp 12-bit resolution ($0$ to $4095$) entirely unaffected by Wi-Fi module power draw or noise.
* **Custom Bit-Shift Driver:** Firmware executes manual bit-shifting sequences over the `MOSI`/`MISO` lines to pass channel configuration bytes and pull back raw binary axis payloads.
* **Low-Latency Wi-Fi UDP Pipeline:** Serializes state configurations into a tiny binary array payload and blasts non-blocking UDP packets over the local network straight to a desktop background server.

## Hardware Interconnect Protocol

The ESP-12E pins must be strictly routed to the MCP3204 control registers. All digital input switches leverage physical $10\text{k}\Omega$ pull-down resistors.

## Hardware Interconnect Protocol

The ESP-12E pins must be strictly routed to the MCP3204 control registers. All digital input switches leverage physical 10kΩ pull-down resistors.

```
text
  +-----------------------+              +-----------------------+
  |   ESP8266 (ESP12E)    |              |     MCP3204 (ADC)     |
  |                       |              |                       |
  |      GPIO14 (HSPI_CLK)|------------->| CLK (Pin 11)          |
  |     GPIO12 (HSPI_MISO)|<-------------| DOUT (Pin 12)         |
  |     GPIO13 (HSPI_MOSI)|------------->| DIN (Pin 13)          |
  |       GPIO15 (HSPI_CS)|------------->| CS/SHDN (Pin 14)      |
  +-----------------------+              +-----------------------+
                                           |      |      |      |
                                          CH0    CH1    CH2    CH3
                                         (Pin1) (Pin2) (Pin3) (Pin4)
                                           |      |      |      |
                                           +------+      +------+
                                              |             |
                                        [Left Stick]  [Right Stick]
                                          X / Y          X / Y
```
## SPI Bus Packet Structure

To parse a single analog channel, the firmware pulls the `CS` line LOW and clocks a 3-byte data structure across the SPI bus:
1. **Byte 1:** Start bit configuration.
2. **Byte 2:** Channel selection mode (`D2`, `D1`, `D0` bits set to target `CH0` through `CH3` via single-ended sampling).
3. **Byte 3:** Clocking out the remaining cycles to latch the returning 12-bit response stream ($B_{11}$ down to $B_0$).

## Host Desktop Software Interface

To use this controller on a PC:
1. Run the background Python/C++ listener server provided in the `/server` directory of this repository.
2. The server binds to an incoming UDP socket port, receives the serialized controller state array, parses the axis positions, and writes them straight to a virtual device driver interface (e.g., **ViGEmBus** or **vJoy**).

---
Maintained under the MIT License.