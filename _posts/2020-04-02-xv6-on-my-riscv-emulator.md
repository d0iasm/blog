---
layout: post
title:  xv6が動くRISC-Vエミュレータを作った
date:   2020-04-02 9:00 +0900
categories: risc-v
---

教育用のシンプルなOSである[xv6][xv6-riscv]が動くRISC-Vエミュレータを作成しました。エミュレータのソースコードは全て[d0iasm/rvemu][rvemu]のリポジトリで公開しています。本記事では、OSを動かすまでに実装したエミュレータの機能について、大きな変更をしたコミットのソースコードをたどることによって振り返ります。

注意：あとから実装のミスに気付いて直すことを繰り返しているため、各時点のソースコードが必ずしも正しい実装とは限りません。

### 2019年10月22日 ([143c7d5: src/lib.rs][1022])
リポジトリを作成して初めてのコミット。勉強のためにRustで開発したい、そして、エミュレータをブラウザで動かすためにWebAssemblyにコンパイルしたいと考えていたため、[Rust and WebAssembly][rust-and-webassembly]のチュートリアルにあるテンプレートを使用して環境を整えた。`src/lib.rs`内でimportしている[wasm-bindgen][wasm-bindgen]がRustとJavaScriptをブリッジしてくれるライブラリである。

エミュレータとしての機能はまだ何もない。

### 2019年10月30日 ([2274ebf: src/cpu/instruction.rs][1030])
2つのレジスタを足し算する`add`と、レジスタと12ビットの即値を足し算する`addi`命令を実装した。

RISC-Vは仕様が全てオープンであり、[riscv.org/specifications][riscv.org/specifications]から仕様書のPDFをダウンロードできる。まずはこの中の*Volume I: Unprivileged ISA*を読んで、CPUの命令を一つずつ実装していくことにした。

コンピュータにおけるCPUの役割は、主にフェッチ、デコード、実行の3ステップによって、0と1のバイナリで構成されるプログラムを順に実行することである。
1. フェッチ (Fetch): プログラムが保存されているメモリから次に実行すべき命令を取り出す。
2. デコード (Decode): 命令列をCPUにとって意味のある形式に分割する。命令をどのように解釈するかは[riscv.org/specifications][riscv.org/specifications]の*Volume I: Unprivileged ISA*に定義されている。
3. 実行 (Execute): デコードによって指定された操作を実行する。ハードウェアならば足し算引き算などの算術演算はALU (Arithmetic logic unit) と連携して操作を行うが、エミュレータでは細かいコンポーネントを気にしないで実装することにした。

### 2019年11月11日 ([2d59abc: src/cpu.rs][1111])
32ビットアーキテクチャ用の基本整数演算命令のRV32Iを実装した。

RISC-VのISAは全てのプラットフォームが実装すべき基本整数演算命令に対して、用途に合わせて他の命令群を拡張することができる。32ビットアーキテクチャ用の基本整数演算をRV32I、64ビットアーキテクチャ用の基本整数演算をRV64Iと呼ぶ。

*Volume I: Unprivileged ISA*では、よく使用される拡張命令群をまとめてRV32G/RV64G (general-purpose ISA) と呼ぶ。これは基本整数演算命令と、以下の6種類の拡張命令群を含む。エミュレータはRV64Gの命令を実装することを試みる。
- RV32I/RV64I: 整数演算、ビット演算、シフト演算、メモリへのロードとストア、分岐命令など最も基本的な命令
- RV32M/RV64M: 掛け算、割り算の命令
- RV32A/RV64A: アトミック命令
- RV32F/RV64F: 単精度浮動小数点数 (Float) の命令
- RV32D/RV64D: 浮動小数点数 (Double) の命令
- Zicsr: コントロールレジスタを操作するのための命令
- Zifencei: フェンス命令 (メモリの書き込み順序を保証するための命令)

