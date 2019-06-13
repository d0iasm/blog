---
layout: post
title:  Code Reading of Coreboot Project for  QEMU/ARM
date:   2019-06-13 9:00 +0900
categories: coreboot
---

GSoC2019 の一環として参加している [coreboot][coreboot] プロジェクトのコードリーディングをしました。coreboot は BIOS/UEFI を置き換えることを目的としているオープンソースのプロジェクトです。私のプロジェクトは、coreboot に QEMU/AArch64 のサポートを加えることです。GSoC 公式サイトにおける説明は[こちら][gsoc]です。本記事では、AArch64 と一部実行フローが重なるであろう QEMU/ARM 上で実行する際の大まかな流れを追ってみました。

現在の coreboot では QEMU を使用したシミュレーションをサポートしており、対象のプラットフォームは x86、ARM、POWER8、RISC-V です。私のプロジェクトで加えることになる AArch64 は ARM アーキテクチャのうち ARMv8 以降の64bitの実行モードです。

本記事では、coreboot のルートディレクトリにて   
`$ qemu-system-arm -bios ./build/coreboot.rom -M vexpress-a9 -nographic`  
と実行した際に、Linux Kernel や GRUB2 などの任意のペイロードに処理を渡すまでを解説します。

<br />
## Architecture
coreboot は Bootblock、Romstage、Ramstage と呼ばれる3つのステージから成り立ちます。それぞれのステージはコンパイル時には別のバイナリとして生成されます。そして最後に全てのバイナリをリンクし、ROM に書き込むためのファームウェア `coreboot.rom` を生成します。なぜ最終的には1つのバイナリにするのに、途中ではバラバラなバイナリとして生成するのかは、最初のステージ以外は LZMA 圧縮アルゴリズムを用いてサイズを縮小するためです。RAM よりも容量の制限が厳しい ROM では、圧縮によってサイズが小さくすることが利点となります。現在のステージが次のステージを解凍し、呼び出すことで進んでいきます。

各ステージの役割は、  
**Bootblock**
1. ヒープとスタックを使用するためのCache-As-RAM: DRAM の初期化を行う以前は CPU Cache を記憶領域として使用します。これにより、初期の頃から高級言語でコードを書くことが可能です。 *1
2. スタックポインタの設定
3. BSS 用のメモリをクリア
4. 次の Romstage を解凍し、メイン関数を呼び出す

*1: 以前は Cache の代わりに全てローカル変数をレジスタに格納することによって、この問題を解決していました。ROMCC という特殊なコンパイラによって、ローカル変数をレジスタの値に置き換えていたようですが、ROMCC の開発が1人のエンジニアに依存しており、x86 以外の他のアーキテクチャのサポートが難しかったため、Cache-As-RAM に変更をしたそうです。

TODO: Cache-As-RAM、スタックポインタの設定、BSS 用のメモリクリアについてはアセンブリで書かれているようで、まだコードを読めていないです。もしかしたら、x86 特有の処理だったりもするかも。

**Romstage**
1. コンソールの初期化
2. 次の Ramstage を解凍し、メイン関数を呼び出す

**Ramstage**
1. プログラミング言語 Ada の初期化: coreboot 内で使用できるが、現在はもうあまり使われていないそうです。
2. コンソールの初期化: コンソールに出力する以前には必ず初期化する必要があります。しかし、Romstage でもコンソールの初期化をしていたので、なぜ同じ初期化コードを呼び出す必要があるのかは要確認。
3. cbmem の初期化: coreboot 特有の出力方法。
4. 例外処理の初期化
5. スレッドの初期化
6. デバイスの初期化
  * チップの初期化
  * 接続されている周辺機器の確認
  * 周辺機器を繋ぐバスの設定
  * バスを有効化
  * 周辺機器初期化
7. コンパイル時にユーザーが設定できる任意のペイロード (ELF、GRUB2、Kernel 等) を解凍し、呼び出す

以下はそれぞれのステージが UEFI のステージとどのように対応しているかの図です。

![coreboot architecture](/assets/comparision_coreboot_uefi.svg)
[coreboot architecture][coreboot arch] より引用

<br />
## Code Reading
C言語で書かれた Bootblock -> Romstage -> Ramstage -> Payload の一連の処理についてコードを読みました。各ステージにおけるエントリポイントは以下の通りです。
* Bootblock: main() - [src/lib/bootblock.c][bootblock]
* Romstage: main() - [src/mainboard/emulation/qemu-armv7/romstage.c][romstage]
* Ramstage: main() - [src/lib/hardwaremain.c][ramstage]

