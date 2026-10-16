# Project Memory

## Purpose and target

- Battery-powered Zigbee gas counter for the Seeed Studio XIAO nRF52840.
- Fork repository (`origin`): `https://github.com/marek-polak/ZigbeeGasCounterNCS` (upstream base: `https://github.com/IgnacioHR/ZigbeeGasCounterNCS`).
- The application counts a mechanical meter's reed-switch transitions locally and exposes cumulative gas consumption over Zigbee.
- The initial image is programmed over SWD. Subsequent firmware can be delivered using Zigbee OTA and MCUboot.
- Primary deployed target: `gas_counter_xiao_nrf52840/nrf52840`.
- Development/debug target may also include Nordic DK boards, but never apply a DK overlay to the XIAO build.

## Required SDK and Zigbee architecture

- Use **nRF Connect SDK v3.4.0** with **Zigbee R23 add-on v1.4.0**.
- Local SDK workspace: `/opt/nordic/ncs/v3.4.0`.
- Toolchain: `/opt/nordic/ncs/toolchains/ccc010f809`.
- Zigbee was removed from the core NCS repository in NCS 3.x. `ncs-zigbee` must be the west manifest:

  ```sh
  cd /opt/nordic/ncs/v3.4.0
  west config manifest.path ncs-zigbee
  west update
  ```

- Verify `west config manifest.path` prints `ncs-zigbee` and the add-on is v1.4.0.
- Do not downgrade this source tree to NCS 2.9.x. The XIAO overlay uses the newer Nordic GPIO `latch-detect` functionality required for System OFF wake handling.
- The default R23 add-on behavior is R22-compatible Zigbee behavior, which is intentional for Zigbee 3.0 interoperability.

## Build

Run west from the NCS workspace through its matching toolchain:

```sh
cd /opt/nordic/ncs/v3.4.0
/opt/nordic/ncs/toolchains/ccc010f809/nrfutil/bin/nrfutil \
  toolchain-manager launch --ncs-version v3.4.0 -- \
  west build \
  --build-dir /Users/marek.polak/Projects/hardware/ZigbeeGasCounterNCS_clean/build \
  /Users/marek.polak/Projects/hardware/ZigbeeGasCounterNCS_clean \
  --pristine \
  --board gas_counter_xiao_nrf52840/nrf52840 \
  -- -DCONF_FILE=prj_xiao.conf \
     -DBOARD_ROOT=/Users/marek.polak/Projects/hardware/ZigbeeGasCounterNCS_clean
```

Do not pass these obsolete/incorrect VS Code arguments:

- `-DDTC_OVERLAY_FILE=boards/nrf5340dk_nrf5340_cpuapp.overlay`
- `-DEXTRA_DTC_OVERLAY_FILE=boards/gas_counter_xiao_nrf52840.overlay`
- `-DEXTRA_CONF_FILE=sysbuild/mcuboot.conf`

The XIAO application overlay and MCUboot configuration are discovered automatically.

Expected artifacts:

- Initial SWD image: `build/merged_gas_counter_xiao_nrf52840_nrf52840.hex`
- Signed application: `build/ZigbeeGasCounterNCS_clean/zephyr/zephyr.signed.hex`
- Debug symbols: `build/ZigbeeGasCounterNCS_clean/zephyr/zephyr.elf`
- Zigbee OTA image: `build/*.zigbee`

`SB_CONFIG_MERGED_HEX_FILES=y` is required to produce the combined initial SWD image.

## Flash layout

The XIAO uses internal nRF52840 flash plus its onboard `p25q16h` QSPI flash:

| Region                   | Device        |   Address |    Size |
| ------------------------ | ------------- | --------: | ------: |
| MCUboot                  | Internal      | `0x00000` |  64 KiB |
| Primary application slot | Internal      | `0x10000` | 916 KiB |
| Settings                 | Internal      | `0xF5000` |   8 KiB |
| ZBOSS NVRAM              | Internal      | `0xF7000` |  32 KiB |
| ZBOSS product config     | Internal      | `0xFF000` |   4 KiB |
| Secondary OTA slot       | External QSPI | `0x00000` | 916 KiB |

