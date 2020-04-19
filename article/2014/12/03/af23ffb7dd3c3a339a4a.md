---
title: 双方向データバインディングとChangeListenerの関係について
tags: Java:8u25 JavaFX:8
author: harry0000
slide: false
---
# 前書き

JavaFX 8の`Slider.valueProperty().addListener()`と`Slider.valueProperty().bindBidirectional()`に関してソースコードを読んで確認した結果をまとめたコラム的な内容になっています。

長いので結論から書きたいのですが、順を追って説明した方が分かりやすそうなので、このような構成になっています。
一応結論は[まとめ](#まとめ)として最後に書きました。

# 本題

2014/11/25に行われた[JavaFX Night](http://javafx.doorkeeper.jp/events/16670)でデータバインディングに関する発表を拝見しました。
その中で2つのスライダーの値(Property)を双方向にバインドし、片方のスライダーを動かすともう片方のスライダーも動くというデモがありました。

そこで疑問に思ったのが、それぞれのスライダーのPropertyに値の変更時に呼ばれる`ChangeListener`を登録した場合、スライダーを動かした時にどのような順で実行されるのだろうか、という事です。

# とりあえず実際にやってみる

簡単なデモアプリを作成し、ソースコードをgithubに上げました。
ビルドにはJDK 1.8とMaven 3.xが必要です。
https://github.com/harry0000/ListenerViewer

以下、コントローラーの初期化処理の抜粋です。
2つのスライダーを双方向にデータバインディングし、スライダーを動かした際、もう片方のスライダーも一緒に動くようにします。
そして次に`ChangeListener`を`addListener()`し、値が変更された際、`TextArea`(変数名`log`)にテキストを出力します。
(`buildText()`はテキスト整形処理)

```ListenerViewerController.java
    @Override
    public void initialize(URL location, ResourceBundle resources) {
        slider1.valueProperty().bindBidirectional(slider2.valueProperty());

        slider1.valueProperty().addListener(
            (observable, oldValue, newValue) -> {
                log.appendText(buildText(slider1, oldValue, newValue));
            }
        );
        slider2.valueProperty().addListener(
            (observable, oldValue, newValue) -> {
                log.appendText(buildText(slider2, oldValue, newValue));
            }
        );
    }
```

さて、Slider1を動かしてみましょう。
Slider1を動かすんだからSlider1→Slider2の順で`ChangeListener`が動作するはず・・・

![move_slider1.png](https://qiita-image-store.s3.amazonaws.com/0/42716/f94918ff-3f4b-8d0f-8536-d7d9c5b2ff2d.png)

！！！！？？？！！！？！？
Slider2を動かすとSlider1→Slider2の順で`ChangeListener`が動作します。なんでや！

　

その後の調査中、何気なく`bindBidirectional()`と`addListener()`の実行順を入れ替えて、Slider1を動かしてみました。
つまり`ChangeListener`を登録した後にデータバインディングを行います。
いや結果変わらんやろ、念の為や・・・

![move_slider1_2.png](https://qiita-image-store.s3.amazonaws.com/0/42716/f720965f-6b23-5139-5983-0605f9eb2c48.png)

！！！！！？！？！？！？？？！？！？！？！？？！？！！！？

# `addListener()`は内部で何をしているのか

とりあえず最初に`Slider.valueProperty().addListener()`の内部動作から見ました。[^1]
`Slider`は値を`Double`で保持していますが、`value`は`DoubleProperty`のサブクラス`DoublePropertyBase`で初期化されます。

```Slider.java
    public final DoubleProperty valueProperty() {
        if (value == null) {
            value = new DoublePropertyBase(0) {
                @Override protected void invalidated() {
                    adjustValues();
//                    accSendNotification(Attribute.VALUE);
                }

                @Override
                public Object getBean() {
                    return Slider.this;
                }

                @Override
                public String getName() {
                    return "value";
                }
            };
        }
        return value;
    }
```

その為、実際には`DoublePropertyBase.addListener()`が呼ばれます。
なにやら`ExpressionHelper`を通してListenerの管理や実行をしているようです。

```DoublePropertyBase.java
    private ExpressionHelper<Number> helper = null;

    @Override
    public void addListener(ChangeListener<? super Number> listener) {
        helper = ExpressionHelper.addListener(helper, this, listener);
    }

    /**
     * Sends notifications to all attached
     * {@link javafx.beans.InvalidationListener InvalidationListeners} and
     * {@link javafx.beans.value.ChangeListener ChangeListeners}.
     *
     * This method is called when the value is changed, either manually by
     * calling {@link #set(double)} or in case of a bound property, if the
     * binding becomes invalid.
     */
    protected void fireValueChangedEvent() {
        ExpressionHelper.fireValueChangedEvent(helper);
    }
```

```ExpressionHelper.java
    public static <T> ExpressionHelper<T> addListener(ExpressionHelper<T> helper, ObservableValue<T> observable, ChangeListener<? super T> listener) {
        if ((observable == null) || (listener == null)) {
            throw new NullPointerException();
        }
        return (helper == null)? new SingleChange<T>(observable, listener) : helper.addListener(listener);
    }

    public static <T> void fireValueChangedEvent(ExpressionHelper<T> helper) {
        if (helper != null) {
            helper.fireValueChangedEvent();
        }
    }
```

```ExpressionHelper$SingleChange.java
        @Override
        protected ExpressionHelper<T> addListener(ChangeListener<? super T> listener) {
            return new Generic<T>(observable, this.listener, listener);
        }
```

`ExpressionHelper.addListener()`の動作は以下のようです。
1. 初回の`addListener()`では`SingleChange`を返す
2. 2回目は`SingleChange.addListener()`で`Generic`を返す
3. 以降、`Generic.addListener()`が実行され、`ChangeListener`を管理する配列に追加されていく

`SingleChange`は単一の`ChangeListener`を扱うクラスで、`Generic`は複数の`ChangeListener`や`InvalidationListener`を扱う為のクラスのようです。

```
ExpressionHelper<T>
 ├─ SingleChange<T>
 └─ Generic<T>
```

`Generic.fireValueChangedEvent()`では配列に格納している`ChangeListener`をforループで回して実行しています。
つまり、1つの`Slider.valueProperty()`対して複数の`ChangeListener`を登録した場合、登録した順で`ChangeListener`が実行されます。
まぁ普通ですね。

# `bindBidirectional()`は内部で何をしているのか

次に`Slider.valueProperty().bindBidirectional()`の内部動作を見ます。
`DoubleProperty.bindBidirectional()`が呼ばれますので辿ってみると・・・

```DoubleProperty.java
    /**
     * {@inheritDoc}
     */
    @Override
    public void bindBidirectional(Property<Number> other) {
        Bindings.bindBidirectional(this, other);
    }
```

```Bindings.java
    public static <T> void bindBidirectional(Property<T> property1, Property<T> property2) {
        BidirectionalBinding.bind(property1, property2);
    }
```

```BidirectionalBinding.java
    public static <T> BidirectionalBinding bind(Property<T> property1, Property<T> property2) {
        checkParameters(property1, property2);
        final BidirectionalBinding binding =
                ((property1 instanceof DoubleProperty) && (property2 instanceof DoubleProperty)) ?
                        new BidirectionalDoubleBinding((DoubleProperty) property1, (DoubleProperty) property2)
                : ((property1 instanceof FloatProperty) && (property2 instanceof FloatProperty)) ?
                        new BidirectionalFloatBinding((FloatProperty) property1, (FloatProperty) property2)
                : ((property1 instanceof IntegerProperty) && (property2 instanceof IntegerProperty)) ?
                        new BidirectionalIntegerBinding((IntegerProperty) property1, (IntegerProperty) property2)
                : ((property1 instanceof LongProperty) && (property2 instanceof LongProperty)) ?
                        new BidirectionalLongBinding((LongProperty) property1, (LongProperty) property2)
                : ((property1 instanceof BooleanProperty) && (property2 instanceof BooleanProperty)) ?
                        new BidirectionalBooleanBinding((BooleanProperty) property1, (BooleanProperty) property2)
                : new TypedGenericBidirectionalBinding<T>(property1, property2);
        property1.setValue(property2.getValue());
        property1.addListener(binding);
        property2.addListener(binding);
        return binding;
    }
```

なんか`ChangeListener`を継承してる`BidirectionalDoubleBinding`をSlider1とSlider2の`valueProperty()`に`addListener()`してますね・・・

自身の`BidirectionalDoubleBinding.changed()`が実行されると、`updating`フラグをtrueにしてから他方のPropertyの値を更新して同期するようです。

`DoublePropertyBase.set()`で実際に値が更新されると`DoublePropertyBase.fireValueChangedEvent()`が実行される為、こちらのPropertyの`BidirectionalDoubleBinding.changed()`も呼ばれますが、`updating`フラグがまだtrueなので何もせず処理を抜けます。

```BidirectionalBinding$BidirectionalDoubleBinding.java
        @Override
        public void changed(ObservableValue<? extends Number> sourceProperty, Number oldValue, Number newValue) {
            if (!updating) {
                final DoubleProperty property1 = propertyRef1.get();
                final DoubleProperty property2 = propertyRef2.get();
                if ((property1 == null) || (property2 == null)) {
                    if (property1 != null) {
                        property1.removeListener(this);
                    }
                    if (property2 != null) {
                        property2.removeListener(this);
                    }
                } else {
                    try {
                        updating = true;
                        if (property1 == sourceProperty) {
                            property2.set(newValue.doubleValue());
                        } else {
                            property1.set(newValue.doubleValue());
                        }
                    } catch (RuntimeException e) {
                        try {
                            if (property1 == sourceProperty) {
                                property1.set(oldValue.doubleValue());
                            } else {
                                property2.set(oldValue.doubleValue());
                            }
                        } catch (Exception e2) {
                            e2.addSuppressed(e);
                            unbind(property1, property2);
                            throw new RuntimeException(
                                "Bidirectional binding failed together with an attempt"
                                        + " to restore the source property to the previous value."
                                        + " Removing the bidirectional binding from properties " +
                                        property1 + " and " + property2, e2);
                        }
                        throw new RuntimeException(
                                        "Bidirectional binding failed, setting to the previous value", e);
                    } finally {
                        updating = false;
                    }
                }
            }
        }
```

```DoublePropertyBase.java
    private void markInvalid() {
        if (valid) {
            valid = false;
            invalidated();
            fireValueChangedEvent();
        }
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void set(double newValue) {
        if (isBound()) {
            throw new java.lang.RuntimeException("A bound value cannot be set.");
        }
        if (value != newValue) {
            value = newValue;
            markInvalid();
        }
    }
```

# つまり・・・？

初期化処理でSlider1とSlider2それぞれの`valueProperty()`に対して、実は2つの`ChangeListener`が登録されていました。

* 初期化処理
  1. `bindBidirectional()`でSlider1とSlider2に`BidirectionalDoubleBinding`が`addListener()`される
  2. Slider1に`ChangeListener`を`addListener()`する
  3. Slider2に`ChangeListener`を`addListener()`する

Slider1を動かした場合、下記のような動作が行われる為、Slider2→Slider1の順で`TextArea`にテキストが出力される事になります。

* Slider1を動かした時
  1. Slider1の値が更新される
  2. Slider1の`Generic.fireValueChangedEvent()`が実行される
  3. Slider1に最初に登録された`BidirectionalDoubleBinding.changed()`でSlider2の値が更新される
  4. Slider2の`Generic.fireValueChangedEvent()`が実行される
  5. Slider2に最初に登録された`BidirectionalDoubleBinding.changed()`が実行される  
(3.で`updating`フラグがtrueのままなので何も行わない)
  6. Slider2に登録した`ChangeListener.changed()`が実行される
  7. Slider1に登録した`ChangeListener.changed()`が実行される

初期化処理で`bindBidirectional()`と`addListener()`の実行順を入れ替えた場合、`BidirectionalDoubleBinding`と`ChangeListener`の登録順が逆転する為、Slider1→Slider2の順でテキストが出力されていたわけです。

# まとめ

`Slider.valueProperty()`での動作を基に双方向データバインディングと`ChangeListener`の関係を見てきました。

双方向データバインディングを行う`bindBidirectional()`を実行すると、値を同期する為の`ChangeListener`がそれぞれの`valueProperty()`に対して登録されていました。

その為、登録した`ChangeListener`が実行されるタイミングは、Propertyに対してどのような双方向データバインディングがどのような順序で設定されているか(または設定するか)によって変わる、という結論になります。

あくまで興味本位で行った調査でしたが、双方向データバインディングを行っている各Propertyに登録した`ChangeListener`の実行順序は制御できそうな事が分かりました。[^2]
ですが`ChangeListener`の実行順序に依存する処理の設計は複雑になり過ぎる為、個人的にはしない方が良さそうな印象を受けました。[^3]

最後にソースコードで確認した項目についても箇条書きでまとめておきます。

* Slider.valueProperty().addListener()
  * `DoublePropertyBase.addListener()`で追加した`ChangeListener`は、`SingleChange`や`Generic`で管理される
  * 複数登録した`ChangeListener`は配列で保持され、`Generic.fireValueChangedEvent()`で登録順に実行される  
    　
* Slider.valueProperty().bindBidirectional()
  * `DoublePropertyBase.bindBidirectional()`は、それぞれのPropertyに`BidirectionalDoubleBinding`を`addListener()`する
  * `BidirectionalDoubleBinding.changed()`は、自身の値を他方の`DoubleProperty`にsetし値を同期する  
(それぞれのPropertyにこの処理を行うListenerが登録される為、双方向にデータバインディングされる)
  * `BidirectionalDoubleBinding.changed()`は、`updating`フラグにより値の更新中にそれぞれのPropertyから呼び出されても1度しか処理を行わない

# 参考リンク

* [Java技術最前線 - Java技術最前線：ITpro](http://itpro.nikkeibp.co.jp/article/COLUMN/20060915/248243/)
* [JavaFX Night でバインドの発表をしてきた - JavaFX in the Box](http://skrb.hatenablog.com/entry/2014/12/01/173548)

---
[^1]:  データバインディングよりは簡単そうなイメージがあった為。
[^2]: ゼロから注意深く設計・実装すれば。またSlider以外のコントロールについて、今回のような調査は行っておらず、あくまで推測です。
[^3]: 実は当初3つのSliderで動作の確認・検証をしていました。3つ以上のコントロールを双方向にバインドした際の動作は、私の頭の処理能力を完全に超えています。
