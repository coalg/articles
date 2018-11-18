# Profunctor Opticsについて

> source: [Don't Fear the Profunctor Optics](https://github.com/hablapps/DontFearTheProfunctorOptics)

Opticsとは、レコードのフィールド、共用体のヴァリアント、コンテナの要素といった、あるデータ構造の構成要素を読み書きするためのアクセサの総称である。ここではOpticsの具体例としてLensを取り上げる。

## Lens

雑に言うと、Lensはあるデータ全体になんらかの焦点(focus)を絞ってアクセスするためのデータ構造である。ここでの「アクセスする」ということはどういうことかというと、ある与えられたデータ全体の焦点に対してviewとupdateができるということである。以下では全体を`s`，焦点を`a`としてviewとupdateを定義する。

```haskell
data Lens s a = Lens { view   :: s -> a
                     , update :: (a, s) -> s }
```

2-タプルの第一要素にアクセスする`π1`は次のように定義できる。

```haskell
π1 :: Lens (a, b) a
π1 = Lens v u where
    v = fst
    u (a', (_, b)) = (a', b)
```

使い方は以下のようになる。

```haskell
λ> view π1 (1, 'a')
1
λ> update π1 (2, (1, 'a'))
(2,'a')
```

これは便利だが、もっと良くすることができる。フォーカスしたとき型を変更して良いことにしてやるのである。つまり先ほどの定義では`(1, 'a')`を`("hi", 'a')`にすることはできなかった。これは多態性がなく弱いので、Lensの型を変更してやる。

```haskell
data Lens s t a b = Lens { view :: s -> a
                         , update :: (b, s) -> t }
```

`π1`の定義は以下のようになる。

```haskell
pi1 :: Lens (a, c) (b, c) a b
pi1 = Lens v u where
    v = fst
    u (b, (_, c)) = (b, c)
```

驚くべきことに、型以外の実装は先ほどの定義から変化していない。

```haskell
λ> update π1 ("hi", (1, 'a'))
("hi",'a')
```

Lensは以下の法則を満たす。

```haskell
viewUpdate :: Eq s => Lens s s a a -> s -> Bool
viewUpdate (Lens v u) s = u ((v s), s) == s

updateView :: Eq a => Lens s s a a -> a -> s -> Bool
updateView (Lens v u) a s = v (u (a, s)) == a

updateUpdate :: Eq s => Lens s s a a -> a -> a -> s -> Bool
updateUpdate (Lens v u) a1 a2 s = u (a2, (u (a1, s))) == u (a2, s)
```

この法則について非形式的に述べると、`update`はある焦点を排他的に変更すること、また`view`はフォーカスした値をそのまま抽出できることを確認している。

さて、タプルを変更したり要素を取り出すだけなら、Lensのような仰々しいシロモノが本当に必要なのかと思われるかもしれない。Lensは入れ子になった不変データ構造と向き合うとき、真の価値を発揮する。**Opticsは合成則を満たす**。これはOpticsの特徴の際たるものである。

例として、タプルが2重入れ子になっている以下のデータを考える。

```haskell
λ> update (π1 |.| π1 |.| π1) ("hi", (((1, 'a'), 2.0), True))
```

ここで合成関数 `|.|` は以下のように定義されている。

```haskell
(|.|) :: Lens s t a b -> Lens a b c d -> Lens s t c d
(Lens v1 u1) |.| (Lens v2 u2) = Lens v u where
    v = v2 . v1
    u (d, s) = u1 (u2 (d, v1 s), s)
((("hi",'a'),2.0),True)
```

とはいえ、このopticsの合成はぎこちないものである。なぜかというと、別の種類のopticsを合成しようとしたときに、また別の合成関数を定義する必要があるからだ。ライブラリごとの具体的な型について定義を考えると冗長の極みとなる。ライブラリのユーザーは、その合成関数についても深い知識を持たなければ使うことができなくなるだろう。これはのちに述べるprofunctor opticsによって解決される。