- Shared partition definition: `dts/nrf52840_xiao_fota_partitions.dtsi`.
- Both the application and MCUboot must use exactly the same partition layout.
- MCUboot's overlay is `sysbuild/mcuboot/boards/gas_counter_xiao_nrf52840.overlay`.
- Do not use the nRF5340 partition overlay or the DK's `mx25r64` node; the XIAO flash node is `p25q16h`.
- Keep `slot1_partition` in external QSPI (`p25q16h`, 916 KiB) rather than upstream's 464 KiB dual-internal-slot layout: Zigbee + FOTA builds (especially with trace/debug symbols at ~581 KiB) exceed 464 KiB.
- Because `sysbuild/mcuboot/prj.conf` replaces MCUboot's default `prj.conf`, it **must** include `CONFIG_MAIN_STACK_SIZE=10240`. During boot validation, `cc310_sha256_update()` allocates a 4 KiB buffer on `z_main_stack`; if `CONFIG_MAIN_STACK_SIZE` defaults to 1024 bytes, the stack overflows into `.bss`, corrupts `FIH_SUCCESS`, and traps MCUboot in `FIH_PANIC` (`0x428`).

## SWD flashing and debugging

Connected programmer:

- Raspberry Pi Debug Probe firmware on Pico/Pico 2
- CMSIS-DAP probe ID: `67FCFD549A37182F`
- pyOCD target: `nrf52840`

Initial full flash:

```sh
/opt/nordic/ncs/toolchains/ccc010f809/bin/pyocd load \
  -u 67FCFD549A37182F \
  -t nrf52840 \
  -e chip \
  -f 1000000 \
  build/merged_gas_counter_xiao_nrf52840_nrf52840.hex
```

The board must explicitly select pyOCD as its default flasher and debugger. `nrfutil` cannot program through this CMSIS-DAP probe. Keep `--dt-flash=n` in the pyOCD runner arguments: NCS 3.4's runner otherwise incorrectly expects legacy `CONFIG_FLASH_LOAD_OFFSET`, while Intel HEX already carries absolute addresses.

For command-line debugging, start `pyocd gdbserver` and connect the Zephyr SDK `arm-zephyr-eabi-gdb` to port 3333 using the application ELF. MCUboot executes first, so application breakpoints trigger only after MCUboot starts the signed image.

Useful breakpoints:

- `main`
- `main_button_isr_cb`
- `reed_input_cb`
- `zboss_signal_handler`
- `leave_action`
- `leave_callback`

Avoid long breakpoints in radio/ZBOSS paths because halting the CPU violates Zigbee timing and may disconnect the device.

The release ZBOSS libraries enable LTO. Source debugging can therefore have inlined/optimized symbols. Do not set `CONFIG_LTO=n` while linking release ZBOSS libraries. For deeper stack debugging, use `CONFIG_ZIGBEE_ENABLE_TRACES=y`, which selects matching non-LTO trace libraries, accepting larger and noisier firmware.

## Hardware pinout

| Function               | XIAO pin | nRF52840 pin | Polarity/use       |
| ---------------------- | -------- | ------------ | ------------------ |
| Main button            | D0       | P0.02        | Active high input  |
| Reed switch            | D1       | P0.03        | Active high input  |
| Battery ADC            | D2/A2    | P0.28/AIN4   | Analog input       |
| Battery divider enable | D3       | P0.29        | Active high output |
| Debug UART TX          | D6       | P1.11        | 115200 baud        |
| Debug UART RX          | D7       | P1.12        | 115200 baud        |

The switch and reed input are defined without internal pull resistors. The external circuit must provide deterministic inactive and active levels; do not leave either GPIO floating. Confirm the intended external pull direction before changing polarity or GPIO flags.

The GPIO overlay applies `latch-detect` and a sense mask for P0.02/P0.03 so the button and reed sensor can wake the device from System OFF.

## Development logging and sleep modes

Feature selection is compile-time in `include/zb_features.h`:

- Production battery mode: `FEATURE_DEEP_SLEEP`.
- Interactive development mode: `FEATURE_LIGHT_SLEEP`.
- Define at most one mode.

For GPIO/Zigbee debugging, the XIAO profile can enable:

- `CONFIG_LOG=y`
- `CONFIG_LOG_BACKEND_UART=y`
- `CONFIG_LOG_DEFAULT_LEVEL=3`
- `CONFIG_DEBUG=y`
- `CONFIG_DEBUG_OPTIMIZATIONS=y`
- `CONFIG_ASSERT=y`
- `CONFIG_ASSERT_VERBOSE=y`
- `CONFIG_DEBUG_THREAD_INFO=y`

Do **not** set `CONFIG_LOG_OVERRIDE_LEVEL=4` or `CONFIG_LOG_DEFAULT_LEVEL=4`: either setting causes Zephyr's `os` module (`kernel/mutex.c`) to emit `LOG_DBG` on every `k_mutex_lock`/`k_mutex_unlock` inside `zb_sched_loop_iteration()`, saturating `zboss_thread` and overflowing `log_buffer`. Enable `LOG_LEVEL_DBG` per-module in application files when needed.

