---
topics: pcie
tags:
    - "pcie"
---

# Advanced Error Reporting (AER)

## Registers

### In PCI Express Capability Structure

```
Capabilities: [c0] Express (v2) Endpoint, MSI 00
        DevCap: MaxPayload 512 bytes, PhantFunc 0, Latency L0s <1us, L1 unlimited
                ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset+ SlotPowerLimit 0W
        DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+ FLReset-
                MaxPayload 256 bytes, MaxReadReq 512 bytes
        DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
```

The `DevCtl` shows if reporting of Correctable Errors (`CorrErr`), Non-Fatal Uncorrectable Errors (`NonFatalErr`), and Fatal Uncorrectable Errors are enabled. The `DevSta` show if an error of that class has ever occured.

### In PCI Express Capability Structure (Root)

Root port being the target of the Error Messages has additional things to control:

```
Capabilities: [40] Express (v2) Root Port (Slot+), MSI 00
        DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                RlxdOrd- ExtTag- PhantFunc- AuxPwr- NoSnoop-
                MaxPayload 128 bytes, MaxReadReq 128 bytes
        DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr+ TransPend-
        ...
        RootCap: CRSVisible-
        RootCtl: ErrCorrectable- ErrNon-Fatal- ErrFatal- PMEIntEna+ CRSVisible-
        RootSta: PME ReqID 0000, PMEStatus- PMEPending-
```

The `RootCtl` control if the root port should signal the platform when the respective error class has happended.

### In AER Capability Structure

```
Capabilities: [100 v2] Advanced Error Reporting
        UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
        UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
        UESvrt: DLP+ SDES- TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
        CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
        CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
        AERCap: First Error Pointer: 00, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
        HeaderLog: 00000000 00000000 00000000 00000000
```

The `UE`, `CE` stands for Uncorrectable Error and Correctable Error respecively:

1. `UESta`, `CESta`: show if the given error has occured.
2. `UEMsk`, `CEMsk`: mask an Error so that the function doesn't report it to root.
3. `UESvrt`: bits to specify if a Uncorrectable Error should be Fatal or Non-Fatal.

For trying to recover what has happend during the error, there are `HeaderLog` and `First Error Pointer`:

1. `HeaderLog`: records the header of the TLP that triggers this Error. Note if an Error is masked, its header won't get logged.
2. `First Error Pointer`: offset into the `UESta` indicating which Uncorrectable Error happened first.

And if the device is `MultHdrRecCap` capable and it is enabled by `MultHdrRecEn`, then clearing the bit in the `UESta` pointed by the `First Error Pointer` will make the `First Error Pointer` point to the next Uncorrectable Error, and the `HeaderLog` will update accordingly. This can be done repeatedly until the `First Error Pointer` points to a bit not set, meaning it already reaches its limit. How many Uncorrectable Error a function can track is implementation-specific.

Note that Header Log Overflow is also an error of its own. It's a Correctable Error.

### In AER Capability Structure (Root)

A root port has additional `RootCmd`, `RootSta` and `ErrorSrc` for indicating an error has happened downstream:

```
Capabilities: [100 v1] Advanced Error Reporting
        UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
        UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
        UESvrt: DLP+ SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
        CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
        CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
        AERCap: First Error Pointer: 00, ECRCGenCap- ECRCGenEn- ECRCChkCap- ECRCChkEn-
                MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
        HeaderLog: 00000000 00000000 00000000 00000000
        RootCmd: CERptEn- NFERptEn- FERptEn-
        RootSta: CERcvd- MultCERcvd- UERcvd- MultUERcvd-
                 FirstFatal- NonFatalMsg- FatalMsg- IntMsg 0
        ErrorSrc: ERR_COR: 0000 ERR_FATAL/NONFATAL: 0000
```

1. `ErrorSrc`: BDF of the downstream device where an Error originate.
2. `RootSta`:
    * `CERcvd`, `UERcvd`: an Correctable Error/Uncorrectable Error has occured downstream.
    * `MultCERcvd`, `MultUERcvd`: multiple Correctable Errors/Uncorrectable Errors have occurred downstream.
    * `INtMsg`: interrupt number in the MSI-X table to signal the root.
3. `RootCmd`: whether to trigger an interrupt on Correctable/Non-Fatal/Fatal Errors.

## Error Severity (6.2.2)

### Correctable (6.2.2.1)

Errors that the hardware can fix by itself. For example, an error in LCRC can be handled by replaying the TLP in data link layer.

### Uncorrectable (6.2.2.2)

Further classified into 2 categories: non-fatal and fatal.

#### Uncorrectable (Non-Fatal)

Errors that hardware cannot fix by itself, but the scope of the errors is limited to a inidividual transactions. The Link remains usable.

For example, recieving a Completion Timeout from a function doesn't really impact the Link. It could issue specific to that target function, but. In this case only traffic to that function is under impact.

#### Uncorrectable (Fatal)

The Link is considered not functional and needs recovering.

## Change in severity

### The "Advisory" Non-Fatal Error (6.2.3.2.4)

In some cases a Non-Fatal Uncorrectable Error has to be reported as Correctable Error.

This could simply because continuation of working is possible even though there's no unified way to fix that scenario, or because multiple Non-Fatal Uncorrectable Errors would be reported in different places, confusing the error logging, so it's better to lower some Non-Fatal Uncorrectable Errors into Correctable Errors instead, to make the error reporting more meaningful.

### The `UESvrt` register

This register in the AER capability structure control which Uncorrectable Error are the Fatal ones. The severity of a Uncorrectable Error can be change by programming this register.

## Reporting errors

### By Error Message (6.2.3.2)

It is a Message TLP implicitly routed to root.

### By Completion Status field (6.2.3.1)

By setting the Completion Status field. Note that not all abnormal completion status trigger the error. Some of them are advisory cases.

## References

### Videos

- [Everything You Wanted to Know About PCIe But Were Too Proud to Ask (19:53)](https://youtu.be/td0zisK-ksQ?si=a8iXf5yHJLyuDr6p&t=1193)
- [PCI Express HW Fault Management (RAS) Solution Implementation considerations (08:11)](https://youtu.be/GfiXlnWBXiA?si=et5Wgga2Y84gIASt&t=491)

### Links

- [5.6. PCI Express Capability Structure, Stratix V Avalon-ST Interface with SR-IOV PCIe Solutions: User Guide](https://www.intel.com/content/www/us/en/docs/programmable/683488/16-0/pci-express-capability-structure.html)
- [PCIe error logging and handling on a typical SoC](https://www.design-reuse.com/article/60757-pcie-error-logging-and-handling-on-a-typical-soc/)