RV32Iでは40の命令を定義しており、今回は`fence`、`ecall`、`ebreak`を除く37命令を実装した。`fence`命令は複数のハードウェアスレッドが存在するときに、スレッド間でメモリの内容の一貫性を保つための命令である。ハードウェアスレッドとは、複数のスレッドの実行をハードウェアで提供することである。複数のハードウェアスレッドが存在するとは、つまり、次に実行すべき命令の位置を保存しているプログラムカウンタを複数持つことである。ハードウェアスレッドの数が多いほど同時に実行できるプログラムが増えるため高速化に繋がる。

エミュレータは1つのハードウェアスレッドしか実装していないので、今のところは`fence`命令は不必要であると判断した。また、`ecall`と`ebreak`は*Volume II: Privileged Architecture*で定義されている特権モードを実装するときに必要になるが、この時点ではまだ実装していない。

### 2019年11月18日 ([caea7c6: src/cpu.rs][1118])
レジスタを64ビットに拡張し、64ビットアーキテクチャ用の基本整数演算命令であるRV64Iを実装した。RV64IはRV32Iの命令に対し12の命令を新たに追加したものである。基本的な操作はRV32Iと同じだが、有効なビット幅が異なる。例えば、RV32Iの`add`命令は32ビット幅のレジスタ2つの足し算だが、RV64Iの`add`命令は64ビット幅のレジスタ2つの足し算である。32ビット幅の足し算をするために、RV64Iでは`addw`という命令が追加されている。

### 2019年11月19日 ([44c0584: src/cpu.rs][1119])
掛け算、割り算の命令を定義するRV64Mを実装した。

掛け算では結果が64ビットに収まらない大きい数になった場合 (これは arithmetic overflow と呼ばれる)、下位の64ビットだけをレジスタに保存し、上位のビットは無視する。また、割り算では0による割り算の結果として全てのビットを立てる。このように、RISC-Vでは算術演算による例外を発生しないことにしている。なぜなら、例外処理をするためには*Volume II: Privileged Architecture*で定義されている例外に関する機構を実装する必要があり、ハードウェアを小さくすることの弊害になり得るからだ。

### 2019年12月30日 ([67558bd: src/cpu.rs][1230])
32ビット幅の単精度浮動小数点数 (float) を定義するRV64Fを実装した。浮動小数点数はIEEEによって標準化されている[IEEE 754-2008][IEEE 754-2008]に準拠している。

RISC-VのCPUは整数のレジスタを32個持っているのだが、浮動小数点数を拡張する場合には追加で32個の浮動小数点数のレジスタを持つことになる。更に`fcsr`という状態を保持するステータスレジスタも持つ。`fcsr`は丸め処理をどのように行うかと、演算による例外発生が起きたかのフラグを保持する。例えば、0除算が起こった場合は`fcsr`の3ビット目 (DZフラグ) がセットされる。

丸め処理は以下のように5種類の方法が定義されているが、今のところちゃんと実装していない。(以下は[wikipedia IEEE 754][IEEE wiki]より引用)
- 最近接丸め（偶数）: 最も近くの表現できる値へ丸める。表現可能な2つの値の中間の値であったら、一番低い仮数ビット（桁）が0になるほうを採用する。これは二進での標準動作かつ十進でも推奨となっている
- 最近接丸め（0から遠いほうへ）: 最も近くの表現できる値へ丸める。表現可能な2つの値の中間の値であったら、正の値ならより大きいほう、負の値ならより小さいほうの値を採用する
- 0方向への丸め: 0に近い側へ丸める。切り捨て (truncation) とも呼ばれる
- +∞への丸め: 正の無限大に近い側へ丸める。切り上げ (rounding up, ceiling) とも呼ばれる
- −∞への丸め: 負の無限大に近い側へ丸める。切り下げ (rounding down, floor) とも呼ばれる

### 2020年1月2日 ([83ef66c: src/cpu.rs][0102])
64ビット幅の倍精度浮動小数点数 (double) を定義するRV64Dを実装した。浮動小数点数はIEEEによって標準化されている[IEEE 754-2008][IEEE 754-2008]に準拠している。

基本的には単精度浮動小数点数 (RV64F) と同じである。今回も丸め処理はちゃんと実装していない。（いつかやる…）

