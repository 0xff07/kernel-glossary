---
topics: power-management
tags:
    - "power-management"
    - "acpi"
---

# ACPI Debugging

## Interactively browsing ACPI tables

Other than disassemble using `iasl -d` all at once, `acpiexect` is another tool where one could interactively browsing ACPI table. After `acpixtract`, simply:

```
$ acpiexec *.dat
```

And repeatedly using `find` and `dis`. For example, to find all \_PS0 method, in the `acpiexec` shell:

```
find _PS0
```

It'll show something like this:

```
             \_SB.PC00.GFX0._PS0 Method       0x586cbdaf8430 099 Args 0 Len 0000 Aml 0x586cbd826ff1
             \_SB.PC00.XHCI._PS0 Method       0x586cbd990f70 001 Args 0 Len 0031 Aml 0x7b205997a312
        \_SB.PC00.XHCI.RHUB._PS0 Method       0x586cbd991380 001 Args 0 Len 0041 Aml 0x7b205997a3b5
             \_SB.PC00.HDAS._PS0 Method       0x586cbd991ed0 001 Args 0 Len 0035 Aml 0x7b205997a874
        \_SB.PC00.I2C1.TPL0._PS0 Method       0x586cbda523d0 001 Args 0 Len 0028 Aml 0x7b20599b3e12
...
             \_SB.PC00.TRP0._PS0 Method       0x586cbdaa03b0 030 Args 0 Len 024B Aml 0x586cbd7ecf84
             \_SB.PC00.TRP1._PS0 Method       0x586cbdaa4df0 030 Args 0 Len 024B Aml 0x586cbd7ee2d0
             \_SB.PC00.TRP2._PS0 Method       0x586cbdaa9830 030 Args 0 Len 024B Aml 0x586cbd7ef61e
             \_SB.PC00.TRP3._PS0 Method       0x586cbdaae270 030 Args 0 Len 024B Aml 0x586cbd7f096c
```

To inspect a particular implementation, use `dis` command. For example, to see how `\_SB.PC00.TRP0._PS0` is implemented:

```
dis \_SB.PC00.TRP0._PS0
```

Which on my laptop looks like:

```
{
    ADBG (Concatenate ("TBT _PS0 Start RP ", ToHexString (TUID)))
    ADBG (Concatenate ("TBT RP VDID -", ToHexString (VDID)))
    HPEV ()
    HPME ()
    If (LEqual (PMEX, One))
    {
        ADBG ("Disable PME SCI")
        Store (Zero, PMEX) /* External reference */
    }

    Sleep (PLAT)
    If (LOr (LEqual (TUID, Zero), LEqual (TUID, One)))
    {
        If (LEqual (\_SB.PC00.TDM0.WACT, One))
        {
            Store (0x02, \_SB.PC00.TDM0.WACT) /* External reference */
            \_SB.PC00.TDM0.WFCC (ITCT)
            Store (Zero, \_SB.PC00.TDM0.WACT) /* External reference */
        }
        ElseIf (LEqual (\_SB.PC00.TDM0.WACT, 0x02))
        {
            ADBG ("Wait until other _PS0 get response")
            While (LNotEqual (\_SB.PC00.TDM0.WACT, Zero))
            {
                Sleep (0x05)
            }

            ADBG ("Other _PS0 got response")
        }
    }
  ...
}
```

And repeatedly `find` and `dis`. For example, if I'd like to see what `HPEV` does, simply `find` and `dis` it again in the `acpiexec` shell:

```
find HPEV
```

Which shows:

```
             \_SB.PC00.TRP0.HPEV Method       0x57ab8cec2ff0 030 Args 0 Len 009C Aml 0x57ab8cc0fb8d
             \_SB.PC00.TRP1.HPEV Method       0x57ab8cec7a30 030 Args 0 Len 009C Aml 0x57ab8cc10ed9
             \_SB.PC00.TRP2.HPEV Method       0x57ab8cecc470 030 Args 0 Len 009C Aml 0x57ab8cc12227
             \_SB.PC00.TRP3.HPEV Method       0x57ab8ced0eb0 030 Args 0 Len 009C Aml 0x57ab8cc13575
```

`dis` the one under the same scope (`\_SB.PC00.TRP0.HPEV`):

```
dis \_SB.PC00.TRP0.HPEV
```

For the ACPI tables on my laptops it shows:

```
{
    If (LAnd (LNotEqual (VDID, 0xFFFFFFFF), HPSX))
    {
        ADBG (Concatenate ("HotPlug Event Start for ITBT Port - ", ToHexString (TUID)))
        If (LAnd (LEqual (PDCX, One), LEqual (DLSC, One)))
        {
            Store (One, PDCX) /* External reference */
            Store (One, HPSX) /* External reference */
            Notify (^, Zero) // Bus Check
        }
        Else
        {
            Store (One, HPSX) /* External reference */
        }

        ADBG (Concatenate ("HotPlug Event End for ITBT Port - ", ToHexString (TUID)))
    }
}
```

## References

### Videos

### Links

- [ACPI Source Language (ASL) Tutorial](https://www.intel.com/content/www/us/en/content-details/772722/basic-acpi-source-language-asl-constructs-tutorial.html) from Intel.
- [Debug ACPI DSDT and SSDT with ACPICA Utilities](https://ubuntu.com/blog/debug-dsdt-ssdt-with-acpica-utilities)
- [ACPI Tricks and Tips](https://wiki.ubuntu.com/Kernel/Reference/ACPITricksAndTips)
- [Debugging ACPI using acpiexec](https://smackerelofopinion.blogspot.com/2010/03/debugging-acpi-using-acpiexec.html)
- [Simple ACPI monitoring tools](https://smackerelofopinion.blogspot.com/2010/01/simple-acpi-monitoring-tools.html)
- [How to Read The ACPI Specification](https://bioshacking.blogspot.com/2010/10/how-to-read-acpi-specification.html): the order between chapters has changed in the latest ACPI spec, but the idea is the same.
