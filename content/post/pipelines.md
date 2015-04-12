+++
date = "2014-03-13T08:19:15+09:00"
draft = true
title = "Goの並行パターン：パイプラインとキャンセル (Go Concurrency Patterns: Pipelines and cancellation)"
tags = ["concurrency", "pipeline", "cancellation"]
+++

# Goの並行パターン：パイプラインとキャンセル
[Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines) by Sameer Ajmani

## はじめに
Goの並行性に関する基本要素によって、I/Oや複数のCPIを効率的に使うことができるストリーミングデータパイプラインを
簡単に構築することができます。この記事ではそのようなパイプラインの例を紹介し、操作が失敗したときに発生する
繊細な事柄にハイライトを当て、また失敗に綺麗に対応するテクニックを紹介します。

## パイプラインとはなにか
Goにおいて、パイプラインの厳密な定義はありません。パイプラインは数ある並行プログラミングの種類の一つに過ぎません。
正式な定義ではないですが、パイプラインとはチャンネルによって接続された一連の _ステージ_ を挿します。
そこでは、各ステージでは同じ関数を実行するゴルーチンのまとまりになっています。
各ステージではゴルーチンは次の役割を果たします。

* _上流_ から _流入_ チャンネル経由で値を受け取る
* そのデータに対してある関数を実行し、通常は新しい値を生成する
* _下流_ へ _流出_ チャンネル経由で値を送信する

各ステージでは、任意の数の流入と流出のチャンネルを持っています。ただし最初と最後のステージは例外で、
それぞれ流出と流入のチャンネルのみが存在します。最初のステージは時々 _ソース_ あるいは _プロデューサー_ と呼ばれ、
最後のステージは _シンク_ あるいは _コンシューマー_ と呼ばれます。

パイプラインの考え方とそのテクニックを説明するために単純なパイプラインの例から始めてみましょう。
あとでより現実的な例を紹介します。