+++
date = "2014-08-25T14:49:44+09:00"
draft = false
title = "定数 (constants)"
+++

# 定数
[Constants](https://blog.golang.org/constants) by Rob Pike

## はじめに
Goは、数値型を混合して操作することを許さない静的型付け言語で。 `flaot64` を `int` に
足せませんし、さらに言えば `int32` を `int` を足すこともできません。しかし、 `1e6*time.Second` や
`math.Exp(1)` あるいは `1<<('\t'+2.0)` と書くことは許されています。Goでは、定数は変数と違って、
通常の数字と同様に振る舞います。この記事では、なぜそうなっているのか、そしてそれが何を意味するのかを
説明します。

## 背景: C言語

Go言語を考え始めた初期の頃、C言語やその系譜の言語が数値型をまぜこぜに使うことを許していることで起きている多くの問題について話しました。
多くの奇妙なバグ、クラッシュ、移植可能性の問題は、サイズと「符号有無」が異なる整数を混ぜている式によって起きています。
軽々の多いCプログラマにとっても、次の計算結果はわかるかもしれませんが、アプリオリには明らかではありません。

```
unsigned int u = 1e9;
long signed int i = -1;
... i + u ...
```

結果はどれくらいの大きさになるのでしょう。その値はなんでしょうか。符号は付いているのかいないのか。

見難いバグがここに潜んでいます。

C言語には「the usual arithmetic conversions」と呼ばれる規則があり、年月を経てその規則が変わってきたことがこの規則の儚さを示しています。
（これから昔から知られているもっと多くのバグを紹介します）

Goを設計するときに、数値型を混ぜてはいけないということをしっかりと決めることでこの地雷原を避ける事にしました。
`i` と `u` を足したい場合には、結果をどうしたいかを明示しなければいけません。次のように変数があったとして、

```
var u uint
var i int
```

`uint(i)+u` または `i+int(u)` のどちらかに書けます。これはどちらの書き方でも足し算の意味と型がはっきりと表現されていますが、
C言語の場合とは異なって `i+u` と書くことはできません。 `int` が32ビット型の場合でさえ、 `int` と `int32` を混ぜることすらできません。

この厳格さによって、よくありがちなバグや他の問題が取り除かれます。これはGoにおいて非常に重要な性質です。
しかしこの厳格さにはコストが掛かります。意味を自明にするために、ときにこの厳格さがプログラマに不格好な
型変換のコードを追加することを求めます。

では定数ではどうでしょうか。これまでに述べてきたことからすると、どうすれば規則を守ったまま `i = 0` または `u = 0` と書けるでしょうか。
`0` の型は何でしょうか。単純な状況、たとえば `i = int(0)` ような場面で、型変換を書かなければいけないのは筋がよくありません。

私たちはすぐに数値の定数を扱う場合には、他のC言語の類似言語での動作とは異なる振る舞いをさせれば良いと気が付きました。
多くの考察と実験の後に、ほぼ常に正しいと信じられ、つねにプログラマを定数の型変換から解放し、それでいてコンパイラに
警告されることなく `math.Sqrt(2)` のように書ける設計を思いつきました。

手短に言えば、Goの定数は、とにかくほとんど場合、うまく動きます。ではそれがどのようになっているか見てみましょう。

## 用語解説

まず、手短に定義します。Goでは `const` は `2` 、 `3.14159` 、 `"scrumptious"` といったスカラー値に対する名前を決める
キーワードです。このような値は、名前が付いているかにかぎらず、Goでは _定数_ と呼ばれます。定数は定数からなる式からも
生成することが出来ます。たとえば `2+3` 、 `2+3i` 、 `math.Pi/2` あるいは `("go"+"pher")` といったものがそれです。

定数がない言語もありますし、 `const` という単語に対してより汎用的な定義がある言語やより汎用的な用途がある言語もあります。
たとえばC言語はC++では、 `const` はより複雑な値のより複雑な性質を分類する型修飾子です。

しかしGoでは、定数はとても単純で、変化しない値のことを指します。以降はGoでの場合のみを話します。

## 文字列定数

数値の定数には多くの種類があります。たとえば整数、浮動小数、ルーン（訳注：Unicodeのコードポイントを表す整数値）、符号あり、符号なし、虚数、複素数などがあります。
まずはより単純な形式の定数である文字列定数から話を始めましょう。文字列定数は理解がしやすく、Goにおける定数の型の問題を調べるときにより小さな部分だけ考えればよくなります。

文字列定数はある文字列をダブルクォーテーションで囲ったものです。（Goにはraw文字列リテラルもあり、これはバッククォートで囲みます。ここの議論においては
両者ともに同様の性質があるため省きます。）ここに文字列定数があります。

```
"Hello, 世界"
```

（文字列の表現と解釈に関してより詳しい内容は[こちらのブログポスト](https://blog.golang.org/strings)を参照してください。）