# BLE Packet Filter

Rust CLI tool for turning noisy Bluetooth Low Energy (BLE) capture logs into human-readable request/notification pairs. The program scans ATT traffic, keeps track of the latest write request, and prints it along with the next handle-value notification (or immediately when `--write-only` is set). Helpful when reverse-engineering GATT characteristics or keeping a record of command/response pairs.

## Requirements
- Rust toolchain (e.g., via [rustup](https://rustup.rs/))

## Building
```bash
cargo build --release
```
Artifacts will land under `target/release/ble_packet_filter`.

## Usage
```bash
ble_packet_filter --input <capture.txt> --output <report.txt> [options]
```
When running directly from the workspace you can `cargo run --release -- …`.

### Options
- `-i, --input <PATH>`: Source capture file (required). Input should resemble the nRF Sniffer/packet-logger format used in the `packets/` samples.
- `-o, --output <PATH>`: Destination for formatted output (required).
- `-c, --handle <HANDLE>`: Optional substring filter applied to the handle column (e.g., `0x0040`). Only packets whose handle text contains the filter are considered.
- `-w, --write-only`: Emit every write request even if no notification follows. Each unpaired write is output just before the next write/write-response or at EOF.

## Typical Workflows
- **Pair writes with notifications**
  ```bash
  cargo run --release -- \
    --input packets/vibrate_charge_start_on.txt \
    --output readable_pairs.txt
  ```
- **Focus on a specific connection handle**
  ```bash
  ble_packet_filter \
    --input packets/pausemode_off.txt \
    --output labeled_notification_pairs.txt \
    --handle 0x0040
  ```
- **Write-only audit**
  ```bash
  ble_packet_filter \
    --input packets/change_vibrate_charge_start_off.txt \
    --output write_only_dump.txt \
    --write-only
  ```

The output groups each entry vertically, includes timestamps, and tags commands by inspecting the first bytes (e.g., identifying `00C0 0100` as “Read Device Info”). See `readable_pairs.txt` for an example result.
