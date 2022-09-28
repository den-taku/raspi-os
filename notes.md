# Notes
## Preparations

- `bundle install --path .vendor/bundle --without development`

（`rust-toolchain.toml`を追加したので不要かも）

- `add aarch64-unknown-none-softfloat`
- `rustup component add llvm-tools-preview`

## 00

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

## 02

linker script
- `ENTRY`：entry addressを決める（ref [3.4.1 Setting the Entry Point](https://sourceware.org/binutils/docs-2.38/ld/Entry-Point.html)）
- `PHDRS`：プログラムヘッダを指定してロードの仕方を決める（ref [3.8 PHDRS Command](https://sourceware.org/binutils/docs-2.38/ld/PHDRS.html)）
- `SECTIONS`（必須）：メモリ配置を定める（ref [3.6 SECTIONS Command](https://sourceware.org/binutils/docs-2.38/ld/SECTIONS.html)）
  - `.text`：プログラム
  - `.rodata`：定数値
  - `.data`：初期値ありグローバル変数
  - `.bss`：初期値なし（0で初期化）グローバル変数
  - `.got`：共有ライブラリ関数

## 03

`0x3F20_1000`：UARTのbase address (Rapberry Pi 3)

## 04

- `UnsafeCell`：`&T`から`&mut T`を生み出すunsafeな型（`pub fn get(&self) -> *mut T;`などの機能）
- `Cell`：値への不変参照が存在しない．内部で可変参照をとる．スレッドセーフではない．
- `RefCell`：オーバーヘッドありで参照，可変参照と取れる．危険な場合はpanicする．

## 05

for my MacOS: `DEV_SERIAL=/dev/tty.usbserial-AU01PR2F make miniterm`

Raspberry Pi 4のプロセッサは`bcm2711`（仕様PDF：[bcm2711/bcm2711-peripherals.pdf](https://datasheets.raspberrypi.com/bcm2711/bcm2711-peripherals.pdf)）

Device Drivers
- `GPIO`
- `PL011Uart`

レジスタの読み書きには[tock-registers](https://crates.io/crates/tock-registers)を使用

`src/bsp/raspberrypi/memory.rs` manages memory-mapped I/O register

### `GPIO`

仕様の65ページから

概要
- GPUから見たベースアドレスが`0x7E20_0000`→CPUからは`0xFE20_0000`（P.5 Figure1の対応関係より）
- Function Select Regs
  - GPFSEL1レジスタ（Offset: 0x04）
    - フィールド
      - [15:7]：FSEL15
      - [14:12]：FSEL14
    - 値
      - `0b000`：Input
      - `0b001`：Output
      - `0b100`：alternate function 0
- Pin Set & Clear Regs
  - CPIO_PUP_PDN_CNTRL_REG0レジスタ（Offset: 0xE4）
    - フィールド
      - [31:30]：GPIO_PUP_PDN_CTRL15
      - [29:28]：GPIO_PUP_PDN_CTRL14
    - 値
      - `0b00`：No registor
      - `0b01`：Pull up registor
      - `0b10`：Pull down registor
- Alternative Function Assignments
  - GPIO14::ALT0：TXD0
  - GPIO15::ALT0：RXD0

### `PL011Uart`（ARM UART）

仕様の144ページから

概要
- PL01 UART
  - 非同期，FIFOを利用可能
  - Input: `RXD`（，`nCTS`）
    - GPIO14, ALT0
  - Output: `TXD`（，`nRTS`）
    - GPIO15, ALT0
- GPUから見たベースアドレスが`0x7E20_1000`→CPUからは`0xFE20_1000`（P.5 Figure1の対応関係より）（UART0）
  - Data Register（Offset: 0x00）
    - [7:0]：データ
    - [11:8]：エラーフラグ
  - Flag register（Offset: 0x18）
    - [7]：送信FIFOがEmpty
    - [5]：送信FIFOがFull
    - [4]：受信FIFOがEmpty
    - [3]：Busy（送信中）
  - Integer Baud rate divisor（Offset: 0x24）
    - [15:0]：値
  - Fractional Baud rate divisor（Offset: 0x28）
    - [5:0]：値
  - Line Control register（Offset: 0x2c）
    - [6:5]：ワード長
      - `0b00`：5bit，`0b01`：6bit，`0b10`：7bit，`0b11`：8bit
    - [4]：FIFOを有効化
  - Contrel register（Offset: 0x30）
    - 設定方法
      1. [0] := 0
      2. 送信終了を待つ
      3. FIFOをflush
      4. 設定後，[0] := 1
    - [9]：受信を可能に
    - [8]：送信を可能に
    - [0]：UARTを可能に
  - Interrupt Clear Register（Offset: 0x44）
    - [10:0]：*IC

## 06

chainloader
1. 自身（`chainloader`）が`0x8_0000`にロードされ実行開始
2. `chainloader`を`0x200_0000`スタートの場所にコピー
3. `0x200_0000`スタートのプログラム中の`_start_rust`にジャンプ
4. `chainloader`はUART経由で`\x03\x03\x03`を送る
5. `\x03\x03\x03`を受け取った`Miniload`は`kernel`のバイナリを送信し始める
6. `chainloader`が`kernel`を`0x8_0000`にロード
7. `0x8_0000`にジャンプして`kernel`を実行する
