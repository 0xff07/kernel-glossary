---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 1, Part 2

## SPECIFICATIONS

## LINUX KERNEL

Link Training Phase 1 is handled by the USB Type-C port controller and PD stack, outside the thunderbolt driver. The thunderbolt driver begins its work after Phase 1 completes.

## OTHER INFORMATION

- [USB Type-C and PD Compliance Testing](https://youtu.be/NlZNAFBcgrk)
- [An Example of USB Power Delivery](https://www.reclaimerlabs.com/blog/2017/5/16/example-usb-power-delivery)
- [All About USB-C: Replying Low-Level PD](https://hackaday.com/2023/02/22/all-about-usb-c-replying-low-level-pd/)
- [USB Type-C and Power Delivery Messaging](https://blog.teledynelecroy.com/2016/05/usb-type-c-and-power-delivery-messaging.html)

## DETAILS

The VBUS power advertised by the DFP is the default power that makes following negotiation operational. That however may not be the final power the downstream device would like to use.

To negotiate what `VBUS` power is suitable for downstream device, a process called PD contract negotiation will happen.

### Negotiate the PD contract

```
   Source (Provider)                        Sink (Consumer)
   -----------------                        ----------------

   |                                              |
   |---- Source_Capabilities (Data) ------------->|
   |<--- GoodCRC (Control) -----------------------|
   |                                              |
   |<--- Request (Data; select PDO) --------------|
   |---- GoodCRC (Control) ---------------------->|
   |                                              |
   |---- Accept (Control) ----------------------->|
   |<--- GoodCRC (Control) -----------------------|
   |                                              |
   |   --- Power transition (vSafe5V → Requested PDO) ---
   |                                              |
   |---- PS_RDY (Control) ----------------------->|
   |<--- GoodCRC (Control) -----------------------|
   |                                              |
   |        >>> Explicit Contract Active <<<      |
   |                                              |

```

### The `Source_Capabilities` Message

This packet encapsulates multuple "Power Data Objects" (PDO), each describes a power configuration.

```
+---------------------+--------------------+----------------------+-----------------+
| Preamble + SOP*     | Header             | 1..N PDOs (32b each) | CRC-32 + EOP    |
+---------------------+--------------------+----------------------+-----------------+
                         N >= 1
```

#### The Power Data Object (PDO)

```
[31:30][29] [28]  [27]  [26]  [25]  [24]  [23]  [22]  [21:20]    [19:10]           [9:0]
+------+----+-----+-----+-----+-----+-----+-----+---------------+----------------+------------------+
| Type |DRP |USBS |UPwr |USBC |DRD  |UEM  |EPR? |Rsvd|   Peak   |Voltage (50mV)  |Max Current (10mA)|
| 00b  |    |Sup? |     |Comm?|     |(PD3)|cap? |    |  Current |                |                  |
+------+----+-----+-----+-----+-----+-----+-----+------=--------+----------------+------------------+

Notes:
- Type=00b = Fixed.
- DRP=Dual-Role Power;
- USBSup?=USB Suspend Supported;
- UPwr=Unconstrained Power;
- USBC=USB Communications Capable;
- DRD=Dual-Role Data; UEM=Unchunked Ext Msgs;
- EPR?=EPR Mode Capable (PD3.1)
- PeakCurrent = source-defined peak profile.
```

### The Request

```
+---------------------+--------------------+----------------------+-----------------+
| Preamble + SOP*     | Header             | 1 x RDO (32 bits)    | CRC-32 + EOP    |
+---------------------+--------------------+----------------------+-----------------+
                         N = 1
```

#### The Request Data Object (RDO)

The sink chooses the power PDO by sending a RDO. The `Obj#` field in the RDO specifies which one in the PDO does it like to use. If none is desirable, it could also send a RDO with Capability Mismatch to let the source know.

```
[31:28]  [27]    [26]     [25]     [24]     [23]    [22]    [21:20]  [19:10]             [9:0]
+------+--------+--------+--------+--------+--------+----------------+--------------------+--------------------+
| Obj# |GiveBack|CapMis? |USBComm?|NoUSBSus| UEM?   | ERP   | Resvd  | Operating Current  | Max/Min Operating  |
|      |        |        |        |        |        | CAP   |        |           (10mA)   | Current (10mA)     |
+------+--------+--------+--------+--------+--------+----------------+--------------------+--------------------+

Notes:
- Obj# = which PDO (1-based) the sink requests.
- GiveBack (historical field)
- CapMismatch=capability mismatch accepted;
- USBComm?/NoUSBSus? = sink's preferences;
- UEM? = Unchunked Ext Msgs support (PD3).
- Operating Current = the current sink intends to draw under normal operation.
- Max/Min Operating Current = headroom (or minimum when GiveBack used historically).
```

### `Accept`

It's a packet without data. There's only header:

```
+---------------------+--------------------+-----------------+
| Preamble + SOP*     | Header (Type=*)    | CRC-32 + EOP    |
+---------------------+--------------------+-----------------+
                         N = 0
```

Where the header is:

```
[15] [14:12][11:9][8]  [7]  [6:4] [3:0]
+----+------+-----+----+----+-----+---------+
|Ext |#DO(0)| IDx | DR |PR=1| Rev |TYPE=0011|
+----+------+-----+----+----+-----+---------+
```

### `PS_RDY`

It's again a packet without data. There's only header:

```
+---------------------+--------------------+-----------------+
| Preamble + SOP*     | Header (Type=*)    | CRC-32 + EOP    |
+---------------------+--------------------+-----------------+
                         N = 0
```

Where the header is:

```
[15] [14:12][11:9][8]  [7]  [6:4] [3:0]
+----+------+-----+----+----+-----+---------+
|Ext |#DO(0)| IDx | DR |PR=1| Rev |TYPE=0110|
+----+------+-----+----+----+-----+---------+
```

