---
layout: post
title:  自作RISC-Vエミュレータに潜んでいたバグたち
date:   2020-12-15 9:00 +0900
categories: risc-v
---

この記事は[自作OS Advent Calendar 2020](https://adventar.org/calendars/4954)の15日目の記事です。本記事は、現在開発中のRISC-Vエミュレータの上で、Linuxを動かすために開発しているときに見つけたエミュレータのバグについて書きます。"自作"エミュレータの上で既存"OS"を動かす話なので、"自作OS"ではないですが、アドベントカレンダーに参加させていただきます。

## 自作RISC-Vエミュレータの現状

1年以上前からWebでもコンソールでも動くRISC-Vエミュレータ（[rvemu](https://github.com/d0iasm/rvemu)）の開発を行っています。現状は、[xv6を動かす](https://d0iasm.github.io/blog/risc-v/2020/04/02/xv6-on-my-riscv-emulator.html)ことができたり、[Linuxを途中まで動かす](https://twitter.com/d0iasm/status/1335551481542406148)ことができます。

エミュレータがサポートしている機能は以下の通りです。

- **RV64GC**: OSを動かすために十分なISA。基本演算、掛け算・割り算演算、浮動小数演算、アトミック命令などを含みます。
- **特権モード**: CPUの動作モードでCPUが実行できる操作を制限することができる。マシンモード、スーパーバイザーモード、ユーザーモードの3種類を実装。
- **関連するステータスレジスタ**: CPUの状態を保存するレジスタ。仕様で決められているうち、xv6とLinuxで使用されている一部を実装。
- **仮想メモリ機構**: Sv32、Sv39、Sv48の3種類のアドレス変換方法が定義されているがSv39のみをサポート。仮想メモリのアドレス幅が39ビット。
- **周辺機器**
  - **UART**: CPUが外部とシリアル通信をするための機器。
  - **CLINT**: CPU内部からの割り込み（local interrupt）を制御するための機器。タイマー割り込みなどを制御する。
  - **PLIC**: CPU外部からの割り込み（glocal interrupt）を制御するための機器。ユーザーからの入力などを制御する。
  - **Virtio (block disk)**: 仮想的な外部記憶装置。
- **デバイスツリー**: 周辺機器の情報を保持するバイナリ。通常はブートローダからLinuxに渡される。

### Linuxを動かしたい

Linuxを自作エミュレータ上で動かすことが今の目標ですが、まだ完成には至っていません。機能的には上記であげたもので十分なはずなのですが、バグが潜んでいたり、未実装な部分があるために、現在はカーネルの実行が途中で止まります。デバッグして、バグを見つけ、直す、をひたすら繰り返して進めていくうちに、いくつか面白いバグを自作エミュレータに見つけました。

本記事で使用するLinuxカーネルはv4.19-rc3のバージョンで、[この記事](https://risc-v-getting-started-guide.readthedocs.io/en/latest/linux-qemu.html)に沿ってビルドしました。

## 自作エミュレータ上のバグ

### **Bug 1**: console_init()で無限ループ

Linuxカーネルにはさまざまな初期化処理を行う`start_kernel`関数があります。その中で[`console_init`関数を呼び出す](https://elixir.bootlin.com/linux/v4.19-rc3/source/init/main.c#L661)と、その先に実行が進まなくなってしまいました。エミュレータが実行しているプログラムカウンタから実行している箇所を調べると、無限ループに陥っているようです。

`console_init`関数は文字を出力するためのコンソールを初期化する関数です。本来ならば、文字の入出力にはPLIC（CPU外部からの割り込みを制御するための機器）が必要なのですが、この時点ではPLICの初期化はまだ行われていません。早めにコンソールの初期化をはじめるのは、少しでも早く文字出力を行ってデバッグなどを行いやすくするためです。

### **Fix 1**: デバイスツリーのbootargsでコンソールの指定

QEMUと自作エミュレータのブートログの出力を見比べていてバグを発見しました。QEMUではKernel comamnd lineが `root=/dev/vda ro console=ttyS0`となっていましたが、自作エミュレータでは空でした。

```
// log in QEMU
[    0.000000] Kernel command line: root=/dev/vda ro console=ttyS0 
```

```
// log in my emulator
[    0.000000] Kernel command line:   
```

このコマンドラインはデバイスツリーの`chosen`ノードの`bootargs`パラメータから渡すことができます。`chosen`ノードは実際のデバイスを表すものではなく、ファームウェアとOS間でデータをやりとりするためのノードです。そして`bootargs`はLinuxに渡すコマンドライン引数です。`root=/dev/vda`によってルートファイルシステムの位置を指定しており、`console=ttyS0`は出力すべきコンソールデバイスを指定しています。

この無限ループのバグは、初期化すべきコンソールが永遠に見つからないことが原因でした。デバイスツリーの`chosen`ノードを以下のようにすることでこのバグを解決することができました。

```
chosen {
    bootargs = "root=/dev/vda ro console=ttyS0";
    stdout-path = "/uart@10000000";
};
```

QEMUのvirtマシンを参考にして作っているため、デバイスツリーもQEMUを参考にしました。QEMUは以下のコマンドによってデバイスツリーのバイナリを出力することができます。更に、`dtc`コマンドによってこのバイナリを文字列の形式に変換することもできます。

```
// デバイスツリーのバイナリを出力
$ qemu-system-riscv64 -nographic -machine virt,dumpdtb=virt.dtb
// バイナリから文字列へ変換
$ dtc -I dtb -O dts virt.dtb > virt.dts
```

このデバイスツリーをみたところ、`chosen`ノードの`bootargs`は以下のように空でした。これを単純に真似したことがバグを生み出しました。

```
chosen {
    bootargs = [00];
    stdout-path = "/uart@10000000";
};
```

QEMUではQEMUの実行時に`-append "root=/dev/vda ro console=ttyS0"`のパラメータを渡すことで、動的にデバイスツリーのバイナリを変更しているようです。

### **Bug 2**: vfs_caches_init()で無限ループ

再び無限ループです。次は`start_kernel`関数の中の[`vfs_caches_init`関数を呼び出す](https://elixir.bootlin.com/linux/v4.19-rc3/source/init/main.c#L717)と、その先に実行が進まなくなってしまいました。

以下の実行バイナリの`0xffffffe000346d48`と`0xffffffe000346d4c`の間で無限ループになっています。

```
ffffffe000346d42 <_raw_write_lock>:
ffffffe000346d42:       1141                    addi    sp,sp,-16                                                      
ffffffe000346d44:       e422                    sd      s0,8(sp)                                                       
ffffffe000346d46:       0800                    addi    s0,sp,16                                                       
ffffffe000346d48:       100527af                lr.w    a5,(a0)                                                        
ffffffe000346d4c:       fff5                    bnez    a5,ffffffe000346d48 <_raw_write_lock+0x6>                      
ffffffe000346d4e:       57fd                    li      a5,-1                                                          
...  
```

### **Fix 2**: Load-Reserved/Store-Conditional命令を正しく実装

`_raw_write_lock`関数はロックを取得しようとしている関数です。ロックとは排他制御を行うための機構で、複数のプロセスやスレッドが環境で、リソースへのアクセスを制限することができます。

今回はロックの中でもスピンロックを使用していました。スピンロックはロックが獲得できるまでループして定期的にロックをチェックしながら待つ方式です。このため、永遠にロックが獲得できないと、今回のように無限ループに陥ってしまいます。

RISC-Vではロックのような機構を実装するためにアトミック命令を用意しています。今回はそのうちの`lr`と`sc`命令の実装が不足していました。`lr`命令はLoad-Reservedの略で、指定されたメモリの位置の値を返すことができます。そのときに指定されたメモリの位置を保存しておきます。`sc`命令はStore-Conditionalの略で、指定されたメモリの位置に値を書き込むことができます。ただし、前回の`lr`命令以降にそのメモリ位置の内容が書き換えられていないときだけ書き込みが成功します。

以前はこれらの命令をアトミックではなく単なるロード・ストア命令として実装していたため、常に`sc`命令が成功していました。よって、複数のスレッドが同じリソースへのロックを持つことができてしまい、デッドロックのような状態に陥ってしまったのではないかと思います。

エミュレータ上で`lr`命令を実行するときに指定されたメモリの位置を保存するようにし、`sc`命令を実行するときに指定されたメモリの位置がすでに保存されているかチェックするようにしました。これにより無限ループのバグは解決できました。

## おわりに

進捗は[d0iasmのTwitter](https://twitter.com/d0iasm)でつぶやいています。はやくLinuxのユーザーログインまでたどり着きたいです！