UART monitor:

```sh
screen /dev/cu.usbmodem11102 115200
```

Wire XIAO D6/TX to the debug probe UART RX and connect grounds. Debug logging materially increases firmware size and power consumption; revert to the production profile before battery-life measurements.

## Source architecture

- `src/zb_main.c`: initialization and top-level startup decisions.
- `src/zb_main_loop.c`: main event loop (`SHALL_ENABLE_ZIGBEE`, `SHALL_MEASURE_BATTERY`, `SHALL_CONFIRM_OTA`; ensure all handled events are included in `main_loop_event_mask()`).
- `src/zb_main_button.c`: GPIO interrupt and button gesture state machine.
- `src/zb_reed.c`: reed GPIO, debounce/stability checks, and pulse handling (registers GPIO callback before enabling edge interrupt; disables interrupt, removes callback, and persists `next_reed_level` before arming `SYSTEMOFF` wake).
- `src/zb_zigbee.c`: Zigbee lifecycle, commissioning, callbacks, `ZB_ZCL_SET_ATTR_VALUE_CB_ID` handling, leave/FOTA behavior.
- `src/zb_zigbee_ctx.c`: endpoint, cluster, attribute, and reporting context declarations. Tracks raw pulse count (`counter_raw`, NVS key `"gm/counter"`) separately from user-configured offset (`curr_summ_delivered_offset`, NVS key `"gm/offset"`), reporting `counter_raw + curr_summ_delivered_offset` as `CurrentSummationDelivered`.
- `src/zb_radio.c`: radio/report transport behavior.
- `src/zb_deep_sleep.c`: System OFF policy and poweroff blockers.
- `src/zb_light_sleep.c`: conventional always-reachable end-device mode.
- `src/zb_nvr.c` and `src/zb_retained.c`: persistent and retained state.
- `src/zb_ota.c`: MCUboot image confirmation and OTA lifecycle.
- `VERSION.json`: single source of truth for application/Zigbee OTA version metadata.

Deep-sleep-only APIs and `POF_BLOCK_*` constants must always be guarded by `#ifdef FEATURE_DEEP_SLEEP`; light-sleep builds must compile independently.

Only one source file may use `LOG_MODULE_REGISTER()` for a given module name. Additional files sharing that module must use `LOG_MODULE_DECLARE()`; `zb_zigbee.c` registers `zigbee`, while `zb_zigbee_ctx.c` declares it.

## Coding standards

- Follow existing Zephyr/NCS C conventions and preserve the current indentation style in each file.
- Use devicetree aliases and `gpio_dt_spec` rather than hardcoded GPIO access in C.
- Keep ISR callbacks short: capture state or release a semaphore and perform work in threads/work queues.
- Preserve the local cumulative counter independently of coordinator availability.
- Treat retained RAM, NVS, and Zigbee reporting synchronization as concurrency-sensitive.
- Keep deep-sleep blockers balanced: every set/begin operation needs a clear/done path, including errors.
- Avoid changing Zigbee manufacturer ID, image type, endpoint, or model identity without checking Zigbee2MQTT and OTA compatibility.
- Update versions only through `VERSION.json`; generated headers/configuration must not be edited manually.
- Never use the default MCUboot development signing key for production deployment. Configure and protect a project-specific key before production use.
- Do not edit files under `/opt/nordic/ncs`; project adaptations belong in this repository.

## Review and verification checklist

Before accepting firmware changes:

1. Build a pristine XIAO sysbuild configuration with NCS 3.4.0 and ncs-zigbee v1.4.0.
2. Confirm both MCUboot and application compile.
3. Confirm the combined HEX and `.zigbee` OTA artifact are generated.
4. Review flash/RAM usage against the 916 KiB primary slot and available RAM.
5. Verify no application and MCUboot partition mismatch exists.
6. Test cold boot through MCUboot after full-chip SWD programming.
7. Verify button press/release and reed active/inactive transitions, including debounce.
8. Verify Zigbee commissioning, rejoin, reporting, and five-click leave behavior.
9. Verify cumulative count survives reset and temporary coordinator loss.
10. If deep sleep is enabled, verify wake from both P0.02 and P0.03 and ensure no poweroff blocker remains stuck.
11. Verify OTA download, reboot, swap, and image confirmation before deploying OTA artifacts.
12. Recheck production settings: deep sleep restored as intended, verbose logs/debug disabled, and a non-development MCUboot key used.
