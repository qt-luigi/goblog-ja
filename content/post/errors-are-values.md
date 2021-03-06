+++
date = "2015-01-12T12:00:00+09:00"
draft = false
title = "エラーは値 (Errors are values)"
+++

# エラーは値

[Errors are values](http://blog.golang.org/errors-are-values) by Rob Pike

Goプログラマ、特にまだGoに不慣れな開発者に共通する議論の話題といえば、エラー処理の方法でしょう。
議論が次のようなコードの連続になることを嘆く結論に至ることがしばしばあります。

```
if err != nil {
    return err
}
```

先日、確認できるすべてのオープンソースプロジェクトをスキャンしてみたところ、このスニペットは
先のような開発者が信じているほどではなく、せいぜい1ページに1つか2つ現れる程度であることがわかりました。
それでもなお、いつも次のイディオムをタイプしなければいけないと強く信じているのであれば、
それは何かが間違っていますし、明らかに問題の対象はGoそれ自信となってしまいます。

```
if err != nil
```

これは不幸なことですし、語弊があり、そして容易に訂正が可能なことです。
おそらく、そのように信じてしまっているような状況になったのは、信じているプログラマがGoを使い始めて
まだ日が浅く、「エラーをどう処理したらよいか」という問いに対して、このパターンを覚えて、
そこで止まってしまっているのだと思います。他の言語ではtry-catch節や他の同様のエラー処理機構を
使っているのでしょう。それゆえ、そのプログラマは、私が古い言語ではtry-catch節を使っていたような場合でも、
Goではただ `if err != nil` と打っていると考えているのでしょう。時間が経つにつれて、
Goではこのようなスニペットが蔓延しだし、その結果不格好になってしまいました。

この表現がしっくり来るかどうかはわかりませんが、こういったGoプログラマはエラーに関して根本的な点を
見失っています。 _エラーは値です。_

値はプログラムが可能で、エラーは値なので、エラーはプログラム可能なのです。

もちろん、エラー値がnilかどうかを検証する文はよくありますが、エラー値でできることは他にもたくさんあります。
そしてそれらをあなたのプログラムに適用することで、プログラムが改善され、丸暗記で使っているおきまりの形のif文を
排除することができます。

次のスニペットは `bufio` パッケージの [Scanner](http://golang.org/pkg/bufio/#Scanner) 型の簡単な例です。
Scanner型の [Scan](http://golang.org/pkg/bufio/#Scanner.Scan) メソッドは、Scannerの中にあるI/Oを処理し、
その中ではもちろんエラーが発生するでしょう。しかし `Scan` メソッドはエラーを一切公開しません。
代わりにbool値を返し、別のメソッドがスキャンが終わったあとに、エラーが発生したかを報告します。
クライアント側のコードは次のようになります。

```
scanner := bufio.NewScanner(input)
for scanner.Scan() {
    token := scanner.Text()
    // tokenの処理
}
if err := scanner.Err(); err != nil {
    // エラーの処理
}
```

たしかに、エラーがnilかどうかの確認はしていますが、その処理は一度しかしていません。 `Scan` メソッドは
代わりに次のように定義することもできたでしょう。

```
func (s *Scanner) Scan() (token []byte, error)
```

この場合はクライアント側のコードは次のようになるでしょう。（トークンの読み出し方に依存します）

```
scanner := bufio.NewScanner(input)
for {
    token, err := scanner.Scan()
    if err != nil {
        return err // あるいはbreak
    }
    // tokenの処理
}
```
さきほどと大きな違いはありませんが、唯一の重要な違いがあります。後者のコードでは、クライアントは
繰り返しの度にエラーの確認をしなければなりません。しかし実際の `Scanner` のAPIでは
エラー処理は、トークンを繰り返し取得する肝心なAPIの要素からは抽象化され切り離されています。
それゆえ、実際のAPIでは、クライアント側のコードはより自然な形になります。
トークンを取り出す処理が完了するまでループして、エラーのことは考えずにすみます。
エラー処理が一連のトークンの処理を分かりにくくすることはありません。

抽象化の下で、実際に何が起きているかですが、もちろん、 `Scan` がI/Oエラーに遭遇したら、
すぐさまそれを記録し、 `false` を返します。クライアントが別のメソッドである `Err` を呼び出したら、
そのエラーの値を返します。これは些細な事ではありますが、これは

```
if err != nil
```

をクライアントのコード内のあちこちに書いたり、トークンごとにエラーを確認することとは違います。
これがエラー値とプログラミングをするということです。単純なプログラミングですが、
それでもこれがプログラミングなのです。

設計のされかたに関わらず、どのようにエラーが公開されていても、それをプログラムが確認することは非常に重要です。
これから議論することはどのようにエラーチェックを避けるかという話ではなく、いかにエラー処理を優雅に行うかという話です。

これから話すエピソードは、東京で開催されたGoCon 2014 autumnに参加した際に話題になった、
エラーの確認処理が繰り返し現れるコードについてです。熱心なGopherである [@jxck_](https://twitter.com/jxck_) が
よく嘆かれているエラーの確認処理について話していました。
彼が書いたコードは、目的としてはこのようなものでした。

```
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// 以下続く
```

このコードは繰り返しが多いですね。実際のコードでは、さらに多く繰り返しがあり、ヘルパー関数を使ってリファクタリングするのは
容易なことではありませんでしたが、理想的な形にすると、エラー値に対する関数リテラルが助けになるでしょう。

```
var err error
write := func(buf []byte) {
    if err != nil {
        return
    }
    _, err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])
// 以下続く
if err != nil {
    return err
}
```

このパターンはそれなりにうまくいきますが、書き込みを行う関数それぞれに対しクロージャが必要になります。
別々のヘルパー関数を用意するのはよりぎこちない書き方になってしまいます。なぜなら、 `err` 変数を
それらの関数の呼び出しごとに管理する必要があるからです。（試してみてください。）

先に触れた `Scan` メソッドでの考え方を借りて、このコードをより綺麗で、一般的で、再利用可能な形にできます。
このテクニックを議論の中で @jxck_ に伝えたのですが、どのように適用するかまでは伝わらなかったようでした。
色々と意見を交わしつつ、いくらか言語の障壁に阻まれながら、最終的に彼にラップトップ借りる許可を得て、
コードを書くことでその方法を実演しました。

私は次のような `errWriter` というオブジェクトを定義しました。

```
type errWriter struct {
    w   io.Writer
    err error
}
```

そして、 `write` という1つのメソッドを付与しました。これは標準的な `Write` というシグネチャで
ある必要はなく、また違いを明確にするという理由もあって、小文字にしました。 `write` メソッドは
`errWriter` の中にある `Writer` の `Write` メソッドを呼び、後に参照される最初のエラーを
記録します。

```
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

エラーが発生すると、ただちに `write` メソッドは何も処理しなくなり、最初のエラーが保存されただけの状態になります。

この `errWrite` 型と `write` メソッドで、先のコードは次のようにリファクタリングされます。

```
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// 以下続く
if ew.err != nil {
    return ew.err
}
```

このコードは、クロージャを用いた場合と比べても、より簡潔になっています。また実際に書き込みを行っている一連の部分は
ページ内で読みやすい形で行われています。もう取り散らかったコードはありません。エラー値（とインターフェース）を使ったプログラミングで
コードがより素敵になりました。

同じパッケージ内の他のコードがこの考えに乗ること、あるいは直接 `errWriter` を使うことはありえます。

また、 `errWriter` があれば、特にもっとわざとらしくない例で、より多くのことができるようになります。
バイト数を積算していくこともできます。アトミックに一つのバッファに書き込みを行うこともできます。
さらにもっとたくさんのことができます。

事実、このパターンは標準ライブラリによく出てきます。 `archive/zip` や `net/http` といったパッケージが使っています。
より顕著な例としては `bufio` パッケージが実際に `errWriter` の考え方を実装しています。
`bufio.Writer.Write` はエラーを返しますが、それは `io.Writer` インターフェースを考慮してのことです。
`bufio.Writer` の `Write` メソッドは、ちょうど先の例の `errWriter.write` メソッドと同様の振る舞いになっています。
`Flush` がエラーを出力するので、先の例はこのように書くことができます。

```
b := bufio.NewWriter(fd)
b.Write(p0[a:b])
b.Write(p1[c:d])
b.Write(p2[e:f])
// 以下続く
if b.Flush() != nil {
    return b.Flush()
}
```

この手法には、少なくともいくつかのアプリケーションにおいては、一つ重大な欠点があります。
エラーが発生するまで、処理がどの程度行われたかを知る術がないのです。それを知るためには、
よりきめ細やかな手法が必要となります。もっとも、多くの場合では最後に全か無かの確認さえすれば十分です。

エラー処理の繰り返しを避けるための手法の一つだけをみていました。心に留めておいてほしいことは、
`errWriter` の利用、つまり `bufio.Writer` の手法がエラー処理を簡潔にする唯一の手法ではなく、
この手法はすべての状況に適しているわけではないということです。しかしながら、ここで学んだ肝心な点は、
エラーは値であり、Goプログラミング言語の全能力をもってすれば、それらを処理することは可能であるということです。

エラー処理を簡潔にするように言語を使いましょう。

しかし覚えておいてください。何をするにおいても、常にエラー処理を行いましょう！

最後に、 @jxck_ さんとのやりとりのすべてを見たい方は、彼が録画した動画とともに、[彼のブログ](http://jxck.hatenablog.com/entry/golang-error-handling-lesson-by-rob-pike) も読んでみましょう。

By Rob Pike
