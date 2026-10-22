# Linux on the ESP32-S31, in Wokwi

Espressif's Linux image for the ESP32-S31 booting to a root shell in the [Wokwi](https://wokwi.com) simulator, plus a CI setup that checks the boot on every push.

## What's in here

- `s31_full_flash.bin` is the 16 MB flash image Espressif's build produces: U-Boot SPL, OpenSBI, U-Boot, a Linux 6.18 kernel (RISC-V, `sv32` paging, executing in place from flash) and a read-only busybox root filesystem. It is built from the unmodified [`meta-espressif`](https://github.com/espressif/meta-espressif) Yocto layer, branch `integration/wrynose-esp32s31` at commit `a12af0f`, with `bitbake esp-image-core`. 
- `diagram.json` puts the image on an ESP32-S31 coreboard with 16 MB of flash and 16 MB of octal PSRAM, and loads it at flash offset 0 (`firmwareOffset`), where the ROM expects the SPL.
- `wokwi.toml` points Wokwi for VS Code and the Wokwi CLI at the image.
- `scenarios/linux-boot.test.yaml` watches the console through the boot chain, presses Enter when busybox asks for it, and runs `uname -a` in the shell.
- `.github/workflows/linux-boot.yml` runs that scenario on every push.

## Quick look, in the browser

[**Run online (no install)**](https://wokwi.com/experimental/viewer?diagram=https://raw.githubusercontent.com/wokwi/esp32-s31-linux-demo/main/diagram.json&firmware=https://raw.githubusercontent.com/wokwi/esp32-s31-linux-demo/main/s31_full_flash.bin)

The viewer downloads the 13 MB image and starts the chip. Watch U-Boot and the kernel log scroll by in the serial monitor, press Enter when you see `Please press Enter to activate this console.`, and you have a root shell:

```
Please press Enter to activate this console.
root@esp32s31:~# uname -a
Linux esp32s31 6.18.0 #1 Thu Aug 20 03:59:26 UTC 2026 riscv32 GNU/Linux
root@esp32s31:~# cat /proc/interrupts
```

Things worth a look once you are in: `cat /proc/cpuinfo`, `cat /proc/interrupts` (the systimer tick and the UART come in through the S-mode CLIC), `free`, `mount`, `cat /proc/iomem`.

## Wokwi for VS Code

Install the [Wokwi for VS Code](https://marketplace.visualstudio.com/items?itemName=wokwi.wokwi-vscode) extension, open this folder, press `F1` and run *Wokwi: Start Simulator*. The first time, the extension asks for a license: `F1`, *Wokwi: Request a new License*, and confirm in the browser. The serial monitor opens in the editor; press Enter there to get the shell.

## Wokwi CLI

Install the [Wokwi CLI](https://docs.wokwi.com/wokwi-ci/cli-installation) and get a token at [wokwi.com/dashboard/ci](https://wokwi.com/dashboard/ci):

```sh
curl -L https://wokwi.com/ci/install.sh | sh
export WOKWI_CLI_TOKEN=...
```

Run the boot test:

```sh
wokwi-cli . --scenario scenarios/linux-boot.test.yaml --timeout 30000
```

You'll see the Linux kernel boot messages, followed by the validation steps from the scenario. The end of the output should look as follows:

```
Please press Enter to activate this console.
[Linux boots to a shell] Expected text matched: "Please press Enter to activate this console."
root@esp32s31:~#
[Linux boots to a shell] Expected text matched: "root@esp32s31:~#"
 uname -a
Linux esp32s31 6.18.0
[Linux boots to a shell] Expected text matched: "Linux esp32s31 6.18.0"
[Linux boots to a shell] Scenario completed successfully
```

For a shell of your own, connect your terminal to the console instead of a scenario:

```sh
wokwi-cli . --interactive --timeout 600000
```

## GitHub Actions

`.github/workflows/linux-boot.yml` checks out the repository and runs the scenario with [`wokwi/wokwi-ci-action`](https://github.com/wokwi/wokwi-ci-action); there is nothing to build. The console log is uploaded as an artifact. To run it in your copy of the repository, add your CLI token as a repository secret named `WOKWI_CLI_TOKEN`.

## Rebuilding the image

To build your own image, follow the [`meta-espressif` README](https://github.com/espressif/meta-espressif): clone `bitbake` and the layer side by side, run `bitbake-setup init` for `esp32s31`, source the build environment and run `bitbake esp-image-core`. A first Yocto build can take a few hours. The build writes `s31_full_flash.bin` to `build/tmp/deploy/images/esp32s31/`. Copy it over the one in this repository and enjoy!
