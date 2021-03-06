# ［C++］volatile修飾子についての考察

※この記事は[C++20を相談しながら調べる会 #3](https://cpp20survey.connpass.com/event/147002/)の成果として書かれました。

[P1152R4 : Deprecating volatile](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1152r4.html)を読み解く過程に生じた`volatile`についての調査（脱線）メモです。ほぼ文章です。

[:contents]

### C++におけるvolatileの効果

C++における`volatile`指定の効果は次のように規定されています（[6.9.1 Sequential execution [intro.execution]](http://eel.is/c++draft/basic.exec#intro.execution-7)）。
>Reading an object designated by a volatile glvalue ([basic.lval]), modifying an object, calling a library I/O function, or calling a function that does any of those operations are all side effects, which are changes in the state of the execution environment.

`volatile`オブジェクト（メモリ領域）へのアクセス（読み/書き）、およびそれを実行する関数呼び出しは全て副作用（*side effects*）である！と言っています。

そしてそのあとに以下のように書いてあります。

>Every value computation and side effect associated with a full-expression is sequenced before every value computation and side effect associated with the next full-expression to be evaluated.

完全式（*full-expression*）の全ての値の計算および副作用（*side effects*）は、次の完全式の全ての値の計算および副作用の評価の前に位置付けられる（*sequenced before*）、と言っています。  
ここでの完全式（*full-expression*）とは、セミコロン`;`で終わる一つの式のことです。変な書き方をしていなければ、通常それは1行で書かれていると思います。

ついでに*sequenced before*と言うのも見てみると

>Sequenced before is an asymmetric, transitive, pair-wise relation between evaluations executed by a single thread ([intro.multithread]), which induces a partial order among those evaluations.  
>Given any two evaluations A and B, if A is sequenced before B (or, equivalently, B is sequenced after A), then the execution of A shall precede the execution of B.

*sequenced before*とは、__シングルスレッド実行においての__ 評価間の関係のことで、評価Aが評価Bよりも前に位置づけられる（*sequenced before*）場合、Aの実行はBの実行より前に行われる、と書かれています。シングルスレッドです。

もう一つ、C++規格がその動作の対象としている抽象機械の[観測可能な動作（*observable behavior*）](http://eel.is/c++draft/intro#abstract-6.1)のところに次のようにあります
>Accesses through volatile glvalues are evaluated strictly according to the rules of the abstract machine.  
>〜中略〜  
>These collectively are referred to as the observable behavior of the program. 

`volatile`オブジェクトへのアクセスは抽象機械の規則に従って厳密に評価される。〜中略〜 これらは、__プログラムの観測可能な動作（*observable behavior*）__ と呼ばれる、とあります。

そして、[4.1.1 Abstract machine [intro.abstract]/5](http://eel.is/c++draft/intro#abstract-5)、[4.1.1 Abstract machine [intro.abstract]/1](http://eel.is/c++draft/intro#footnote-4)にある様に、C++の適合実装（コンパイラ及び実行環境）はこの抽象機械上プログラムの __観測可能な動作__ をエミュレートするように実装され、それ以外の事をエミュレートする必要はありません。  
非`volatile`な変数へのアクセスはこの観測可能な動作に含まれていないので、最終的なプログラムの出力に影響を与えない範囲において、それらのアクセス・計算は削除されたり順序が変わったりします。これこそがコンパイラに許可されている最適化の範囲です（この事を「as-ifルール」と呼びます）。

`volatile`オブジェクトへのアクセスおよびその先で起こることは全て副作用（*side effects*）になりますが、観測可能な動作（*observable behavior*）はそのうちの`volatile`オブジェクトへのアクセスそのものしか含んでいません。  
従って、コンパイラが気にするのは`volatile`オブジェクトへアクセスしたかどうかだけであり、その先で起こる事については感知しなくてもいいわけです。  
またこのことから「観測可能な動作⊂副作用」と言う関係が成り立ち、副作用は必ずしも最適化対象外にあるわけではないこともわかります。

これらの条文から、C++が規定する`volatile`指定の効果とは

- シングルスレッド実行上において
- `volatile`オブジェクトへのアクセス（読み/書き）の順序を保証し
- `volatile`オブジェクトへのアクセスに関してはコンパイラの最適化対象外となる

と言う事だとわかります。

この事は次の様な意味でもあります。

- `volatile`オブジェクトへのアクセス（読み/書き）の順序を保証し
  - C++コード上で書いた通りの順序になる事を保証
- `volatile`オブジェクトへのアクセスに関してはコンパイラの最適化対象外となる
  - `volatile`オブジェクト（の関与する部分）は最適化されない、と言うことと等価


### マルチスレッドとvolatile

マルチスレッド時に同期のために用いられる変数には以下3つの性質が必要です。

- 原子性（*atomicity*）
  - データの読み書きが分割不可能な1操作で行える事
  - データを読み出した際に別のスレッドが更新途中の中途半端な値を読み出さない事が保証される
- 可視性（*visibility*）
  - あるスレッドにて変数に書き込まれた結果が、別のスレッドから（いつかは）観測できる事
- 順序性（*ordering*）
  - あるスレッドから見た別スレッド上のメモリアクセスが同一の順序であるように観測できる事

`volatile`の保証する効果は上記のようにシングルスレッドにおいてのものであり、マルチスレッドにおいてはなんら記述がないのでこの3つの性質の全てを満たしません。`volatile`変数をスレッド間同期等に利用すると常に未定義動作の世界にこんにちはします。

原子性（*atomicity*）や可視性（*visibility*）に関してはなんら関わっていない事が直ぐにわかるでしょう。  
順序性（*ordering*）だけは、概念的に少し重複する部分があるので保証されるように見えてしまうかもしれませんが、`volatile`が保証するアクセス順序はシングルスレッド上の`volatile`変数間のものだけです。やはりマルチスレッドでは何の意味も持ちません。

マルチスレッドでのスレッド間同期には`std::atomic`と適切なメモリバリアを利用しましょう。`volatile`は無力です。

ただ、他のプログラミング言語の中には`volatile`指定の効果に、C++の`volatile`の効果に加えて`std::atomic`の持つ一部の性質がくっついている事があります。例えばC#やJavaがそうですが、このような言語においてはその規則の範囲内において期待する効果が得られるかもしれません。  
これが`volatile`の勘違いの原因の一つでもある気がしますが・・・・

### 用途と副次的効果

ではこの`volatile`オブジェクトへのアクセス、というのはいつ有効に活用できるのでしょうか？  
おそらく一番のユースケースは、メモリマップドI/O方式を採用している環境においてI/Oアクセスを行う時に使用する変数に付加することです。  
メモリマップドI/O方式というのは、同一のアドレス空間上にメインメモリのアドレスと外部I/Oデバイスのポートやレジスタ、メモリ等のアドレスをマッピングする事でメインメモリアクセスと同じようにI/Oアクセスできるようにする仕組みのことです。

```cpp
#include <cstdint>

#define IO_ADDR 0xffffff;  //どこかのI/Oポートを指すアドレス

int main() {
  volatile std::unit16_t* io_ptr = IO_ADDR;

  *io_ptr = 0x01;       //I/Oに書き込み
  auto data = *io_ptr;  //I/Oから読み込み
  *io_ptr = 0x80;       //I/Oに書き込み
}
```

例えばこのような場合、`IO_ADDR`のアドレスの指す先は外部デバイスのレジスタであり、その領域へのアクセスはI/Oという意味を持ちます（例えばLチカだったり、センサからのデータ取得だったり・・・）。  
しかし上記プログラムはコンパイラから見ると何の意味もない処理でしかなく、最適化の結果`io_ptr`へのアクセスは消去可能です。

このようなI/Oアクセスはこのコード上からは観測不可能な外部デバイスで何かしらの意味を持ち、アクセス及びその順序にもデバイス毎に違った意味がある筈です。  
そのため、そのアクセスを消されたり順序を変えられてしまうと意図通りの結果を得ることができなくなるので、`volatile`の出番というわけです。

この様な意味から考えると、`volatile`なオブジェクト（`volatile`と指定されたメモリ領域）へのアクセスは、CPU内部のキャッシュ機構を通り越してその領域へ直接読み書きする形になっているはずです。そうしないとこのような目的を達せないためです。  
それはすなわち、`volatile`な領域への書き込みはすぐにそのメモリ領域に書き込まれ、読み込みは常にそのメモリ領域から最新のデータを読みだしてくることになります。

この副次的な効果として、マルチスレッドから`volatile`オブジェクトを通してメモリ領域を読み書きした場合、CPU内部のキャッシュが介在することによって生じるデータ同期の遅延が生じなくなります。  
また、`volatile`の効果によってシングルスレッド上では少なくとも`volatile`領域への読み書きの順序が入れ替わることはないため、結果としてマルチスレッド間でアクセス順序が意図通りに行われているように見える事もあるでしょう。  
そして、CPUのデータバス幅未満のサイズの変数でアライメント要求などを満たしていれば、その読み書きはatomicに行われうるため、CPUと型のサイズの組み合わせによっては変な値を読みだしてしまう事が起きないかもしれません。

すなわち、マルチスレッド間の`volatile`指定した変数アクセスが原子性、可視性および順序性を満たしているかのように見えることがあるかもしれません。  
無論これは保証されるようになったわけではなく、未定義動作の範疇としてたまたまそうなっているだけです。完全に砂上の楼閣であり、1つでも条件が変われば地獄に落ちるかもしれません。  
結局、`volatile`指定をマルチスレッド処理の同期用などに用いることは間違っています、`std::atomic`と適切なメモリバリアを利用しましょう。

この様な事が起こりうるのも`volatile`の効果についての混乱の原因の一つであるのでしょう・・・

### アウトオブオーダー実行とvolatile

`volatile`オブジェクトへのアクセスはコンパイラの最適化の対象とならず、その順序が保たれたコードが出力されることが保証される、という事は分かりました。  
しかし、このようなメモリ領域へのアクセス順はもう一つの所で変更される可能性があります。それは、CPUにおけるアウトオブオーダー実行時です。

アウトオブオーダーにおいては、最終的な結果が変わらない範囲において命令順を並べ替えて効率的に実行しようとします。この時、計算の順序だけでなく、メモリアクセスの順番も変化しえます。

C++抽象機械の観測可能な動作を厳密にエミュレートするのであれば、アウトオブオーダー実行においてもその順序を維持するという事は変わらないように思えます。実際、それはメモリバリア命令を挿入するなどによって実現可能です。

しかしどうやら、C++標準はこれを保証しないようです。

C++抽象機械としての`volatile`オブジェクトへのアクセスは観測可能な動作（*observable behavior*）として実装は厳密にエミュレートしなければならず、結果的にC++標準が定義する*sequenced before*関係が保たれたプログラムが出力されます。  
しかし、観測可能な動作（*observable behavior*）とはあくまで抽象機械上のことであり、実際の観測可能な順序（*observable ordering*）に関してを何ら規定していません。

従って、C++標準はアウトオブオーダー実行によって実行時に命令が並べ替えられて`volatile`アクセスの順序が変更されないことを保証していません。  
もやもやしますが、これがC++の現在の見解のようです・・・・

この様な場合に命令順序を入れ替えられたくない場合、適切なメモリバリア命令を明示的に利用するか、`std::atomic`を利用することができます。  
デフォルトの`std::atomic`変数の各種オペレーションはマルチスレッドにおける逐次一貫性（*sequential consistency*）を保証します。  
このため、シングルスレッドにおいてもメモリバリアがその読み書きにおいて適切に利用されるため、アウトオブオーダー実行でも順序関係が維持されるようになります。


### 参考文献
- [6.9 Program execution [basic.exec] - Working Draft, Standard for Programming Language C++ (N4830)](http://eel.is/c++draft/basic.exec)
- [How are side effects and observable behavior related in C++? - stackoverflow](https://stackoverflow.com/questions/13271469/how-are-side-effects-and-observable-behavior-related-in-c)
- [volatileが必要な場面を見つけ出す - teratail](https://teratail.com/questions/114172)
- [ミューテックスとアトミック処理について - teratail](https://teratail.com/questions/165667)
- [volatile版atomic操作関数が存在する理由 - yohhoyの日記](https://yohhoy.hatenadiary.jp/entries/2012/07/01)
- [volatile変数とマルチスレッドとの関係についての押し問答（前編） - yohhoyの日記](https://yohhoy.hatenadiary.jp/entry/20121016/p1)
- [volatile変数とマルチスレッドとの関係についての押し問答（中編） - yohhoyの日記](https://yohhoy.hatenadiary.jp/entry/20131009/p1)
- [volatile変数とマルチスレッドとの関係についての押し問答（後編） - yohhoyの日記](https://yohhoy.hatenadiary.jp/entry/20140808/p1)
- [POS03-C. volatile を同期用プリミティブとして使用しない - JPCERT CC](http://www.jpcert.or.jp/sc-rules/c-pos03-c.html)
- [デバイスにアクセスするには | 学校では教えてくれないこと - uQuest](https://www.uquest.co.jp/embedded/learning/lecture13.html)
- [Android のための SMP 入門](http://milkpot.sakura.ne.jp/note/smp-primer.html)
- [P1152R0 : Deprecating volatile](https://wg21.link/p1152r0)
- [Does the C++ volatile keyword introduce a memory fence? - stackoverflow](https://stackoverflow.com/questions/26307071/does-the-c-volatile-keyword-introduce-a-memory-fence)

### 謝辞
この記事の9割は以下の方々によるご指摘によって成り立っています。

- [@yohhoyさん](https://twitter.com/yohhoy/status/1181102762546712578)
- [@yohhoyさん](https://twitter.com/yohhoy/status/1181104668413267973)
- [@yohhoyさん](https://twitter.com/yohhoy/status/1181101292296343552)
- [@yohhoyさん](https://twitter.com/yohhoy/status/1181487680648957953)
- [@yohhoyさん](https://twitter.com/yohhoy/status/1181489122600374272)
- [@yohhoyさん](https://twitter.com/yohhoy/status/1182264372736880640)

[この記事のMarkdownソース](https://github.com/onihusube/blog/blob/master/2019/20191009_study_of_volatile.md)