### 2020年1月5日 ([533ea69: src/cpu.rs][0105])
アトミック命令を定義するRV64Aを実装した。

アトミック命令とは複数のハードウェアスレッド間でメモリの一貫性を保つための命令である。有名なものとして、x86では[compare-and-swap (CAS)][cas]と呼ばれるアトミック命令が定義されている。compare-and-swap命令ではあるメモリ位置の内容と指定された値を比較し、等しければそのメモリ位置に別の指定された値を格納する。つまり、とあるメモリ位置の内容が変更されていないことを保証することができる。しかし、値が変更されていないことだけを確認するため、他のスレッドが元々の値Aを異なる値Bに変更したあと、また元の値Aに書き直すと値がAのままに見えるため、「変更がなかった」とcompare-and-swap命令を行うスレッドが誤認してしまうことがある。この問題を[ABA問題][ABA problem]という。

RISC-VではABA問題を避けるために、compare-and-swapの代わりに[load-reserved (LR) 命令とstore-conditional (SC) 命令][lrsc]を採用した。load-reserved命令では指定されたアドレスから値を読み、レジスタに値を保存する。その際にアドレスを予約する。store-conditional命令では、予約されたアドレスに何か変更があった場合、アドレスへの書き込みが失敗する。アドレスの値をチェックするcompare-and-swapとは異なり、load-reserved/store-conditionalではアドレス自体を監視するためABA問題を回避できる。

他にもアトミックに加算をする命令などが存在する。しかし、エミュレータではまだ1CPU (1スレッド) の実装なのでアトミック性は実装していない。（どのように実装すれば良いのかもまだよくわかっていない。）

### 2020年2月14日 ([a751a6b: src/cpu.rs][0214])
RISC-VはCPUの状態を保存するステータスレジスタ (CSR) を最大で4096個持つことができる。ステータスレジスタを読み書きするRV64Zicsrを実装した。

ステータスレジスタは[RISC-Vの仕様書][riscv.org/specifications]の*Volume II: Privileged Architecture*に定義されている。

### 2020年2月18日 ([ba64dfa: src/exception.rs][0218])
例外処理の一部を実装した。

RISC-Vでは例外、割り込み、トラップの用語を以下のように定義している。これらの用語はアーキテクチャや教科書によって異なる使われ方をしていることがあるが、本記事ではRISC-Vの定義に従う。
- 例外 (Exception): プログラム実行中にCPU内部で発生する同期イベント。例えば、サポートされていない命令を実行しようとすると、例外が発生する
- 割り込み (Interrupt): CPU外部で発生する非同期イベント。例えば、ユーザのキーボード入力などは割り込みとしてCPUに通知される
- トラップ (Trap): 例外や割り込みが発生したさいに実行がトラップハンドラに移ること。トラップハンドラにはトラップが発生したときに実行するコードが格納されており、OSが用意する

例外の種類は現時点で14種類あり、例外が発生したときに`mcause`ステータスレジスタに例外の種類を示す値を保存する。

### 2020年2月22日 ([6b6f7f9: src/cpu.rs][0222])
特権レベルを遷移する`mret`と`sret`命令を実装した。

RISC-Vにおける特権レベルには現時点でマシンモード (M-mode)、ユーザモード (U-mode)、スーパーバイザーモード (S-mode) の3種類存在する。全てのプラットフォームは必ず最も原始的なマシンモードをサポートしなければならず、この時点までのエミュレータはマシンモードを前提として動いていた。

マシンモードのみを実装したハードウェアは、仮想アドレスなどの機能を持たずCPUはプログラムの命令を順に実行していくだけのシンプルものである。組み込みなどの分野で複雑な機能は必要ないが、ハードウェアを小さくしたい場合などには重宝される。対して、OSなどの複雑なシステムを動かしたい場合は、3種類のモードを全て実装する必要がある。

