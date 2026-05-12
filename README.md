# senseid-definitions

YAML definitions for the SenseID tag families. These files are consumed by
the `senseid` SDK and any other tool that needs to identify SenseID tags
and decode their sensor payload.

| File | Tag family | Identifier | Sensor data lives in |
|------|-----------|-----------|----------------------|
| [`senseid_rain.yaml`](senseid_rain.yaml) | SenseID RAIN UHF | PEN `00 00 00 F1 D3`, byte 6 ∈ `0x01..0xFE` | EPC |
| [`senseid_legacy.yaml`](senseid_legacy.yaml) | SenseID legacy (Kliskatek) | PEN `00 00 00 F1 D3`, byte 6 = `0xFF` | User memory (USER bank, word `0x100`) |
| [`senseid_farsens.yaml`](senseid_farsens.yaml) | Farsens RM family | PEN `00 00 A9 3C` (after a leading `0x00` header byte) | User memory (USER bank, word `0x100`) |
| [`senseid_ble.yaml`](senseid_ble.yaml) | SenseID BLE | Manufacturer-data PEN | BLE advertisement |
| [`senseid_nfc.yaml`](senseid_nfc.yaml) | SenseID NFC | NTAG type config | NDEF / NFC bulk read |

## RAIN UHF families share the same PEN

Both standard SenseID RAIN and the Kliskatek legacy line use the same PEN
header (`00 00 00 F1 D3`) and the same `type` namespace, so the byte
immediately after the type is used as a **family marker** to tell them apart:

```
[5 B] PEN header  = 00 00 00 F1 D3
[1 B] type        = sensor model (e.g. 0x05 = EVAL-*-RHAT)
[1 B] family marker:
        0x01..0xFE  -> standard SenseID (this byte is the real fw_version,
                      followed by 3 bytes of SN and 4 bytes of sensor data)
        0xFF        -> Kliskatek legacy (followed by 5 bytes of SN; the
                      sensor data lives in User memory instead of the EPC)
[…] tail per family
```

Parsers MUST refuse to decode a SenseID-PEN EPC whose byte 6 is `0xFF` as
standard SenseID, and conversely a legacy parser MUST require byte 6 ==
`0xFF` to accept it.

## YAML schema

Common fields:

- `version` / `date` — schema version and date of last edit.
- `pen_header` — array of bytes that identifies the tag family in the EPC
  (or equivalent identifier on BLE/NFC).
- `types` — dictionary keyed by the 1-byte `type` field; each entry has at
  least `name`, `description`, `data_def` (list of channels) and
  `fw_versions` (firmwares known to fit this layout).

Family-specific fields used by `senseid_legacy.yaml` and
`senseid_farsens.yaml`:

- `memory_bank`, `word_offset`, `word_count` — where the reader must
  perform the embedded Read (USER bank, word `0x100`, N words).
- `preamble` *(Farsens only)* — the first byte of the User-memory datagram
  (`0xAA`); anything else means the SPI buffer was not armed and the
  reading must be discarded.
- `data_index` *(Farsens only)* — byte offset where the sensor data starts
  inside the User-memory datagram.
- `epc_family_marker` *(legacy only)* — byte 6 of the EPC that distinguishes
  legacy from standard SenseID (currently `0xFF`).
- `skip_when` *(legacy only)* — datagram-level conditions that should
  produce `data=None` (e.g. `fw_version: [0x00, 0xFF]` for stale samples).

## Sensor data channel (`data_def` entry)

| Field | Type | Notes |
|-------|------|-------|
| `magnitude` / `magnitude_short` | `str` | Long and short human name |
| `unit_long` / `unit_short` | `str` | Unit name and symbol |
| `type` | `uint16` \| `int16` \| `float` | Raw size and signedness in the payload (little-endian) |
| `transform` | `none` \| `linear` \| `thermistor-beta` | How to turn the raw value into a calibrated reading |
| `coefficients` | `list[float]` | Parameters of the transform |

## License

MIT (see [LICENSE](LICENSE)).
