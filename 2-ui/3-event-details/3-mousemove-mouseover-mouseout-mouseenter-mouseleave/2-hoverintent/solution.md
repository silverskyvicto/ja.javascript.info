
アルゴリズムはシンプルです:
1. 要素上に `onmouseover/out` ハンドラを置きます。また、ここでは `onmouserenter/leave` を使うこともできますが、汎用性が下がり、移譲を導入すると上手く動作しません。
2. マウスカーソルが要素に入ったとき、`mousemove` で速度の計算を開始します。
3. もし速度が遅い場合、`over` を実行します。
4. その後、要素から出て、`over` が実行された場合には `out` を実行します。

質問: "どうやって速度を測る？"

最初のアイデア: `100ms` 毎に関数を実行し、前後の座標間の距離を計算する方法です。もしそれが小さい場合、スピードは小さいです。

残念ながら、JavaScript で "現在のマウス座標" を取得する方法はありません。`getCurrentMouseCoordinates()` のような関数はありません。

座標を取得する唯一の方法は、`mousemove` のようにマウスイベントをリッスンすることです。

したがって、座標を追跡しそれを覚えるために `mousermove` のハンドラを設定できます。

P.S. 注意: 解決策のテストでは、`dispatchEvent` を使用して、ツールチップが正しく動作するかを確認します。
