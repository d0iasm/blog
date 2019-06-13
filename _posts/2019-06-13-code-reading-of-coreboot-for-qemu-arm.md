---
layout: post
title:  Code Reading of Coreboot Project for  QEMU/ARM
date:   2019-06-13 9:00 +0900
categories: coreboot
---

GSoC2019 の一環として参加している [coreboot][coreboot] プロジェクトのコードリーディングをしました。coreboot は BIOS/UEFI を置き換えることを目的としているオープンソースのプロジェクトです。私のプロジェクトは、coreboot に QEMU/AArch64 のサポートを加えることです。GSoC 公式サイトにおける説明は[こちら][gsoc]です。本記事では、AArch64 と一部実行フローが重なるであろう QEMU/ARM 上で実行する際の大まかな流れを追ってみました。

現在の coreboot では QEMU を使用したシミュレーションをサポートしており、対象のプラットフォームは x86、ARM、POWER8、RISC-V です。私のプロジェクトで加えることになる AArch64 は ARM アーキテクチャのうち ARMv8 以降の64bitの実行モードです。

QEMU/ARM でサポートしているマシンは vexpress-a9、CPU は cortex-a9 です。coreboot のルートディレクトリにおいて `qemu-system-arm -bios ./build/coreboot.rom -M vexpress-a9 -nographic` と実行した際に、Linux Kernel や GRUB2 などの任意のペイロードに処理を渡すまでを解説します。

## Architecture
coreboot は大きく分けて3つのステージから成り立ちます。Bootblock、Romstage、Ramstage と呼ばれるそれぞれのステージはコンパイル時には別のバイナリとして生成されます。そして最後に全てのバイナリをリンクし、ROM に書き込むためのファームウェア `coreboot.rom` を生成します。なぜ最終的には1つのバイナリにするのに、途中ではバラバラなバイナリとして生成するのかは、最初のステージ以外は LZMA 圧縮アルゴリズムを用いてサイズを縮小するためです。1つ前のステージが次のステージを解凍し、実行権を渡すことで進んでいきます。

各ステージの役割は、  
**Bootblock**
1. ヒープとスタックを使用するためのCache-As-RAM: DRAM の初期化を行う以前は CPU Cache を記憶領域として使用します。これにより、初期の頃から高級言語でコードを書くことが可能です。以前は Cache の代わりに全てローカル変数をレジスタに格納することによって、この問題を解決していました。ROMCC という特殊なコンパイラによって、ローカル変数をレジスタの値に置き換えていたようです。
2. スタックポインタの設定
3. BSS 用のメモリをクリア
4. 次の Romstage を解凍し、メイン関数を呼び出す

TODO: Cache-As-RAM、スタックポインタの設定、BSS 用のメモリクリアについてはアセンブリで書かれているようで、まだコードを読めていないです。もしかしたら、x86 特有の処理だったりもするかも。

**Romstage**
1. コンソールの初期化
2. 次の Ramstage を解凍し、メイン関数を呼び出す

**Ramstage**
1. プログラミング言語 Ada の初期化: coreboot 内で使用できるが、現在はもうあまり使われていないそうです。
2. コンソールの初期化: コンソールに出力する以前には必ず初期化する必要があります。しかし、Romstage でもコンソールの初期化をしていたので、なぜ同じ初期化コードを呼び出す必要があるのかは要確認。
3. : コンソールに出力する以前には必ず初期化する必要があります。しかし、Romstage でもコンソールの初期化をしていたので、なぜ同じ初期化コードを呼び出す必要があるのかは要確認。
4. cbmem の初期化: coreboot 特有の出力方法。
5. 例外処理の初期化
6. スレッドの初期化
7. デバイスの初期化
  * チップの初期化
  * 接続されている周辺機器の確認
  * 周辺機器を繋ぐバスの設定
  * バスを有効化
  * 周辺機器初期化
8. コンパイル時にユーザーが設定できる任意のペイロード (ELF、GRUB2、Kernel 等) を解凍し、呼び出す

以下はそれぞれのステージが UEFI のステージとどのように対応しているかの図です。

![coreboot architecture](/assets/comparision_coreboot_uefi.svg)
[coreboot architecture][coreboot arch] より引用


## Code Reading for Each Stage








{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

[coreboot]: https://www.coreboot.org/
[gsoc]: https://summerofcode.withgoogle.com/projects/#5148970366533632
[coreboot arch]: https://doc.coreboot.org/getting_started/architecture.html
