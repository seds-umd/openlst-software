# UMD OpenLST Software

This is a modified version of the software for [Planet Labs' OpenLST](https://github.com/OpenLST/openlst).

Changes include:
* Restructured build process to avoid using Vagrant and simplify makefiles
* TODO: Rewrite python interface to be simpler
* TODO: Add commands to change modulation, data rate, output power
* TODO: Code for SDR to talk to OpenLST

## Setup

Install required packages

```bash
sudo apt install build-essential pkg-config libusb-1.0-0-dev sdcc cc-tool
```

Set up Python environment (must have Python 3.6+ installed)

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install -e .
```

Build bootloader and application

```bash
cd openlst
make all
```

Flash bootloader and application. Change HWID from 1234 to something else. Add signing keys for flight builds.

```bash
source venv/bin/activate
./openlst_tools/flash.py openlst/build 1234
./openlst_tools/bootload.py --port /dev/ttyUSB0 1234 openlst/build/radio.hex
```

## Python Interface

To launch the python interface, run (replacing ttyUSB0 with the correct serial port and 7001 with the HWID of the connected OpenLST):

```bash
./openlst_tools/openlst.py --port /dev/ttyUSB0 7001
```

This creates an IPython shell . Some of the available commands:

```python
openlst.reboot() # Reboots OpenLST
openlst.transmit(b"Hello there", dest_hwid=0x1234) # Transmit message over RF
openlst.get_time() # Get time since J2000
openlst.set_time() # Sets time using computer time
openlst.get_telem() # Gets telemetry
openlst.set_rf_params(
    frequency=437e6,
    chan_bw=60268,
    drate=7416,
    deviation=3707,
    power=0x12
) # Set RF parameters (example values are defaults)
```

## File Structure

* `ground` - Python interface for controlling OpenLST over UART
* `openlst` - C firmware for CC1110
    * `board` - Board specific files
    * `bootloader` - Bootloader code
    * `build` - Directory created during build process to hold all build artifacts
    * `common` - Files used by both bootloader and application
    * `radio` - Main application

## RF Parameters

### Data Rate

Page 191, Section 13.5

### Receiver Channel Filter Bandwidth

Page 191, Section 13.6

### Modulation Formats

Page 196, Section 13.9

DEVIATION_M, DEVIATION_E

### Frequency

Page 205, Section 13.3

* Base frequency set using FREQ2, FREQ1, FREQ0
* 8 bit channel selector and CHANSPC settings are used to choose specific channel relative to base frequency
* Must be changed while radio is in idle state

### Output Power

Page 207, Section 13.15

* Table with list of settings for given frequencies and power settings
* TODO: Test these values and other values and see what happens

## UART Protocol

Multi-byte fields are all least significant byte first

| Field       | Length | Purpose                                         |
| ----------- | ------ | ----------------------------------------------- |
| Start bytes | 2      | Denotes start of frame using bytes [0x22, 0x69] |
| Length | 1 | Number of bytes in packet, not including start bytes and length byte. Must be greater than 0 and less than 251. |
| HWID | 2 | ID of destination node |
| Sequence number | 2 | Increments by one after each command |
| System | 1 | Must be the same as `MSG_TYPE_RADIO_IN` |
| Command | 1 | Determine how the rest of the packet is interpreted |
| Data | N | Data bytes |

### Default Commands

#### 0x00 - BOOTLOADER_PING

Ping bootloader. This returns a BOOTLOADER_ACK and resets bootloader watchdog.

#### 0x01 - BOOTLOADER_ACK

ACK returned by bootloader.

#### 0x02 - BOOTLOADER_WRITE_PAGE

Write data to application flash. Data is structured as:

```c
typedef struct {
	uint8_t flash_page;
	uint8_t page_data[FLASH_WRITE_PAGE_SIZE];
} msg_bootloader_write_page_t;
```

#### 0x0C - BOOTLOADER_ERASE

Erase application flash sections.

#### 0x10 - ACK

#### 0xFF - NACK

#### 0x11 - ASCII

#### 0x12 - Reboot

Reboot OpenLST. Data field is either empty for immediate reboot or `uint32_t` representing number of seconds until reboot.

#### 0x13 - Get Time

Replies with 0x14 Set Time containing current internal time, or NACK if no time is set.

#### 0x14 - Set Time

Sets internal time. First 4 bytes are seconds, second 4 bytes are nanoseconds. Replies with ACK.

#### 0x15 - Ranging

TBD

#### 0x16 - Ranging ACK

TBD

#### 0x17 - Get Telem

Requests OpenLST telemetry. Reply is 0x18 Telem.

#### 0x18 - Telem

Struct containing:
* `uint8_t reserved` - Always 0
* `uint32_t uptime` - Uptime in ms
* `uint32_t uart0_rx_count` - Number of packets received on UART0
* `uint32_t uart1_rx_count` - Number of packets received on UART1
* `uint8_t rx_mode` - RX radio mode
* `uint8_t tx_mode` - TX radio mode
* `int16_t adc[ADC_NUM_CHANNELS]` - 8 ADC channels, temperature sensor, VDD/3
* `int8_t last_rssi` - RSSI of last packet. See page 198 of CC1110 datasheet to convert to dBm.
* `uint8_t last_lqi` - Link quality indicator of last packet. Relative indicator of how easily the signal was demodulated. High value means better link. Only comparable if RF settings are identical.
* `int8_t last_freqest` - Estimated frequency offset of carrier.
* `uint32_t packets_sent` - Number of packets sent.
* `uint32_t cs_count` - Carrier sense?
* `uint32_t packets_good` - Number of successfully received packets.
* `uint32_t packets_rejected_checksum` - Number of packets rejected due to CRC failure.
* `uint32_t packets_rejected_reserved` - Always 0.
* `uint32_t packets_rejected_other` - Number of packets rejected due to missing data in header.
* `uint32_t reserved0`
* `uint32_t reserved1`
* `uint32_t custom0`
  * [7:0] - continuous RSSI measurement
* `uint32_t custom1` - Start of frame detection count

#### 0x19 - Get Callsign

#### 0x1A - Set Callsign

#### 0x1B - Callsign

### Custom Commands

Planned commands to be added.

#### 0x80 - Set RF Parameters

Struct containing:
* `uint32_t FREQ`
  * [7:0] - FREQ0
  * [15:8] - FREQ1
  * [23:16] - FREQ2
* `uint8_t FSCTRL0`
* `uint8_t FSCTRL1`
* `uint8_t RF_CHAN_BW`
  * [1:0] - CHANBW_M
  * [3:2] - CHANBW_E
* `uint8_t RF_DRATE_E`
  * [3:0] - DRATE_E
* `uint8_t RF_DRATE_M`
  * [7:0] - DRATE_M
* `uint8_t RF_DEVIATN`
  * [2:0] - DEVIATN_M
  * [6:4] - DEVIATN_E
* `uint8_t PA_CONFIG0`

#### 0x81 - Get RF Parameters

TODO

#### 0x82 - Set Bypass

Single byte, 1 to bypass LNA+PA, 0 to not bypass LNA+PA.
