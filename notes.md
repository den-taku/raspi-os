# Notes
# Preparations

- `add aarch64-unknown-none-softfloat`
- `rustup component add llvm-tools-preview`

# 00

boot flow: `cpu::boot::arch_boot::_start()` in `src/_arch/__arch_name__/cpu/boot.s`

`bsp`: board support package

## 01

[Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)

ld: linker script

ブートプロセス：[ref: [Raspberry Pi 3 Model Bのブート周り](https://bunkyu3.hatenablog.com/entry/2018/10/21/185206)]
1. SDカードに`bootcode.bin`，`start.elf`，`config.txt`，`kernel.img`を置いておく
2. 電源ON
3. GPUが起動
4. CPUがSoS内ROMから第1段ブートローダを実行
5. 第1段BLは第2段BL（`bootcode.bin`）をキャッシュにロードしそこにジャンプ
6. 第2段BLがSDRAM（Synchronous Dynamic RAM）を有効化
7. 第2段BLがGPUファームウェア（`start.elf`）をSDRAMにロード
8. GPUファームウェアにジャンプ
9. GPUFWが`config.txt`を読みCPUの設定を行う
10. GPUFWが`kernel.img`を読みSDRAMにロード
11. GPUFWがCPUのリセット信号を有効化
12. CPUがSDRAMに置かれた`kernel.img`を実行

```bash
$ ls /Volumes/boot
COPYING.linux			bootcode.bin			issue.txt
LICENCE.broadcom		cmdline.txt			kernel8.img
bcm2710-rpi-2-b.dtb		config.txt			overlays
bcm2710-rpi-3-b-plus.dtb	firstrun.sh			start.elf
bcm2710-rpi-3-b.dtb		fixup.dat			start4.elf
bcm2710-rpi-cm3.dtb		fixup4.dat			start4cd.elf
bcm2710-rpi-zero-2-w.dtb	fixup4cd.dat			start4db.elf
bcm2710-rpi-zero-2.dtb		fixup4db.dat			start4x.elf
bcm2711-rpi-4-b.dtb		fixup4x.dat			start_cd.elf
bcm2711-rpi-400.dtb		fixup_cd.dat			start_db.elf
bcm2711-rpi-cm4.dtb		fixup_db.dat			start_x.elf
bcm2711-rpi-cm4s.dtb		fixup_x.dat
```
