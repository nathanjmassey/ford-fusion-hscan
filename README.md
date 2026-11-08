# ford-fusion-hscan

How to read door, boot and central-lock state, plus stored fault counts, from a **2002-2012 Ford Fusion** with a cheap ELM327 OBD-II dongle. No MS-CAN adapter or wiring is needed.

Everything here was found by observing the car's own bus traffic: opening one door at a time and watching which bits changed. Nothing was decompiled or copied from diagnostic software.

## Test vehicle

| | |
|---|---|
| Car | 2008 Ford Fusion 1.4 TDCi, left-hand drive, EU market |
| Adapter | Generic Bluetooth ELM327, firmware v2.2 (`ATI` = `ELM327 v2.2`) |
| Bus | HS-CAN at the OBD-II port (pins 6/14), 500 kbit/s, 11-bit IDs (ELM protocol 6) |
| Date | November 2026 |

The Fusion shares its platform with the Fiesta Mk6, so this probably applies there too. **That is untested.** On right-hand-drive cars, the driver and passenger meanings are probably swapped (see the note under the table).

## Door, boot and lock state: ID `0x433`

The instrument cluster broadcasts `0x433` about 4 times a second while the ignition is at position II. DLC 8.

**Byte 0: door and boot bits** (bit set = open; all closed = `0x00`)

| Mask | Meaning (LHD car) |
|---|---|
| `0x80` | Front left door (driver) |
| `0x40` | Front right door (passenger) |
| `0x20` | Rear left door |
| `0x10` | Rear right door |
| `0x08` | Boot / tailgate |

The bits are physical positions. On an RHD car, `0x80` is probably the passenger door and `0x40` the driver door.

**Byte 5: central locking**

| Value | Meaning |
|---|---|
| `0x10` | Locked |
| `0x20` | Unlocked |

The lock byte follows the central locking (fob or the driver's interior lock). The passenger's interior lock caused no change on the bus, so there's no per-door lock state.

**Real frames** (bytes 0-7, hex):

```
00 10 60 00 00 20 00 00   all closed, unlocked
40 10 60 00 00 20 00 00   front right (passenger) door open
08 10 60 00 00 20 00 00   boot open
00 10 60 00 00 10 00 00   all closed, locked
```

### Limits

- The data only exists with the ignition at **II**. At position I (accessory) the bus winds down within about 30-60 s and `0x433` stops. With the key out, nothing is broadcast.
- The fob won't lock the car with the key at II, so you only see "locked" after the driver's interior lock is used.

## Reading `0x433` with an ELM327

Two settings matter:

- `ATCSM1` (CAN silent monitoring): the adapter only listens and never acknowledges frames.
- `ATCAF0` (CAN auto-formatting off): **this is required.** With formatting on, the ELM treats byte 0 as an ISO-TP header (PCI byte) and corrupts these frames.

Sequence, with the adapter already connected on protocol 6 (500 kbit/s, 11-bit):

```
ATCSM1        silent monitoring
ATCAF0        raw frames, no ISO-TP formatting
ATH1          show the CAN ID
ATCRA433      only receive ID 0x433
ATMA          monitor; stop it (send any character) after the first 433 line
```

Then restore the adapter before normal OBD polling:

```
ATCRA         clear the receive filter
ATCAF1
ATH0
```

A single read takes roughly 400-800 ms on a Bluetooth ELM327, including the setup and restore. If you also poll normal OBD PIDs, read doors every few seconds while parked and much less often when moving, or your PID rate will drop.

## Stored fault count (KWP2000)

Only two modules answer on HS-CAN:

| Request ID | Response ID | Module |
|---|---|---|
| `7E0` | `7E8` | Engine (PCM) |
| `720` | `728` | Instrument cluster |

ABS, airbag and body modules didn't answer on HS-CAN. They're presumably on K-line or MS-CAN.

- UDS `19` (ReadDTCInformation) is rejected with negative response `7F 19 11` (service not supported).
- KWP2000 `18 00 FF 00` (ReadDiagnosticTroubleCodesByStatus) works. The reply is `58 <count> <codes...>`, so `58 00` = no stored faults.

```
ATSH7E0
18 00 FF 00     ->  58 00
```

## HS-CAN bus inventory (ignition at II, engine off)

Broadcast IDs seen, with approximate rates. Only `0x433` is decoded. The rest are leads, not confirmed meanings.

| ID | Rate | Notes |
|---|---|---|
| `0x200` | 38 Hz | |
| `0x201` | 19 Hz | Looks like rpm/speed (`0x2710` offset at rest) |
| `0x210` | 22 Hz | Byte 6 is a counter |
| `0x230` | 29 Hz | |
| `0x360` | 40 Hz | |
| `0x420` | 4 Hz | |
| `0x428` | 4 Hz | Byte 1 looks like system voltage x10 (`0x7A` = 12.2 V, dipped under window-motor load) |
| `0x430` | 19 Hz | Bytes 0/2/4 vary at rest (analogue values?) |
| `0x433` | 4 Hz | **Doors, boot, lock (decoded above)** |
| `0x4B0` | 28 Hz | Looks like four wheel speeds (`0x2710` x4 at 0 km/h) |
| `0x620` | 4 Hz | |
| `0x630` | 0.4 Hz | |

## Files

- [`ford-fusion-hscan.dbc`](ford-fusion-hscan.dbc): DBC definition of `0x433`, for SavvyCAN, cantools and similar tools.

## Related work

[roncapat/Ford-Fiesta-MK5-MS-CAN-bus](https://github.com/roncapat/Ford-Fiesta-MK5-MS-CAN-bus) independently documented `0x433` on the Fiesta Mk5's MS-CAN (front doors, trunk, lock, reverse gear). Its bits match the ones here. This repo adds the rear-door bits, confirms the message on the Fusion's HS-CAN at the OBD port, and gives the ELM327 recipe and the fault-count request.

## Contributing

Results from other cars are welcome, especially the Fiesta Mk6, RHD cars, and five-door versus three-door bodies. Open an issue with the model, year, market, adapter and a few raw frames.

## Disclaimer

This is not affiliated with or endorsed by Ford Motor Company. Everything here is read-only and listen-only. Use it at your own risk, and don't send arbitrary commands to modules on a car you rely on.

## Licence

[CC BY 4.0](LICENSE): free to use and adapt, with attribution.
