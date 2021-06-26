# SNI - Super Nintendo Interface

SNI is an interface that allows multiple concurrent applications to access
various kinds of Super Nintendo devices connected to the computer.

SNI is cross-platform and works equally well on Windows, MacOS, and Linux
computers.

SNI is designed and implemented by **jsd1982** and was started in May 2021.

# For End Users

Simply start `sni.exe` and leave it running.

Once started, a systray icon will appear for Windows and Mac users; however,
there are currently no useful menu options for end users other than Quit.
SNI is intended to be easy to use with little to no direct user interaction.
It should always Just Work™.

Currently supported SNES devices are:

* FX Pak Pro a.k.a. SD2SNES hardware USB-enabled SNES cartridge with usb2snes-compatible firmware
* Lua Bridge compatible emulators e.g. Snes9x-rr, BizHawk
* RetroArch with bsnes-mercury emulator core

## Configuration

All configuration options are exposed via environment variables. At start-up,
SNI logs details about the environment variables that it reads and the defaults
it assumes if they are not set.

The following environment variables are defined:

| Name | Default | Purpose |
| --- | --- | --- |
| SNI_GRPC_LISTEN_HOST | 0.0.0.0 | host to listen on for gRPC connections |
| SNI_GRPC_LISTEN_PORT | 8191 | port to listen on for gRPC connections |
| SNI_FXPAKPRO_DISABLE | 0 | fxpakpro: set to 1 to disable FX Pak Pro driver |
| SNI_RETROARCH_DISABLE | 0 | retroarch: set to 1 to disable Retroarch driver |
| SNI_RETROARCH_HOSTS | localhost:55355 | retroarch: list of comma-delimited host:port pairs to detect retroarch instances on; configure these with `network_cmd_port` setting in `retroarch.cfg` |
| SNI_LUABRIDGE_LISTEN_HOST | 127.0.0.1 | luabridge: host/IP to listen on |
| SNI_LUABRIDGE_LISTEN_PORT | 65398 | luabridge: port number to listen on |

## Log Files

SNI logs important activity to a log file found in your system's temporary
folder.

On Windows, this temporary folder is `%APPDATA%\Local\Temp`.

On MacOS, this temporary folder may be `/var/folders/1q/.../T/`.

During start-up, in the console window, SNI will output where the current log
file is located at:
```
2021/06/25 22:45:57.049004 logging to '/var/folders/1q/0q0chpwx4j795vg322m8y20rcdxpg6/T/sni-2021-06-25T17-45-57-048Z.log'
```

# For Developers

