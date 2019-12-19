---
layout: post
title:  Fibonacci Numbers on My RISC-V Emulator
date:   2019-12-19 9:00 +0900
categories: risc-v
---

この記事は[自作OS Advent Calendar 2019](https://adventar.org/calendars/4027)の19日目の記事です。本記事では、現在開発中のRISC-Vエミュレータの上で、フィボナッチ数列を計算するバイナリを動かし、それに関するRISC-Vテクノロジについて書きます。

## Introduction
現在、RISC-Vのエミュレータを趣味で開発しています。メインのコードをRustで書き、WebAssemblyにコンパイルすることによって、WEB上で動かすことができます。

最終目標はOSをエミュレータ上で動かすことですが、現時点ではまだ50を超える命令セットが最もプリミティブなモードであるマシンモード (M mode) で動くのみです。まだ目標達成までやることはかなり残っていますが、それでもプログラムっぽいものが動くようになってきています。以下がフィボナッチ数列を自作エミュレータ上で実行してみたデモです。

![Demo fib on rvemu]({{ site.url }}/blog/assets/2019-12-19-fib-rvemu.gif)

現在のエミュレータは以下の4つのコマンドによって操作することができます。
* upload: RISC-Vバイナリをローカルのコンピュータからアップロードする
* ls: アップロードされたバイナリの一覧を表示する
* run [file]: アップロードされたバイナリを実行する
* help: コマンドの使い方を表示する

デモ動画ではあらかじめ準備しておいたフィボナッチ数を計算するバイナリファイル `fib` と他のテスト用バイナリフィアルをアップロードし、`run fib`によって実行しています。実行後は32個あるレジスタがどのような値になっているのかを表示します。今回は10番目のフィボナッチ数を計算しており、`x10`の行を見ると`55`になっているため、無事に計算できていそうなことがわかります。

## RISC-V Overview
RISC-Vは誰でも無料で使用することができ、RISCの名が示す通り、命令セットの複雑さ、数、種類を減らしできるだけ単純化する設計思想の元に作られている比較的新しい命令セット (ISA) です。

RISC-VのISAは複数のモジュールによって成り立っています。最低限実装されている必要がある基本のモジュールはあるものの、その他は必要な機能だけを実装することが可能です。つまり、低コスト・低パフォーマンスで良い機器であれば最低限のモジュールだけを実装し、逆に何よりもパフォーマンスを重視している場合はSIMD命令等のモジュールも実装することができます。

本エミュレータはOSを動かすことが目標のため、基本的なモジュールだけではなく、仕様書に定義されている`RV32G`または`RV64G`というモジュールセットを実装する必要があると考えています。`RV32G`と`RV64G`はそれぞれ32/64 bit アーキテクチャのための汎用ISA ("general-purpose" ISA) と言われており、以下の全てのモジュールを含みます。
* RV32I/RV64I: 足し算、ブランチ、ロード、ストアなど最も基本的なモジュール。
* RV32M/RV64M: 掛け算、割り算のモジュール。
* RV32A/RV64A: アトミック命令のモジュール。
* RV32F/RV64F: 不動小数点数 (Float) のモジュール。
* RV32D/RV64D: 不動小数点数 (Double) のモジュール。
* Zicsr: コントロールレジスタを操作するのためのモジュール。
* Zifencei: フェンス命令 (メモリの書き込み順序を保証するための命令) のモジュール。

現在はこのうちの RV32I/RV64I/RV32M/RV64M を実装済みです。

仕様書は主にISAやメモリモデルについて定義しているUnprivileged Specificationとスーパバイザなどの特権モードについて定義しているPrivileged Specificationの2種類があります。どちらも[RISC-V Foundationのサイト](https://riscv.org/specifications/)から見ることができます。今回の実装に関しては、全てUnprevileged Specification (*The RISC-V Instruction Set ManualVolume I: Unprivileged ISADocument Version 20191213*) を参考にしました。


## RISC-V Assembly for Fibnacci Numbers
今回動かすプログラムはC言語で書いたフィボナッチ数列を求めるプログラムを、RISC-Vツールチェーンを使用しディスアセンブリしたものを手で簡略化しました。フィボナッチ数列を求めるときには再帰関数を使う手法もありますが、スタックがまだうまく動いていないようだったので、変数を全てレジスタに置き換えています。また、入出力等もまだ実装していないため、今回のプログラムでは決め打ちで10番目のフィボナッチ数 (55) を求めます。

``` c
int main() {
  int n=10;
  if(n <= 1) {
    return n;
  }
  int fib = 1;
  int fibPrev = 1;
  for(int i = 2; i < n; i++) {
    int temp = fib;
    fib += fibPrev;
    fibPrev = temp;
  }
  return fib;
}
```

```asm
main:
  li t3,10
  li t4,1
  blt t4,t3,label1	# if (1 < n) goto label1;
  j label3
label1:
  li t4,1		# int fib=1;
  li t5,1 		# int fibPrev=1;
  li t0,2		# int i=2;
label2:
  bge t0,t3,label3 	# if (i<n)
  mv t6,t4		# int temp=fib;
  add t4,t4,t5		# fib+=fibPrev;
  mv t5,t6 		# fibPrev=temp;
  addi t0,t0,1		# i++
  j label2
label3:
  mv a0,t4
```

### Assembler Mnemonics for Registers
上のアセンブリでまず気になるのが、レジスタの名前です。RISC-Vは`x0`から`x31`までの32個の汎用レジスタを持っています。しかし、上のアセンブリでは`t3`や`a0`などとなっています。これはアセンブラのニーモニックです。汎用レジスタと言いながらもそれぞれ役割を持っており、名前によって役割がわかるようになっています。

以下の表は`x0`から`x31`までのレジスタに名付けられたニーモニックです。今回使用したのは、変数を保存するための一時的な`t0`、`t3`から`t6`と、変数の返り値として`a0`を使用しました。`zero (x0)`は例外的に常に0が入っており、このレジスタの値を書き換えようとしても何も変化しません。
![Assembler mnemonics]({{ site.url }}/blog/assets/2019-12-19-assembler-mnemonics.png)
*The RISC-V Instruction Set Manual Volume I: Unprivileged ISA Document Version 20191213 (Chapter 26 RISC-V Assembly Programmer’s Handbook / Table 26.1)*


### Pseudoinstructions
次にアセンブリ自体を見ていきます。実は`li`、`j`、`mv`などのニーモニックはRISC-VのISAには存在しません。これらはアセンブリ言語の擬似コードで、実際に実行する前には変換されます。

具体的には、以下のように変換されます。
* `li t3, 10` ==> `addi x28, x0, 10` など
* `j label3` ==> `jal x0, label3`
* `mv t6, t4` ==> `addi x31, x29, 0`

このようにアセンブリの擬似コードを使用することによって、読みやすさを確保しつつも、実際のISAの数は減らして簡単な仕様を保っているんですね。

## Future Work
今後の予定としては、まず実装していないISAを実装すること、そしてスーパーバイザのモードを追加すること、そして、コードをクラウドサーバにアップロードして誰でもアクセスできる状態にすることなどを考えています。

エミュレータを作成しはじめたモチベーションの1つとして、ISAの仕様書を全て読み通してみたい気持ちがありました。以前OS自作界で有名な緑の本を読みつつコードを書いていたとき、どの部分が仕様などで決まっていて自分で勝手に決められないのか、どの部分なら自分で好き勝手に決めてよいのかわかりませんでした。なので、仕様書を読むことで、ISA (CPU) によって決められている範囲はどこからどこまでかを知ることができると考えたからです。
現在はまだ Unprivileged Specification の4分の1程度しか読めていないので、他の仕様書も含めて読み通すことも今後の目標です。

コードは全て[d0iasm/rvemu](https://github.com/d0iasm/rvemu)のリポジトリで公開しているので、興味があれば見てみてください。