今回実装した`mret`命令はマシンモード他の特権レベルに遷移するための命令である。遷移先の特権レベルは`mstatus`ステータスレジスタの`mpp`フィールドの値によって決定する。`mpp`フィールドの値が0ならユーザモード、1ならスーパーバイザーモード、3ならマシンモードに遷移する。`sret`命令は同様にスーパーバイザーモードから他の特権レベルに遷移するための命令である。

### 2020年3月4日 ([859d9fa: src/exception.rs][0304])
例外をあげることによって特権レベルを遷移する`ecall`を実装した。

`ecall`命令は、現在の特権レベルがマシンモードなら`environment-call-from-M-mode`、スーパーバイザーモードなら`environment-call-from-S-mode`、ユーザモードなら`environment-call-from-U-mode`の例外を発生させる。そしてこれらの例外をトラップするときに、より権限の高いモードに遷移する。基本的には`ecall`が発生するとマシンモードに遷移するが、`medeleg`と`mideleg`のステータスレジスタの値によってスーパーバイザーモードに権限を移譲することができる。

`ecall`と`mret`/`sret`は補完関係にある。以下に特権レベル遷移の簡略化した概要図を載せる。
![privilege levels transition]({{ site.url }}/blog/assets/03-31-privilege_levels_transition.png)

### 2020年3月9日 ([7cc38ab: src/devices/uart.rs][0309])
CPUが外部とシリアル通信するための周辺機器であるUART (universal asynchronous receiver-transmitter) を実装した。

RISC-Vは周辺機器とメモリマップドI/Oの手法によってやり取りする。メモリマップドI/Oとは、周辺機器のレジスタが存在する特定のアドレスに読み書きすることによってデータをやり取りする手法である。CPUからはメモリアクセスと同じように扱うことができる。特定のアドレスの位置はRISC-Vのの仕様には載っておらず、（おそらく）マザーボードを設計した人が決める。今回はQEMUのvirtマシンと同じように実装した。

[QEMUの実装][qemu virt]を見ると、UARTは`0x10000000`の位置から`0x100`の大きさだけマップされていることがわかる。また、今回動かそうとしている[xv6][xv6-riscv]の[UARTの実装][xv6 uart]を見ると、"16550a UART"を採用しており、この[UARTの仕様][uart spec]はウェブからアクセスすることができる。

仕様では5種類のレジスタが存在しているが、全ての機能は実装せずxv6で使われている機能だけを実装した。

### 2020年3月13日 ([ea77341: src/cpu.rs][0313])
仮想メモリ機構によるアドレス変換を実装した。メモリにアクセスする前に、[`translate`][translate]関数でアドレスを変換することにした。

今まではCPUがアドレスを指定すると、そのアドレスに手を加えることなくメモリやメモリマップドされた周辺機器とやり取りした。実際にメモリや周辺機器が存在するアドレスを物理アドレスと言う。仮想メモリ機構が有効化されると、CPUが扱うアドレスは仮想アドレスと呼ばれ、アドレスを物理アドレスに変換する必要がある。仮想メモリ機構は`satp`ステータスレジスタの`mode`フィールドの値によって有効化される。アドレスの変換はハードウェアではMMU (memory management unit) が担当することが多いが、エミュレータではCPU内部に実装した。

RISC-Vではアドレス変換の方法をSv32、Sv39、Sv48の3種類定義している。今回はSv39のみを実装した。Sv39では仮想メモリのアドレス幅が39ビットあるため、最大で`512GiB (=2**39)`のメモリを使用できる。

アドレスの変換はハードウェアのページテーブルウォークによって行われる。ページテーブルとは仮想アドレスと物理アドレスを対応づけるために使われるデータ構造で、いわゆる固定長要素の配列である。RISC-VのSv39ではページテーブルが3段存在し、このテーブルの要素を辿っていくことで物理アドレスに変換する。

### 2020年3月16日 ([d690e52: src/devices/plic.rs][0316])
周辺機器からの割り込みを制御するPLIC (platform-level interrupt controller) を実装した。