SNI offers a [gRPC](https://grpc.io/) API as its primary means of communication
with application clients.

## Design Goals

The gRPC protocol is entirely **stateless** and every request is paired with a
response.

Clients do not "bind" to any specific device and are instead free to make
requests of any device connected to the system at any time.

All gRPC methods may be invoked concurrently with all other gRPC methods.
If you find a scenario where this is _not_ the case, I urge you to file an
[issue report](https://github.com/alttpo/sni/issues/new).

A client's connection to the gRPC server does NOT indicate any device-specific
connection has been established. Device connections are only established between
the SNI service and the device itself and these connections are made on-demand
when clients make requests. Device connections are maintained until an
unrecoverable device error is encountered when talking with the device. When a
device error is encountered, the device connection is immediately closed and
should be re-established with the next client request. This is done to attempt
to keep the device in a consistent and usable state at all times and to never
get "stuck" in a useless state where no requests can be satisfied.

## Generating Client Code
To get started, choose your favorite programming language and use grpc's
`protoc` tool to generate client code [using the provided sni.proto file](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto).

There is plenty of existing documentation on grpc's tooling, so we consider such
documentation out of the scope of SNI's documentation.

A great UI tool for ad-hoc testing of gRPC services is [grpcui](https://github.com/fullstorydev/grpcui).

To use `grpcui`, invoke it like this on the command line:

```grpcui -plaintest -listen 8192 localhost:8191```

SNI has grpc reflection enabled to allow using such ad-hoc testing tools as `grpcui`.

SNI also only exposes the "insecure" grpc protocol and does not make use of TLS
because of the need for low latency.

## gRPC Foreword

In this documentation we'll only refer to the gRPC services, methods, and messages
as they are defined by the [sni.proto](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto) service definition file.
It would not make much sense to discuss gRPC details in terms of any one
specific programming language's generated gRPC client API, so we avoid doing so.
The gRPC client APIs generated by `protoc` can and do look very different
depending on the programming language and framework generated for and their
respective naming conventions and best practices.

## gRPC Services

In gRPC terms, a "service" is simply a collection of related methods. One can
think of it as somewhat analogous to a "class" in object-oriented terms.

SNI offers three primary gRPC services:
* `Devices`
* `DeviceMemory`
* `DeviceControl`

Let's start with the [Devices](#devices) service as it serves as the main entry
point to SNI.

### Devices

The `Devices` service exposes a single method:

#### [ListDevices](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto#L7)

This method allows us to query the system and detect the currently connected
SNES devices.

[Request](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto#L53):
```grpc
  // optional list of device kind filters
  repeated string kinds = 1;
```

As part of the request, we can filter on specific `kind`s of devices if we
want, or just leave the field empty to request all devices.

[Response](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto#L58):
```grpc
  message Device {
    // URI that describes exactly how to connect to the device, e.g.:
    // RetroArch:  "ra://127.0.0.1:55355"
    // FX Pak Pro: "fxpakpro://./dev/cu.usbmodemDEMO000000001" (MacOS)
    //             "fxpakpro://./COM4"                        (Windows)
    // uri is used as the unique identifier of the device for clients to refer to
    string uri = 1;
    // friendly display name of the device
    string displayName = 2;
    // device kind, e.g. "fxpakpro", "retroarch", "lua"
    string kind = 3;
    // all device capabilities:
    repeated DeviceCapability capabilities = 4;
    // default address space for the device:
    AddressSpace defaultAddressSpace = 5;
  }

  repeated Device devices = 1;
```

The response contains a repeated list of `Device`s.

The ideal user experience would be to provide the end user with the list of
devices detected and let them select the best device to use for their gaming
session. This list should be refreshed periodically to accommodate for devices
being connected and disconnected at will.

`ListDevices` must never interfere with any other service call, so it should
always be safe to call at any point in time.

### Device URIs

A `Device` is uniquely identified by its `uri`. A Device URI always contains
enough information to uniquely identify the device it represents. Example
Device URIs are as follows:

* `ra://127.0.0.1:55355` (RetroArch instance)
* `fxpakpro://./COM4` (FX Pak Pro on Windows)
* `fxpakpro://./dev/cu.usbmodemDEMO000000001` (FX Pak Pro on MacOS)
* `luabridge://127.0.0.1:50996` (Lua Bridge client)

These URIs are NOT URLs; they have no meaning outside the SNI system. They
are arbitrarily constructed by SNI to be used as device identifiers. End users
cannot "click" on them and expect any useful action to occur. Ideally, end users
should never see a Device URI in an application's UI except perhaps in a log
file or console window.

Generally, the URI scheme (e.g. `ra`, `fxpakpro`) indicates the SNI driver used
to connect to the device. The URI host:port or the path component is then used
to uniquely identify the device, depending on its kind. If only a path is
required, then a `.` is used for the host:port to indicate a local device and
also to avoid URI parsing ambiguities.

Device URIs SHOULD NOT be hard-coded nor constructed dynamically by application
code. Most Device URIs are NOT guaranteed to be predictable due to several
external factors.

As an example, when an FX Pak Pro device is connected it may be assigned a random
COM port number on Windows systems e.g. COM3, COM4, COM8. This random assignment
is of course not predictable and would be very difficult to hard-code or assume
as a default.

Another counter example is the URIs for the Lua Bridge driver. Since SNI acts as
a server, and the lua script as a client, the URIs that SNI generates
represent the host:port of the _remote_ side of the TCP connection from SNI's
perspective as a server. These remote port numbers are randomly assigned by the
OS and are unpredictable.

For these reasons and more, it is best to avoid hard-coding Device URIs in
your application and instead always rely on the `ListDevices` method to return
the known URIs of currently connected devices.

However, there are certain cases where hand-crafting your own Device URIs may
be your only option where device detection would otherwise fail. For example,
you could have a RetroArch instance that is NOT running on the local system and
thus cannot be easily detected by `ListDevices`. For scenarios like these,
specific to the retroarch driver, the environment variable `SNI_RETROARCH_HOSTS`
is read on SNI start-up to allow for custom endpoints to be scanned to look for
RetroArch instances.

#### DisplayName
Each `Device` has a `displayName` field that can be presented to an end user.

#### Kind
The `Device`'s `kind` field can be used to group devices together which are
detected by the same driver. There are only a handful of distinct `kind` values
which correspond with the internal driver that the device is detected by.
These `kind` values are:

* `fxpakpro`
* `luabridge`
* `retroarch`

#### Capabilities
Each `Device` has a list of `capabilities` that it supports. This list determines
what methods can be called on the `DeviceMemory` service and the `DeviceControl`
service.

Currently defined capabilities are:
```grpc
// capabilities of a SNES device
enum DeviceCapability {
  None = 0;
  ReadMemory = 1;
  WriteMemory = 2;
  ExecuteASM = 3;
  ResetSystem = 4;
  PauseUnpauseEmulation = 5;
  PauseToggleEmulation = 6;
}
```

The `ReadMemory` capability grants usage of the `DeviceMemory` service's
`SingleRead`, `MultiRead`, and `StreamRead` methods.

The `WriteMemory` capability grants usage of the the `DeviceMemory` service's
`SingleWrite`, `MultiWrite`, and `StreamWrite` methods.

The `ResetSystem` capability grants usage of the `DeviceControl` service's
`ResetSystem` method.

The `PauseUnpauseEmulation` capability grants usage of the
`DeviceControl.PauseUnpauseEmulation` method.

The `PauseToggleEmulation` capability grants usage of the
`DeviceControl.PauseToggleEmulation` method.

There are two kinds of Pause capabilities (mainly for emulators) due to the
two kinds of commonly available yet incompatible pause control systems:
explicit pause vs. unpause, and toggling of the pause state without feedback.

### DeviceMemory Service

The memory access subsystem of SNI is designed to allow for flexibility in the
way memory addresses are specified to and translated by SNI.

For every memory access request (reads and writes), an address tuple must be specified
that represents a memory location:

* The 24-bit address value e.g. $7E0010, $F50010, $00FFB0
* The address space the address value is interpreted in e.g. FX Pak Pro, SNES A-bus, Raw
* The memory mapping mode of the ROM currently loaded e.g. LoROM, HiROM, ExHiROM

When a memory request is handled by SNI, the request address tuple is translated
into a device address tuple. The device address tuple is used to specify the
address to the specific device.

The request address space does not have to be the same as the device address
space. SNI knows how to translate between the available address spaces using
the values provided in the request address tuple. This feature allows
developers to select a single consistent address space to specify all memory
addresses with and let SNI translate as necessary.

The memory mapping mode is only required when translating between different
address spaces.

Let's define the concept of an **address space**. Memory addresses may be specified
in one of three address spaces:

* [FX Pak Pro address space](#fx-pak-pro-address-space)
* [SNES A-bus address space](#snes-a-bus-address-space)
* [Raw address space](#raw-address-space)

#### FX Pak Pro Address Space
The FX Pak Pro address space presents a 24-bit custom mapping where the various
kinds of SNES memory are linearly mapped to specific address ranges:

```
$00_0000..$DF_FFFF =   ROM contents, linearly mapped
$E0_0000..$EF_FFFF =  SRAM contents, linearly mapped
$F5_0000..$F6_FFFF =  WRAM contents, linearly mapped
$F7_0000..$F7_FFFF =  VRAM contents, linearly mapped
$F8_0000..$F8_FFFF =   APU contents, linearly mapped
$F9_0000..$F9_01FF = CGRAM contents, linearly mapped
$F9_0200..$F9_041F =   OAM contents, linearly mapped
$F9_0420..$F9_04FF =  MISC contents, linearly mapped
$F9_0500..$F9_06FF =         PPUREG, linearly mapped
$F9_0700..$F9_08FF =         CPUREG, linearly mapped
```

This mapping takes its name from the FX Pak Pro / SD2SNES cart. The cart
monitors all memory access exposed over the SNES memory bus and records the
data read from or written to each SNES memory chip into its own static RAM
chip found inside the cart. The address ranges above are the static RAM chip's
addresses used to store the memory data that was intercepted.

The FX Pak Pro SNI driver natively uses this address space and all requests
made to it are translated into this address space.

#### SNES A-bus Address Space
The SNES A-bus is the primary memory bus that SNES code deals with. If you
are familiar with SNES development, this address space is probably the most
natural to you.

Depending on the cart type, this address space is generally limited to
accessing only ROM, WRAM, and SRAM.

The exact address ranges and their interpretation depends on the memory mapping
mode of the ROM e.g. LoROM, HiROM, or ExHiROM.

The RetroArch SNI driver assumes most emulator cores expose the SNES A-bus
address space. This is true for the bsnes-mercury core at least. Unfortunately,
RetroArch provides no ability via network commands to detect which emulator
core is running let alone determine the address space the core uses. It is
therefore impossible for SNI to determine how best to translate addresses
for RetroArch in general.

In this case, it is advisable for the application to offer the end user a
manual selection of which core is running and then use the Raw address space
for memory requests so that SNI does not translate the address mistakenly.

#### Raw Address Space
The Raw address space serves as an escape mechanism to allow developers
complete control over the address values submitted to the underlying device.
When SNI sees a request with a raw address space, no address translation is
performed; the request address value is handed directly to the device as-is.

#### [SingleRead](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto#L117) method

Reads a single memory segment with a given size from the given device.

[Request](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto#L53):
```grpc
message SingleReadMemoryRequest {
  string uri = 1;
  ReadMemoryRequest request = 2;
}
message ReadMemoryRequest {
  uint32        requestAddress = 1;
  AddressSpace  requestAddressSpace = 2;
  MemoryMapping requestMemoryMapping = 4;

  uint32 size = 3;
}
```

The `uri` is required to identify the device to read memory from.

The `request` contains the memory details. It may seem unnecessary to split
the request into two messages but makes sense from a reuse standpoint when we
consider the `MultiReadRequest` method.

[Response]():
```grpc
message SingleReadMemoryResponse {
  string uri = 1;
  ReadMemoryResponse response = 2;
}
message ReadMemoryResponse {
  uint32        requestAddress = 1;
  AddressSpace  requestAddressSpace = 2;
  MemoryMapping requestMemoryMapping = 6;

  // the address sent to the device and its space
  uint32       deviceAddress = 3;
  AddressSpace deviceAddressSpace = 4;

  bytes data = 5;
}
```

The `response` contains both the request and device address tuples and the `data`
that was read. The device address tuple allows the developer to see what SNI
translated the address to in the device's address space.

#### [SingleWrite](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto#L119) method

Writes a single memory segment with the given data to the given device.

Request:
```grpc
message SingleWriteMemoryRequest {
  string uri = 1;
  WriteMemoryRequest request = 2;
}
message WriteMemoryRequest {
  uint32        requestAddress = 1;
  AddressSpace  requestAddressSpace = 2;
  MemoryMapping requestMemoryMapping = 4;

  bytes data = 3;
}
```

Similarly to `SingleRead`, the device `uri` must be specified and the `request`
to be fulfilled as well. The request address tuple specifies what memory location
to write to and the `data` is simply the data to write to that location.

Response:
```grpc
message SingleWriteMemoryResponse {
  string uri = 1;
  WriteMemoryResponse response = 2;
}
message WriteMemoryResponse {
  uint32        requestAddress = 1;
  AddressSpace  requestAddressSpace = 2;
  MemoryMapping requestMemoryMapping = 6;

  uint32       deviceAddress = 3;
  AddressSpace deviceAddressSpace = 4;

  uint32 size = 5;
}
```

The response returns the `size` of the data written and where possible (depending
on the SNES device) waits for the write operation to complete before returning.

#### [MultiRead](https://github.com/alttpo/sni/blob/main/protos/sni/sni.proto#L121) method

This method acts exactly the same as `SingleRead` except it allows multiple requests
to be executed together. The exact behavior depends on the SNES device connected to.
All read requests are issued to the device in the order they are requested.

## Device Behavior

### FX Pak Pro

#### WRAM writes

WRAM cannot be **written** to directly by the FX Pak Pro due to design limitations of
the SNES. SNES carts cannot issue their own read or write requests to the main bus.
The FX Pak Pro **can**, however, write to SRAM.

In order to *effectively* perform WRAM writes to affect game state, a strategy can be
devised where the application writes the intended data into an unused portion of SRAM.
The application could patch the ROM (statically or dynamically) to transfer the contents
from the unused portion of SRAM into its final WRAM destination. It would be best to
perform this transfer at a well-known point in the frame lifecycle, such as after NMI
is completed or right before the main game logic subroutine is jumped to.

An alternative method that offers more control is for the application to generate
SNES ASM code that performs the WRAM writes, write that code into SRAM, and patch the
ROM to regularly `JSL` to the generated SRAM code which would self-modify after execution
to disable itself (replace its first instruction byte with `RTL`) until it is overwritten
again by the application to perform a different series of WRAM writes.

Unfortunately, there is no general solution to the WRAM write problem that is available.
The details depend entirely on the ROM in question.

#### MultiRead / MultiWrite

For the FX Pak Pro, the firmware VGET/VPUT commands are used for read and write
respectively, which allow for 8 smaller requests to be sent to the cart in a single
USB request-response cycle. Each small request can be up to 255 bytes thus
allowing for a maximum of 2040 bytes to be read by a single VGET command. Each
original client `request` larger than 255 bytes is broken up into several small
255-byte sized requests followed by the remainder if any. After this step, the small
requests are batched into groups of 8 and put into VGET/VPUT commands which are then
sent to the device until all original client requests are satisfied.

When targeting the FX Pak Pro it is advisable to keep this 8 requests per USB
request-response cycle limitation in mind as it could impact the correctness of your
application logic.

### RetroArch

#### Reads and Writes

RetroArch processes its network commands during vsync just after the last game frame
is presented to the end user. This means there is a maximum delay of 16ms between when
RetroArch receives the network command into its buffer and when it performs the command
and sends back the reply. SNI takes advantage of this regular cadence and sends multiple
requests during the 16ms window and then reads all the responses in the order it sent
them when RetroArch processes the commands and sends the replies.

RetroArch version 1.9.0 and earlier have different behavior from RetroArch version 1.9.2
and later. Version 1.9.1 is broken so don't use it.

Version 1.9.0 and earlier only support the commands `READ_CORE_RAM` and `WRITE_CORE_RAM`.
Version 1.9.2 and later added new commands `READ_CORE_MEMORY` and `WRITE_CORE_MEMORY`.

The older `WRITE_CORE_RAM` command does not send back a reply for successful writes.
SNI does not know when the write completed thus making writes appear to the client to
complete much faster than they actually do.

The newer `READ_CORE_MEMORY` and `WRITE_CORE_MEMORY` commands always write back replies.
SNI always waits for those replies before completing the requests and returning the
response to the application. These commands can also reply back with specific text
describing errors when they occur. SNI forwards those error messages to the application.

### DeviceControl

