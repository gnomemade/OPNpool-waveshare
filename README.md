# OPNpool on Waveshare ESP32-S3-RS485-CAN

Pool automation for Pentair EasyTouch/IntelliTouch controllers using an [OPNpool](https://github.com/cvonk/OPNpool) ESPHome component on a [Waveshare ESP32-S3-RS485-CAN](https://www.waveshare.com/esp32-s3-rs485-can.htm) board, integrated with Home Assistant.

## Hardware

- **Board:** Waveshare ESP32-S3-RS485-CAN (ESP32-S3, built-in isolated RS485 transceiver)
- **Controller:** Pentair EasyTouch 8 (also supports IntelliTouch, SunTouch)
- **Connection:** RS485 bus via the controller's COM port (J20)

### Wiring

Connect the Waveshare board to the Pentair controller's **COM PORT** (J20 on EasyTouch motherboard):

| Pentair Wire | Color  | Waveshare Terminal |
|--------------|--------|--------------------|
| GND          | Black  | RS485 GND          |
| -DT (Data-)  | Green  | RS485 B (A-)       |
| +DT (Data+)  | Yellow | RS485 A (B+)       |
| +15V         | Red    | **DO NOT CONNECT**  |

> **Important:** Connect to the COM PORT directly, not through a port multiplexer board. The Waveshare board's isolated RS485 interface requires the GND connection for proper signal decoding. Power the Waveshare board separately (USB-C or DC input), not from the Pentair +15V line.

### RS485 Pin Mapping (Waveshare ESP32-S3-RS485-CAN)

| Function | GPIO |
|----------|------|
| RS485 TX | 17   |
| RS485 RX | 18   |
| RS485 DE/RE | 21 |

## Changes from Stock OPNpool

This build modifies [OPNpool](https://github.com/cvonk/OPNpool) for the Waveshare ESP32-S3-RS485-CAN board:

### YAML Configuration (`opnpool-waveshare-s3.yaml`)

| Setting | Stock (ESP32-C6) | This Build (ESP32-S3) |
|---------|-------------------|-----------------------|
| `esp32.variant` | `esp32c6` | `esp32s3` |
| `esp32.board` | `esp32-c6-devkitc-1` | `esp32-s3-devkitc-1` |
| `rs485.tx_pin` | 21 | 17 |
| `rs485.rx_pin` | 22 | 18 |
| `rs485.rts_pin` | 23 | 21 |
| `flash_size` | `8MB` | `16MB` |
| `logger.hardware_uart` | (default) | `USB_SERIAL_JTAG` |
| OpenThread | commented out | removed (ESP32-S3 doesn't support Thread) |

### Code Change (`components/opnpool/pool_task/rs485.cpp`)

One change: UART mode set to `UART_MODE_UART` instead of `UART_MODE_RS485_HALF_DUPLEX`. The Waveshare board's isolated RS485 transceiver works correctly with OPNpool's existing manual GPIO direction control (DE/RE on GPIO21). The ESP-IDF hardware RS485 half-duplex mode is not needed and can interfere with the isolated transceiver.

## Prerequisites

### Docker Host

A Linux server running containers (Docker or Podman) for Home Assistant and ESPHome. This guide uses Podman on Debian 12.

### ESPHome (for compiling firmware)

ESPHome runs as a container alongside Home Assistant:

```bash
podman run -d \
  --name esphome \
  --restart unless-stopped \
  --network host \
  -v /opt/homeassistant/esphome-config:/config \
  -v /etc/localtime:/etc/localtime:ro \
  ghcr.io/esphome/esphome
```

The ESPHome dashboard is available at `http://<host>:6052`.

### Home Assistant

Home Assistant runs as a privileged container (required for Bluetooth/DHCP discovery):

```bash
sudo podman run -d \
  --name homeassistant \
  --restart unless-stopped \
  --network host \
  --privileged \
  -v /opt/homeassistant/config:/config \
  -v /etc/localtime:/etc/localtime:ro \
  -v /run/dbus:/run/dbus:ro \
  ghcr.io/home-assistant/home-assistant:stable
```

Home Assistant UI is available at `http://<host>:8123`.

> **Note:** With rootless Podman, Home Assistant requires `sudo podman` (runs as root) for the `--privileged` flag to work correctly. ESPHome can run as a regular user.

### Container Persistence (Podman)

To ensure containers start on reboot:

**Home Assistant (root):**
```bash
sudo podman generate systemd --name homeassistant --new > /etc/systemd/system/container-homeassistant.service
sudo systemctl daemon-reload
sudo systemctl enable container-homeassistant.service
```

**ESPHome (user):**
```bash
sudo loginctl enable-linger $USER
mkdir -p ~/.config/systemd/user
podman generate systemd --name esphome --new > ~/.config/systemd/user/container-esphome.service
systemctl --user daemon-reload
systemctl --user enable container-esphome.service
```

## Setup

### 1. Clone and Configure

```bash
git clone https://github.com/gnomemade/OPNpool-waveshare.git
cd OPNpool-waveshare

# Create secrets file from example
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with your credentials
```

Generate an API encryption key:
```bash
openssl rand -base64 32
```

### 2. Copy to ESPHome Config Directory

Copy the YAML config, secrets, and components to your ESPHome config directory:

```bash
scp opnpool-waveshare-s3.yaml secrets.yaml <host>:/opt/homeassistant/esphome-config/
scp -r components/ <host>:/opt/homeassistant/esphome-config/
```

### 3. Compile

```bash
# On the Docker host, or via ESPHome dashboard
podman exec esphome esphome compile /config/opnpool-waveshare-s3.yaml
```

### 4. Flash

**First flash (USB):**
```bash
# Install esptool locally
pip install esptool

# Download the factory firmware from the ESPHome build
scp <host>:/opt/homeassistant/esphome-config/.esphome/build/opnpool-1/.pioenvs/opnpool-1/firmware.factory.bin .

# Flash via USB-C
esptool.py --chip esp32s3 --port /dev/cu.usbmodem101 write_flash 0x0 firmware.factory.bin
```

**Subsequent updates (OTA):**
```bash
podman exec esphome esphome run /config/opnpool-waveshare-s3.yaml --device <device-ip> --no-logs
```

### 5. Integrate with Home Assistant

1. Open Home Assistant at `http://<host>:8123`
2. Go to **Settings > Devices & Services > Add Integration**
3. Search for **ESPHome**
4. Enter the device IP address, port `6053`
5. Enter the API encryption key from your `secrets.yaml`

## Testing

A standalone RS485 test config (`rs485-test.yaml`) is included for hardware debugging. It reads raw UART bytes and logs them without the OPNpool protocol layer, useful for verifying RS485 connectivity.

```bash
podman exec esphome esphome run /config/rs485-test.yaml --device <device-ip> --no-logs
```

## Troubleshooting

### No RS485 data received
- Verify wiring: GND, A/B connections to the correct Waveshare terminals
- Confirm the RS485 LED on the Waveshare board is lit (indicates transceiver sees bus activity)
- Connect directly to the controller COM port, not through a port multiplexer
- Try swapping A and B wires (polarity)

### Garbled data / no valid Pentair preambles
- Ensure the Pentair GND (black wire) is connected to the Waveshare **RS485 GND** terminal (isolated side), not the DC power GND
- Check for damaged wiring or multiplexer boards in the signal path

### "Controller address still unknown"
- The device hasn't seen a broadcast from the Pentair controller yet
- Verify the controller is powered on and active
- Check RS485 wiring and signal quality

### Web server / API connection drops
- Reduce log verbosity in the YAML config
- Close extra browser tabs connected to the device
- The ESPHome API supports max 8 simultaneous connections

## Credits

- [OPNpool](https://github.com/cvonk/OPNpool) by Coert Vonk — ESP32 Pentair pool controller integration
- [ESPHome](https://esphome.io/) — firmware framework
- [Home Assistant](https://www.home-assistant.io/) — home automation platform