PLICもUARTと同じくメモリマップドデバイスの一つである。[QEMUの実装][qemu virt]を見ると、PLICは`0xc000000`の位置から`0x4000000`の大きさだけマップされていることがわかる。

PLICの主な機能は、どの周辺機器がどのハードウェアスレッドに対して割り込んだかを伝えることである。[xv6の割り込みの実装][xv6 plic claim]を見てみると、`plic_claim`の関数によって、どの周辺機器が割り込みを行ったかを示す値をPLICから得ている。[xv6のplicの実装][xv6 plic]の実装を見ると、`plic_claim`関数では`PLIC_SCLAIM`の位置にメモリマップドされた値を読んでいるだけだとわかる。よって、この時点ではまだ実装していないが、割り込みが発生したときにこのアドレスの位置の周辺機器を識別する値を書き込めばよい。

### 2020年3月18日 ([1e7964f: src/interrupt.rs][0318 interrupt])
割り込みを実装した。

今まで実装したUARTやPLICの周辺機器は、メモリマップドされたアドレスに存在する値を読み書きできるだけで、まだ割り込みをすることができていなかった。

[`src/interrupt.rs`][rvemu plic claim]で割り込みが発生ときにPLICの`sclaim`レジスタに周辺機器を識別する値を書き込んでいる。これにより、OS側の割り込みハンドラがこの値を読むことでどの周辺機器が割り込みを発生させたのか識別できる。

### 2020年3月18日 ([e535d27: src/devices/clint.rs][0318 clint])
タイマー割り込みを行うためのCLINT (core-local interruptor) を実装した。

CLINTもUARTやPLICと同じくメモリマップドデバイスの一つである。[QEMUの実装][qemu virt]を見ると、CLINTは`0x2000000`の位置から`0x10000`の大きさだけマップされていることがわかる。

[タイマーを初期化しているxv6の実装][xv6 timerinit]を見ると、インターバルタイムの1000000と`MTIME`を足した値を`CLINT_MTIMECMP`の位置に書き込んでいるのがわかる。CPUが何サイクル実行したかをカウントする`MTIME`レジスタの値がCPUこの値より大きくなったときに、タイマー割り込みが発生する。

### 2020年3月20日 ([c225992: src/devices/virtio.rs][0320])
ディスクやネットワークのインターフェースであるvirtioを実装した。ネットワークの方は実装しておらず、ディスクの読み書き部分である。

VirtioもUART、PLIC、CLINTと同じくメモリマップドデバイスの一つである。[QEMUの実装][qemu virt]を見ると、virtioは`0x10001000`の位置から`0x1000`の大きさだけマップされていることがわかる。

[virtioの仕様書][virtio spec]は読まずに、[xv6のvirtioのドライバの実装][xv6 virtio]内のディスクを読み書きする関数である[`virtio_disk_rw`][xv6 virtio_disk_rw]に対応するようにエミュレータを実装した。Virtioの実装時には[takahirox/riscv-rustの実装][riscv-rust/mmu.rs]も参考にさせていただいた。

これでxv6が動く。ここまでで実装した機能は、以下の通りである。しかし、浮動小数点数の丸めやアトミック命令の原子性はまだ実装できていないがxv6が動いているので、CPUの命令はここまで実装する必要が無かったのではないかと思う。
- RV64G命令
- 特権レベル
- 関連するステータスレジスタ
- 仮想メモリ機構
- UART
- CLINT
- PLIC
- Virtio

次の目標はLinuxを動かすことだが、まだわからないことだらけなので徐々に学んでいきたい。