各ステージが共通で行う処理は、
1. 現在のステージで必要な初期化処理等を行う
2. 次のステージに関する情報を `prog` 構造体によって管理する

この2つのステップを各ステージにて繰り返すことによって、処理を進めています。

ステージ実体は以下の `prog` 構造体によって管理されます。現在のステージが次のステージの `prog` 構造体の初期化をします。 `entry` には関数ポインタが格納され、現ステージでの処理が終わったらこれを呼ぶことよって、次のステージに移動します。

{% highlight c %}
// src/include/program_loading.h

/* Representation of a program. */
struct prog {
  /* The region_device is the source of program content to load. After
   * loading program it represents the memory region of the stages and
   * payload. For architectures that use a bounce buffer
   * then it would represent the bounce buffer. */
   enum prog_type type;
   uint32_t cbfs_type;
   const char *name;
   struct region_device rdev;
   /* Entry to program with optional argument. It's up to the architecture
    * to decide if argument is passed. */
    void (*entry)(void *);
    void *arg;
};
{% endhighlight %}

<br />
Ramstage では `boot_state` 構造体の配列によって必要な処理を管理し、順に実行していきます。`id` に現在の状態、`run_state` に処理の関数ポインタを格納します。そして、処理が終わったら `complete` フラグを立て、次の処理に移動します。

{% highlight c %}
// src/lib/hardwaremain.c

struct boot_state {
  const char *name;
  boot_state_t id;
  u8 post_code;
  struct boot_phase phases[2];
  boot_state_t (*run_state)(void *arg);
  void *arg;
  int complete : 1;
#if CONFIG(HAVE_MONOTONIC_TIMER)
  struct boot_state_times times;
#endif
  };
{% endhighlight %}

必要な処理は [boot_states][boot_states] 配列として予め初期化されており、[bs_walk_state_machine][bs_walk_state_machine] 関数の中の while 文によって、各処理が実行されます。

デバイスの検出、デバイスに必要な資源の獲得と有効化等をした後に、Payload を呼び出して coreboot の仕事は終了です。以下が具体的に行われる各処理の説明です。
{% highlight c %}
// src/include/bootstate.h

* BS_PRE_DEVICE - before any device tree actions take place
* BS_DEV_INIT_CHIPS - init all chips in device tree
* BS_DEV_ENUMERATE - device tree probing
* BS_DEV_RESOURCES - device tree resource allocation and assignment
* BS_DEV_ENABLE - device tree enabling/disabling of devices
* BS_DEV_INIT - device tree device initialization
* BS_POST_DEVICE - all device tree actions performed
* BS_OS_RESUME_CHECK - check for OS resume
* BS_OS_RESUME - resume to OS
* BS_WRITE_TABLES - write coreboot tables
* BS_PAYLOAD_LOAD - Load payload into memory
* BS_PAYLOAD_BOOT - Boot to payload
{% endhighlight %}

<br />
## Conclusion
全体として、アーキテクチャ依存のコードが多くなりすぎず、うまく構造化して綺麗にまとまっているなという感想を持ちました。しかしまだC言語レベルでのざっくりとした理解しかできていないので、アセンブリでの処理も引き続き読み進めていきたいです。

感想、疑問等あればお気軽に[@d0iasm][d0iasm]まで。


GitHub へのリンク先はあくまでも2019年06月時点で正しい位置なので、今後変更する可能性があるのでご注意ください。


[coreboot]: https://www.coreboot.org/
[gsoc]: https://summerofcode.withgoogle.com/projects/#5148970366533632
[coreboot arch]: https://doc.coreboot.org/getting_started/architecture.html
[bootblock]: https://github.com/coreboot/coreboot/blob/master/src/lib/bootblock.c#L62
[romstage]: https://github.com/coreboot/coreboot/blob/master/src/mainboard/emulation/qemu-armv7/romstage.c#L19
[ramstage]: https://github.com/coreboot/coreboot/blob/master/src/lib/hardwaremain.c#L434
[boot_states]: https://github.com/coreboot/coreboot/blob/master/src/lib/hardwaremain.c#L101
[bs_walk_state_machine]: https://github.com/coreboot/coreboot/blob/master/src/lib/hardwaremain.c#L326
[d0iasm]: https://twitter.com/d0iasm
