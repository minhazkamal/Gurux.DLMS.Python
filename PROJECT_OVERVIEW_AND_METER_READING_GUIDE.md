# Gurux DLMS Python Project Overview and Meter Reading Guide

## What This Project Is

This repository is the Python implementation of Gurux DLMS/COSEM. DLMS/COSEM is a protocol commonly used to communicate with electricity, gas, and water meters.

The project has two different roles:

- `Gurux.DLMS.python` is the reusable DLMS protocol library.
- The other folders are example applications that show how to use the library to read meters, execute XML-defined DLMS commands, or listen for push notifications.

The core library does not own the physical connection to the meter. It builds DLMS frames, parses replies, manages COSEM objects, handles authentication/ciphering, and translates DLMS messages. TCP/IP or serial communication is handled by companion Gurux media libraries such as `gurux_net` and `gurux_serial`.

## Repository Layout

```text
Gurux.DLMS.Python/
  README.md
  Gurux.DLMS.python/
    setup.py
    gurux_dlms/
      GXDLMSClient.py
      GXDLMSServer.py
      GXDLMS.py
      GXDLMSTranslator.py
      objects/
      enums/
      secure/
      asn/
      ecdsa/
  Gurux.DLMS.Client.Example.python/
    main.py
    GXSettings.py
    GXDLMSReader.py
    requirements.txt
  Gurux.DLMS.XmlClient.python/
    main.py
    GXSettings.py
    GXDLMSReader.py
    Messages/
  Gurux.DLMS.Push.Listener.Example.python/
    main.py
    GXSettings.py
```

## Main Components

### Core Library

Path:

```text
Gurux.DLMS.python/gurux_dlms
```

Important classes:

- `GXDLMSClient`: client-side DLMS engine. Creates SNRM, AARQ, read, write, method, disconnect, and receiver-ready messages.
- `GXDLMSServer`: server-side DLMS engine for simulator/server implementations.
- `GXDLMS`: low-level frame and PDU handling.
- `GXDLMSTranslator`: converts DLMS frames/PDU data to and from XML.
- `GXReplyData`: stores parsed reply data.
- `GXByteBuffer`: byte buffer helper.
- `objects/`: COSEM object model, such as register, clock, profile generic, disconnect control, security setup, etc.
- `secure/`: secure client/server variants.

### Client Example

Path:

```text
Gurux.DLMS.Client.Example.python
```

This is the best starting point for reading a real meter.

Important files:

- `main.py`: application entry point.
- `GXSettings.py`: parses command-line parameters and configures client/media/security.
- `GXDLMSReader.py`: performs the actual meter communication flow.

### XML Client

Path:

```text
Gurux.DLMS.XmlClient.python
```

This sends DLMS commands described as XML files. It is useful for testing specific meter requests without changing Python source code.

### Push Listener

Path:

```text
Gurux.DLMS.Push.Listener.Example.python
```

This waits for unsolicited push, event, or notification messages from a meter and decodes them.

## How Meter Reading Works

The normal client flow is:

1. Parse command-line settings.
2. Create a DLMS client.
3. Open TCP/IP or serial media.
4. If needed, perform IEC optical Mode E startup.
5. Send `SNRM` to initialize HDLC communication.
6. Parse the meter's `UA` response.
7. Send `AARQ` to create the DLMS application association.
8. Parse the meter's `AARE` response.
9. If high authentication is used, perform application association authentication.
10. Read association view to discover meter COSEM objects.
11. Read selected OBIS attributes or read all available objects.
12. Send release/disconnect.
13. Close the media.

The key implementation is in:

```text
Gurux.DLMS.Client.Example.python/GXDLMSReader.py
```

Look especially at:

- `initializeConnection`
- `read`
- `readList`
- `getAssociationView`
- `readAll`
- `close`

## Using This Source Without Installing `gurux-dlms`

You can use this checkout directly by putting the source package on `PYTHONPATH`.

PowerShell example:

```powershell
$env:PYTHONPATH = "E:\GitHub\Gurux.DLMS.Python\Gurux.DLMS.python"
```

However, the client example also imports these companion packages:

```text
gurux_common
gurux_net
gurux_serial
```

Those packages are not included in this repository. If you do not want prebuilt packages, get their source repositories too and add them to `PYTHONPATH`.

Expected source layout example:

```text
E:\GitHub\
  Gurux.DLMS.Python\
  gurux.common.python\
  gurux.net.python\
  gurux.serial.python\
```