[xv6-riscv]: https://github.com/mit-pdos/xv6-riscv
[rvemu]: https://github.com/d0iasm/rvemu
[1022]: https://github.com/d0iasm/rvemu/blob/143c7d58b909e9ed8c943dfdb48e98fea058bc04/src/lib.rs
[rust-and-webassembly]: https://rustwasm.github.io/docs/book/introduction.html
[wasm-bindgen]: https://github.com/rustwasm/wasm-bindgen
[1030]: https://github.com/d0iasm/rvemu/blob/2274ebf884136af0413727e3da46c2bbc7ae2e90/src/cpu/instruction.rs
[riscv.org/specifications]: https://riscv.org/specifications/
[1111]: https://github.com/d0iasm/rvemu/blob/2d59abc76a834886520b82304bde4fd7113fa2a9/src/cpu.rs
[1118]: https://github.com/d0iasm/rvemu/blob/caea7c67701d9dcbef878e89299efe17ba8a873a/src/cpu.rs
[1119]: https://github.com/d0iasm/rvemu/blob/44c05849e9994ac988273c1f4522cc6f99f77fdc/src/cpu.rs
[1230]: https://github.com/d0iasm/rvemu/blob/67558bd1e0663c1865f954f6ec2c5aabf309bd06/src/cpu.rs
[IEEE 754-2008]: https://ieeexplore.ieee.org/document/4610935
[IEEE wiki]: https://ja.wikipedia.org/wiki/IEEE_754
[0102]: https://github.com/d0iasm/rvemu/blob/83ef66c1bf0ea80240ca4e393a97ef1172efee08/src/cpu.rs
[0105]: https://github.com/d0iasm/rvemu/blob/533ea6910ba32d0223fc8e44fc03ede40d660f7f/src/cpu.rs
[cas]: https://en.wikipedia.org/wiki/Compare-and-swap
[ABA problem]: https://en.wikipedia.org/wiki/ABA_problem
[lrsc]: https://en.wikipedia.org/wiki/Load-link/store-conditional
[0214]: https://github.com/d0iasm/rvemu/blob/a751a6b572fe1d1ae342ae7d1af20fdcd1273045/src/cpu.rs
[0218]: https://github.com/d0iasm/rvemu/blob/ba64dfa24d2dcf090f1e37b13581945bc3eec78e/src/exception.rs
[0222]: https://github.com/d0iasm/rvemu/blob/6b6f7f9189f01c62e36d2686afb12f56c7ffc521/src/cpu.rs
[0304]: https://github.com/d0iasm/rvemu/blob/859d9fad1b3f2e1cb14eaa749d9e86758616637a/src/exception.rs
[0309]: https://github.com/d0iasm/rvemu/blob/7cc38ab2ea75465bce9f8dd709c29228d4d581cd/src/devices/uart.rs
[qemu virt]: https://github.com/qemu/qemu/blob/master/hw/riscv/virt.c#L54-L71
[xv6 uart]: https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c
[uart spec]: http://byterunner.com/16550.html
[0313]: https://github.com/d0iasm/rvemu/blob/ea7734110dc36146a8396c477bf415910a970f54/src/cpu.rs
[translate]: https://github.com/d0iasm/rvemu/blob/ea7734110dc36146a8396c477bf415910a970f54/src/cpu.rs#L218-L325
[0316]: https://github.com/d0iasm/rvemu/blob/d690e525bd943de08d9bbc6321b2402a7919a128/src/devices/plic.rs
[xv6 plic claim]: https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L185-L186
[xv6 plic]: https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/plic.c
[0318 interrupt]: https://github.com/d0iasm/rvemu/blob/1e7964fe0f390ac2efa0e49c4d46c0bc4d330735/src/interrupt.rs
[rvemu plic claim]: https://github.com/d0iasm/rvemu/blob/1e7964fe0f390ac2efa0e49c4d46c0bc4d330735/src/interrupt.rs#L40-L42
[0318 clint]: https://github.com/d0iasm/rvemu/blob/e535d2721290fd228e8e299038391fe82bb6633b/src/devices/clint.rs
[xv6 timerinit]: https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/start.c#L51-L82
[0320]: https://github.com/d0iasm/rvemu/blob/c225992ae1d34cd37ebad96542e84b3d2667461e/src/devices/virtio.rs
[xv6 virtio]: https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/virtio_disk.c
[xv6 virtio_disk_rw]: https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/virtio_disk.c#L168-L249
[virtio spec]: https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf
[riscv-rust/mmu.rs]: https://github.com/takahirox/riscv-rust/blob/master/src/mmu.rs
