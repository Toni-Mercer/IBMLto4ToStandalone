# IBMLto4ToStandalone
IBM LTO 4 Tape Drive Standalone Configuration Script

## Introduction

This repository contains a Python script and wiring guide to configure an IBM LTO 4 Tape Drive (removed from a tape library) into standalone mode. By sending the correct Set_Config hex payload via RS-422, the drive will initialize its Fibre Channel or SAS data ports without waiting for library robot communication.

Adapter used:
DSD TECH SH-U11 USB to RS485 RS422 Converter with FTDI FT232R Chip Work for Modbus
Can be found [here](https://www.amazon.es/dp/B07B416CPK)
Make sure the chip is `FTDI FT232R`. Do not convert from USB to RS232 and then to RS232 to RS422 or the RX differential circuit will not have enough power to transmit to the LTO drive.

I wrote the code based on the repo from [AC7RNsphnHVbyT4.](https://github.com/AC7RNsphnHVbyT4/ibm-tape-drive-automatic-standalone).


## Wiring

Set all switches to OFF.

My drive had 7 wires and the correct colors where:
Purple   =   GND / Shield
Brown    =   RX+
Red      =   RX-
Orange   =   TX+
Yellow   =   TX-

Because RS 422 is a crossover connection the adapter's Transmit (TX) pins wire directly to the drive's Receive (RX) pins matching the polarity. Therefore;
On the adapter the color schema is:
1: Brown   (Drive RX+ to Adapter TXD+)
2: Red     (Drive RX- to Adapter TXD-)
3: Orange  (Drive TX+ to Adapter RXD+)
4: Yellow  (Drive TX- to Adapter RXD-)
5: Purple  (GND / Shield)



## Script

As [AC7RNsphnHVbyT4](https://github.com/AC7RNsphnHVbyT4/ibm-tape-drive-automatic-standalone) stated on his repo we need to transmit the hex to the drive and wait for a `06 03` response.

The following scripts can be used. The first is a one shot script ran on bash that imports serial and dumps the hex. It won't wait for a response.

### Oneshot

First try, because wiring was incorrect on the TX side it never reached the drive and it kept doing the heartbeat.

```bash
pip3 install pyserial
```

```bash
sudo python3 -c 'import serial; s = serial.Serial("/dev/ttyUSB0", 38400, timeout=1); s.write(b"\x02\x00\x36\xAC\x01\x00\x00\x00\x01\xFF\xF2\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xE6\x03")'
```

### Listen to heartbeat

This second script listens to the drive's heartbeat, catches the exact timing window, and sends the hex payload to set it to standalone.


```python3
import serial
import time

def set_drive_standalone_sync(port='/dev/ttyUSB0', baudrate=38400):
    try:
        ser = serial.Serial(
            port=port,
            baudrate=baudrate,
            parity=serial.PARITY_NONE,
            stopbits=serial.STOPBITS_ONE,
            bytesize=serial.EIGHTBITS,
            timeout=0.1
        )
        print(f"Listening on {port} for drive heartbeat...")
        
        payload = bytes.fromhex(
            "02 00 36 AC 01 00 00 00 01 FF F2 00 00 00 00 00 "
            "00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "
            "00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "
            "00 00 00 00 00 00 00 00 00 00 E6 03"
        )
        
        buffer = bytearray()
        
        while True:
            if ser.in_waiting > 0:
                chunk = ser.read(ser.in_waiting)
                buffer.extend(chunk)
                print(f"Rx: {chunk.hex(' ').upper()}")
                
                if b'\x02\x00\x07\xAB' in buffer:
                    print("\n[+] Config_Request (Heartbeat) detected!")
                    print("[+] Firing Set_Config payload immediately to catch the timing window...")
                    ser.write(payload)
                    
                    time.sleep(0.5)
                    response = ser.read(ser.in_waiting)
                    print(f"Drive Response Rx: {response.hex(' ').upper()}")
                    
                    if b'\x06' in response:
                        print("\n[SUCCESS] Received ACK (0x06)! The drive should now initialize its data ports.")
                    else:
                        print("\n[FAILED] Payload sent, but no ACK received.")
                    break
                    
            time.sleep(0.01)
            
    except serial.SerialException as e:
        print(f"Serial Error: {e}")
    except KeyboardInterrupt:
        print("\nStopping.")
    finally:
        if 'ser' in locals() and ser.is_open:
            ser.close()

if __name__ == '__main__':
    set_drive_standalone_sync('/dev/ttyUSB0')
```

Output:

```bash
sudo python3 ~/Desktop/sync_drive.py 
[sudo] password for toni: 
Listening on /dev/ttyUSB0 for drive heartbeat...
Rx: 02 00 07 AB FF FF 00 00 00 01 01 B3 03

[+] Config_Request (Heartbeat) detected!
[+] Firing Set_Config payload immediately to catch the timing window...
Drive Response Rx: 06 03

[SUCCESS] Received ACK (0x06)! The drive should now initialize its data ports. 
```

