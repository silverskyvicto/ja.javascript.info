
```js run demo
let num;

do {
  num = prompt("Enter a number greater than 100?", 0);
} while (num <= 100 && num);
```

ループ `do..while` は両方のチェックが真になるまで繰り返します。:

1. `num <= 100` のチェック -- つまり、入力値がまだ `100` よりも大きくない。
2. `&& num` チェックは、`num` が `null` または空文字の場合に false です。そのとき、`while` ループも停止します。

P.S. `num`が `null` の場合、`num <= 100` は `true` なので、2回目のチェックがなければ、ユーザーがCANCELをクリックするとループは止まらなくなります。 両方のチェックが必要です。