Then set `PYTHONPATH` to include all source roots:

```powershell
$env:PYTHONPATH = @(
  "E:\GitHub\Gurux.DLMS.Python\Gurux.DLMS.python",
  "E:\GitHub\gurux.common.python",
  "E:\GitHub\gurux.net.python",
  "E:\GitHub\gurux.serial.python"
) -join ";"
```

If the companion repositories keep their import package inside a nested project folder, point `PYTHONPATH` at that folder instead. The correct folder is the one that directly contains package directories such as `gurux_common`, `gurux_net`, or `gurux_serial`.

## Python Version

The current `python` on this machine is:

```text
Python 3.7.0
```

This checkout contains Python `match` syntax, so Python 3.7 cannot import it. Use Python 3.10 or newer.

Check your active version:

```powershell
python --version
```

If you install another Python version on Windows, you can usually run it with:

```powershell
py -3.10 --version
py -3.11 --version
py -3.12 --version
```

## Minimal Environment Check

After setting `PYTHONPATH`, verify that the source package imports:

```powershell
py -3.12 -c "import gurux_dlms; print(gurux_dlms.name)"
```

Expected output:

```text
gurux_dlms
```

Then verify the example dependencies:

```powershell
py -3.12 -c "import gurux_common, gurux_net, gurux_serial; print('media packages OK')"
```

## Information You Need From The Meter

Before reading a meter, collect these values from the meter vendor, utility, meter manual, or a known working configuration:

- Transport: TCP/IP, serial optical head, RS-485, modem, etc.
- TCP host/IP and port, or serial port settings.
- Referencing mode: `ln` for Logical Name or `sn` for Short Name.
- Interface type: usually `HDLC` or `WRAPPER`.
- Client address, often `16`.
- Server address, often `1`, but manufacturer-specific.
- Authentication mode: `None`, `Low`, `High`, `HighGMac`, etc.
- Password if authentication is used.
- Security level if ciphering is used.
- System title, meter system title, authentication key, block cipher key, and invocation counter OBIS if secure communication is used.
- OBIS codes and attribute indexes you want to read.

Without correct addressing, authentication, and security parameters, the meter will reject communication.

## Read A Meter Over TCP/IP

Go to the normal client example:

```powershell
cd E:\GitHub\Gurux.DLMS.Python\Gurux.DLMS.Client.Example.python
```

Set source paths:

```powershell
$env:PYTHONPATH = @(
  "E:\GitHub\Gurux.DLMS.Python\Gurux.DLMS.python",
  "E:\GitHub\gurux.common.python",
  "E:\GitHub\gurux.net.python",
  "E:\GitHub\gurux.serial.python"
) -join ";"
```

Read all available values using Logical Name referencing:

```powershell
py -3.12 main.py -h 192.168.1.100 -p 4059 -r ln -c 16 -s 1 -t Info
```

Use `Verbose` trace when debugging:

```powershell
py -3.12 main.py -h 192.168.1.100 -p 4059 -r ln -c 16 -s 1 -t Verbose
```

Use WRAPPER instead of HDLC if the meter expects DLMS TCP wrapper framing:

```powershell
py -3.12 main.py -h 192.168.1.100 -p 4059 -i WRAPPER -r ln -c 16 -s 1 -t Info
```

## Read A Specific OBIS Attribute

Use `-g` with this format:

```text
"OBIS_CODE:ATTRIBUTE_INDEX"
```

Example: read total active energy import. For many meters this is OBIS `1.0.1.8.0.255`, attribute `2`.

```powershell
py -3.12 main.py -h 192.168.1.100 -p 4059 -r ln -c 16 -s 1 -g "1.0.1.8.0.255:2"
```

Read several attributes:

```powershell
py -3.12 main.py -h 192.168.1.100 -p 4059 -r ln -c 16 -s 1 -g "0.0.1.0.0.255:2;1.0.1.8.0.255:2"
```

Common examples:

```text
0.0.1.0.0.255:2     Clock time
0.0.96.1.0.255:2    Logical device name
1.0.1.8.0.255:2     Active energy import total
1.0.2.8.0.255:2     Active energy export total
1.0.32.7.0.255:2    Voltage L1
1.0.31.7.0.255:2    Current L1
```

OBIS availability is meter-specific. The association view tells you which objects your meter actually exposes.

## Cache Association View

