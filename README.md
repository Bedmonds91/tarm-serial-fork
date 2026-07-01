[![GoDoc](https://godoc.org/github.com/tarm/serial?status.svg)](http://godoc.org/github.com/tarm/serial)
[![Build Status](https://travis-ci.org/tarm/serial.svg?branch=master)](https://travis-ci.org/tarm/serial)

=====================================================================

## This fork: macOS arbitrary baud rate support

Upstream `tarm/serial` hard-errors on macOS for any baud rate that isn't one of
a fixed set of termios constants (tops out at 115200) — even though `serial_posix.go`
already had a dead, commented-out `IOSSIOSPEED` implementation (the standard way
to set arbitrary baud rates on Darwin) that was never finished or wired in.

This fork finishes it: non-standard rates (e.g. `1000000`/1Mbaud, needed by some
MCUboot bootloaders' serial DFU/recovery mode) now work correctly on macOS via
[`IOSSIOSPEED`](https://developer.apple.com/library/archive/documentation/DeviceDrivers/Conceptual/WritingDeviceDriver/SpecialConsid/SpecialConsid.html),
instead of failing immediately with `Unknown baud rate`.

### Using this fork

Add a `replace` directive to your project's `go.mod`:

```
replace github.com/tarm/serial => github.com/Bedmonds91/tarm-serial-fork v0.0.1
```

This is a drop-in replacement — no API changes, only `openPort()`'s internal
behavior on macOS/Darwin is affected. Linux and Windows are unchanged.

### Why this exists

`tarm/serial` appears unmaintained (no recent commits), so tools built on it —
like [newtmgr](https://github.com/apache/mynewt-newtmgr) and
[mcumgr](https://github.com/apache/mynewt-mcumgr-cli) — inherit this limitation.
If you hit `Unknown baud rate` on macOS with either of those tools, this fork
(via a `replace` directive) fixes it without waiting on upstream.

### Quick start: building newtmgr with this fix

```sh
# 1. Clone newtmgr
git clone https://github.com/apache/mynewt-newtmgr.git
cd mynewt-newtmgr

# 2. Point it at this fork instead of the stock tarm/serial
go mod edit -replace github.com/tarm/serial=github.com/Bedmonds91/tarm-serial-fork@v0.0.1
go mod tidy

# 3. Build
go build -o newtmgr ./newtmgr

# 4. Install and put it on your PATH
mkdir -p ~/go/bin
mv newtmgr ~/go/bin/
echo 'export PATH="$HOME/go/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 5. macOS Gatekeeper will kill a freshly-built, ad-hoc-signed Go binary on
#    first run with no useful error — this isn't specific to newtmgr or this
#    fork, it happens to any locally-built Go binary on macOS. Re-sign it:
codesign --sign - --force --deep ~/go/bin/newtmgr

# 6. Verify
newtmgr version
```

Then connect at whatever baud rate you need:

```sh
ls /dev/tty.*   # find your device
newtmgr --conntype serial --connstring "dev=/dev/tty.usbserial-XXX,baud=1000000" image upload path/to/firmware.bin
```

The same steps apply to [mcumgr](https://github.com/apache/mynewt-mcumgr-cli) —
just swap the repo URL and build target in step 1/3.

=====================================================================


Serial
========
A Go package to allow you to read and write from the
serial port as a stream of bytes.

Details
-------
It aims to have the same API on all platforms, including windows.  As
an added bonus, the windows package does not use cgo, so you can cross
compile for windows from another platform.

You can cross compile with
   GOOS=windows GOARCH=386 go install github.com/tarm/serial

Currently there is very little in the way of configurability.  You can
set the baud rate.  Then you can Read(), Write(), or Close() the
connection.  By default Read() will block until at least one byte is
returned.  Write is the same.

Currently all ports are opened with 8 data bits, 1 stop bit, no
parity, no hardware flow control, and no software flow control.  This
works fine for many real devices and many faux serial devices
including usb-to-serial converters and bluetooth serial ports.

You may Read() and Write() simulantiously on the same connection (from
different goroutines).

Usage
-----
```go
package main

import (
        "log"

        "github.com/tarm/serial"
)

func main() {
        c := &serial.Config{Name: "COM45", Baud: 115200}
        s, err := serial.OpenPort(c)
        if err != nil {
                log.Fatal(err)
        }
        
        n, err := s.Write([]byte("test"))
        if err != nil {
                log.Fatal(err)
        }
        
        buf := make([]byte, 128)
        n, err = s.Read(buf)
        if err != nil {
                log.Fatal(err)
        }
        log.Printf("%q", buf[:n])
}
```

NonBlocking Mode
----------------
By default the returned Port reads in blocking mode. Which means
`Read()` will block until at least one byte is returned. If that's not
what you want, specify a positive ReadTimeout and the Read() will
timeout returning 0 bytes if no bytes are read.  Please note that this
is the total timeout the read operation will wait and not the interval
timeout between two bytes.

```go
	c := &serial.Config{Name: "COM45", Baud: 115200, ReadTimeout: time.Second * 5}
	
	// In this mode, you will want to suppress error for read
	// as 0 bytes return EOF error on Linux / POSIX
	n, _ = s.Read(buf)
```

Possible Future Work
-------------------- 
- better tests (loopback etc)