Reading association view can be slow. Save it to a local XML file:

```powershell
py -3.12 main.py -h 192.168.1.100 -p 4059 -r ln -c 16 -s 1 -o meter-cache.xml
```

Later reads can reuse it:

```powershell
py -3.12 main.py -h 192.168.1.100 -p 4059 -r ln -c 16 -s 1 -o meter-cache.xml -g "1.0.1.8.0.255:2"
```

## Read Over Serial

Example with explicit serial settings:

```powershell
py -3.12 main.py -S COM1:9600:8None1 -r ln -c 16 -s 1 -t Info
```

With low authentication:

```powershell
py -3.12 main.py -S COM1:9600:8None1 -r ln -c 16 -s 1 -a Low -P password -t Info
```

With optical IEC Mode E startup:

```powershell
py -3.12 main.py -S COM1 -i HdlcWithModeE -r ln -c 16 -s 1 -t Verbose
```

## Secure Meter Example

For encrypted/authenticated communication, you must know the exact security values for the meter.

Example shape:

```powershell
py -3.12 main.py `
  -h 192.168.1.100 `
  -p 4059 `
  -r ln `
  -c 16 `
  -s 1 `
  -a HighGMac `
  -C AuthenticationEncryption `
  -T 4775727578313233 `
  -M 4D45544552303132 `
  -A D0D1D2D3D4D5D6D7D8D9DADBDCDDDEDF `
  -B 000102030405060708090A0B0C0D0E0F `
  -v 0.0.43.1.0.255 `
  -t Verbose
```

Do not reuse those key values unless they are actually your meter's configured keys. They are only placeholders.

## Useful Command-Line Options

```text
-h    Host name or IP address.
-p    TCP port.
-S    Serial port. Example: COM1 or COM1:9600:8None1.
-r    Referencing mode: ln or sn.
-i    Interface type: HDLC, WRAPPER, HdlcWithModeE, Plc, PlcHdlc.
-c    Client address.
-s    Server address.
-n    Server address calculated from serial number.
-a    Authentication: None, Low, High, HighMd5, HighSha1, HighGMac, HighSha256, HighEcdsa.
-P    Password. Use 0x prefix for hex password.
-g    Selected object reads. Example: "1.0.1.8.0.255:2".
-o    Association view cache file.
-t    Trace level: Off, Error, Warning, Info, Verbose.
-C    Security level: None, Authentication, Encryption, AuthenticationEncryption.
-V    Security suite: Suite0, Suite1, Suite2.
-T    Client system title in hex.
-M    Meter system title in hex.
-A    Authentication key in hex.
-B    Block cipher key in hex.
-D    Dedicated key in hex.
-v    Invocation counter object logical name.
-d    Standard: DLMS, India, Italy, SaudiArabia, IDIS.
```

## Troubleshooting

If import fails with a syntax error around `match`, use Python 3.10 or newer.

If `gurux_common`, `gurux_net`, or `gurux_serial` cannot be imported, add their source repositories to `PYTHONPATH` or install them.

If the meter does not respond, check host/port or serial port settings first.

If the meter rejects association, check client address, server address, referencing mode, authentication, and interface type.

If secure communication fails, check security suite, invocation counter, system titles, authentication key, and block cipher key.

If reads fail after connection succeeds, read association view first and confirm the OBIS code and attribute index exist on that meter.

Use verbose trace to inspect raw TX/RX frames:

```powershell
py -3.12 main.py -h 192.168.1.100 -p 4059 -r ln -c 16 -s 1 -t Verbose
```

## Recommended First Test

Once Python 3.10+ and all source dependencies are available, run this first:

```powershell
cd E:\GitHub\Gurux.DLMS.Python\Gurux.DLMS.Client.Example.python

$env:PYTHONPATH = @(
  "E:\GitHub\Gurux.DLMS.Python\Gurux.DLMS.python",
  "E:\GitHub\gurux.common.python",
  "E:\GitHub\gurux.net.python",
  "E:\GitHub\gurux.serial.python"
) -join ";"

py -3.12 main.py -h YOUR_METER_IP -p YOUR_METER_PORT -r ln -c 16 -s 1 -t Verbose -o meter-cache.xml
```

If that succeeds, read one known value:

```powershell
py -3.12 main.py -h YOUR_METER_IP -p YOUR_METER_PORT -r ln -c 16 -s 1 -o meter-cache.xml -g "0.0.1.0.0.255:2"
```
