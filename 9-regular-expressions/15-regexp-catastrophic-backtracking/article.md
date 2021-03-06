# 破壊的なバックトラック(Catastrophic backtracking)

一部の正規表現は、単純に見えますが非常に実行時間が長く、JavaScript エンジンを "ハング" させることがあります。

遅かれ早かれ、多くの開発者はたまにこのような振る舞いに直面します。典型的な症状は、正規表現はときどきうまく機能しますが、特定の文字の場合 "ハング" し、CPUを 100% 消費します。

このような場合、Webブラウザはスクリプトを停止し、ページをリロードするよう提案します。これは確かに良いことではありません。

サーバサイド JavaScript では、このような正規表現はサーバプロセスをハングさせる可能性があり、より深刻です。そのため、絶対に見ておくべきことです。

## 例

文字列があり、それぞれの文字の後に任意のスペース `pattern:\s?` を持つ文字 `pattern:\w+` から構成されるかを確認したいとしましょう。

正規表現を組み立てる明白な方法は、文字の後に任意のスペース `pattern:\w+\s?` を取り、それを `*` で繰り返すことです。

これで、正規表現 `pattern:^(\w+\s?)*$` ができ、先頭 `pattern:^` から始まり、行末で終わる `pattern:$`、ゼロ個以上の単語を指します。

動作:

```js run
let regexp = /^(\w+\s?)*$/;

alert( regexp.test("A good string") ); // true
alert( regexp.test("Bad characters: $@#") ); // false
```

正規表現は動作しているように見え、結果も正しいです。ですが、特定の文字列の場合には非常に時間がかかります。それはJavaScript エンジンが CPU 100% 消費で "ハング" するほどの長さです。

以下の例を実行した場合、JavaScript が "ハング" し恐らくなにも表示されないでしょう。Webブラウザがイベントへ反応するのをやめ、UI は機能しなくなります（ほとんどのブラウザはスクロールだけ許可します）。しばらくすると、ページのリロードを提案するでしょう。なので、これには気をつけてください。

```js run
let regexp = /^(\w+\s?)*$/;
let str = "An input string that takes a long time or even makes this regexp to hang!";

// 非常に時間がかかります
alert( regexp.test(str) );
```

公平を期するために、正規表現のエンジンによってはこのような検索も効果的に扱えることに留意してください。ですが、それらの多くはできません。通常、ブラウザエンジンはハングします。

## 分かりやすい例

何がおきているのでしょう。なぜ正規表現がハングするのでしょう？

これを理解するために、分かりやすい例にしましょう: スペース v`pattern:\s?` を除きます。すると、`pattern:^(\w+)*$` になります。

そして、より物事を明確にするために、`pattern:\w` を `pattern:\d` に置き換えます。結果の正規表現は依然としてハングします。たとえば:

```js run
let regexp = /^(\d+)*$/;

let str = "012345678901234567890123456789z";

// 非常に時間がかかります (注意してください!)
alert( regexp.test(str) );
```

では、この正規表現の何が問題になっているでしょうか。

まず、正規表現 `pattern:(\d+)*` は少しおかしいことに気づくかもしれません。量指定子 `pattern:*` は無関係に見えます。数字が必要な場合は、`pattern:\d+` が使えます。

前の例を単純化することで得たものなので、確かに正規表現は不自然です。ですが、遅い理由は同じなので、これで理解していきましょう。そうすれば、前の例も明らかになります。

行 `subject:123456789z` （分かりやすくするために少し短くしました。末尾に数字以外の文字 `subject:z` があることに注意してください。重要です。）での `pattern:^(\d+)*$` の検索中には何が起きているのでしょうか、なぜそれほど時間がかかるのでしょう？

これは、正規表現エンジンが行っていることです:

1. 最初に、正規表現エンジンは括弧の内容を見つけようとします: 数値 `pattern:\d+` です。プラス `pattern:+` はデフォルトでは貪欲なので、これはすべての数値を消費します:

    ```
    \d+.......
    (123456789)z
    ```

    すべての数値が消費された後、`pattern:\d+` は見つけられたと判断されます（`match:123456789`）。
    
    次に、アスタリスク量指定子 `pattern:(\d+)*`  が適用されます。が、テキストにはもう数値はないので、アスタリスクは何もとりません。

    パターンの次の文字は文字列の終わり `pattern:$` です。しかし、テキストには代わりに `subject:z` があるのでマッチしません。

    ```
               X
    \d+........$
    (123456789)z
    ```

2. 一致しなかったので、貪欲量指定子 `pattern:+` は繰り返しの数を減らし、1文字戻ります。

    いま、 `pattern:\d+` は最後の1つを除いた全ての数値を取ります (`match:12345678`):
    ```
    \d+.......
    (12345678)9z
    ```
3. 次に、エンジンは次の位置(`match:12345678`の直後)から検索を続けようとします。

    アスタリスク `pattern:(\d+)*` が適用されます -- `pattern:\d+` のもう1つの一致、数値 `match:9` が与えられます。:

    ```

    \d+.......\d+
    (12345678)(9)z
    ```

    エンジンは 再び `pattern:$` への一致を試みますが、代わりに `subject:z` があるので失敗します。:

    ```
                 X
    \d+.......\d+
    (12345678)(9)z
    ```


4. 一致しないので、エンジンはバックトラッキングを続け、繰り返しの回数を減らしていきます。バックトラッキングは一般的にはこのように機能します: 最後の貪欲量指定子が、可能な限り繰り返し回数を減らします。次に、前の貪欲な量指定子が減少していきます。


    可能なすべての組み合わせが試行されます。これがその例です。

    最初の数値 `pattern:\d+` は7桁で、次は2桁の数値です:

    ```
                 X
    \d+......\d+
    (1234567)(89)z
    ```

    最初の数値は7桁で、次にそれぞれ1桁の数値が2つです:

    ```
                   X
    \d+......\d+\d+
    (1234567)(8)(9)z
    ```

    最初の数値は6桁で、次の数値は3桁です:

    ```
                 X
    \d+.......\d+
    (123456)(789)z
    ```

    最初の数値は6桁で、次は2つの数値です:

    ```
                   X
    \d+.....\d+ \d+
    (123456)(78)(9)z
    ```

    ...etc


数値の並び `123456789` を数値に分割する方法は多くあります。正確には、<code>2<sup>n</sup>-1</code> で、`n` は数字列の長さです。

- `123456789` の場合、`n=9` であり、 511 の組み合わせになります。
- `n=20` のより長い並びの場合、約 100万（1048575）の組み合わせになります。
- `n=30` なら、1000倍以上(1073741823 の組み合わせ)になります。


それらを1つずつを試みることが、検索に時間がかかる理由です。

## Back to words and strings

The similar thing happens in our first example, when we look words by pattern `pattern:^(\w+\s?)*$` in the string `subject:An input that hangs!`.

The reason is that a word can be represented as one `pattern:\w+` or many:

```
(input)
(inpu)(t)
(inp)(u)(t)
(in)(p)(ut)
...
```

For a human, it's obvious that there may be no match, because the string ends with an exclamation sign `!`, but the regular expression expects a wordly character `pattern:\w` or a space `pattern:\s` at the end. But the engine doesn't know that.

It tries all combinations of how the regexp `pattern:(\w+\s?)*` can "consume" the string, including variants with spaces `pattern:(\w+\s)*` and without them `pattern:(\w+)*` (because spaces `pattern:\s?` are optional). As there are many such combinations (we've seen it with digits), the search takes a lot of time.

What to do?

Should we turn on the lazy mode?

Unfortunately, that won't help: if we replace `pattern:\w+` with `pattern:\w+?`, the regexp will still hang. The order of combinations will change, but not their total count.

Some regular expression engines have tricky tests and finite automations that allow to avoid going through all combinations or make it much faster, but most engines don't, and it doesn't always help.

## How to fix?

There are two main approaches to fixing the problem.

The first is to lower the number of possible combinations.

Let's make the space non-optional by rewriting the regular expression as `pattern:^(\w+\s)*\w*$` - we'll look for any number of words followed by a space `pattern:(\w+\s)*`, and then (optionally) a final word `pattern:\w*`.

This regexp is equivalent to the previous one (matches the same) and works well:

```js run
let regexp = /^(\w+\s)*\w*$/;
let str = "An input string that takes a long time or even makes this regex to hang!";

alert( regexp.test(str) ); // false
```

Why did the problem disappear?

That's because now the space is mandatory.

The previous regexp, if we omit the space, becomes `pattern:(\w+)*`, leading to many combinations of `\w+` within a single word

So `subject:input` could be matched as two repetitions of `pattern:\w+`, like this:

```
\w+  \w+
(inp)(ut)
```

The new pattern is different: `pattern:(\w+\s)*` specifies repetitions of words followed by a space! The `subject:input` string can't be matched as two repetitions of `pattern:\w+\s`, because the space is mandatory.

The time needed to try a lot of (actually most of) combinations is now saved.

## Preventing backtracking

It's not always convenient to rewrite a regexp though. In the example above it was easy, but it's not always obvious how to do it. 

Besides, a rewritten regexp is usually more complex, and that's not good. Regexps are complex enough without extra efforts.

Luckily, there's an alternative approach. We can forbid backtracking for the quantifier.

The root of the problem is that the regexp engine tries many combinations that are obviously wrong for a human.

E.g. in the regexp `pattern:(\d+)*$` it's obvious for a human, that `pattern:+` shouldn't backtrack. If we replace one `pattern:\d+` with two separate `pattern:\d+\d+`, nothing changes:

```
\d+........
(123456789)!

\d+...\d+....
(1234)(56789)!
```

And in the original example `pattern:^(\w+\s?)*$` we may want to forbid backtracking in `pattern:\w+`. That is: `pattern:\w+` should match a whole word, with the maximal possible length. There's no need to lower the repetitions count in `pattern:\w+`, try to split it into two words `pattern:\w+\w+` and so on.

Modern regular expression engines support possessive quantifiers for that. Regular quantifiers become possessive if we add `pattern:+` after them. That is, we use `pattern:\d++` instead of `pattern:\d+` to stop `pattern:+` from backtracking.

Possessive quantifiers are in fact simpler than "regular" ones. They just match as many as they can, without any backtracking. The search process without bracktracking is simpler.

There are also so-called "atomic capturing groups" - a way to disable backtracking inside parentheses.

...But the bad news is that, unfortunately, in JavaScript they are not supported. 

We can emulate them though using a "lookahead transform".

### Lookahead to the rescue!

So we've come to real advanced topics. We'd like a quantifier, such as `pattern:+` not to backtrack, because sometimes backtracking makes no sense.

The pattern to take as much repetitions of `pattern:\w` as possible without backtracking is: `pattern:(?=(\w+))\1`. Of course, we could take another pattern instead of `pattern:\w`.

That may seem odd, but it's actually a very simple transform.

Let's decipher it:

- Lookahead `pattern:?=` looks forward for the longest word `pattern:\w+` starting at the current position.
- The contents of parentheses with `pattern:?=...` isn't memorized by the engine, so wrap `pattern:\w+` into parentheses. Then the engine will memorize their contents
- ...And allow us to reference it in the pattern as `pattern:\1`.

That is: we look ahead - and if there's a word `pattern:\w+`, then match it as `pattern:\1`.

Why? That's because the lookahead finds a word `pattern:\w+` as a whole and we capture it into the pattern with `pattern:\1`. So we essentially implemented a possessive plus `pattern:+` quantifier. It captures only the whole word `pattern:\w+`, not a part of it.

For instance, in the word `subject:JavaScript` it may not only match `match:Java`, but leave out `match:Script` to match the rest of the pattern.

Here's the comparison of two patterns:

```js run
alert( "JavaScript".match(/\w+Script/)); // JavaScript
alert( "JavaScript".match(/(?=(\w+))\1Script/)); // null
```

1. In the first variant `pattern:\w+` first captures the whole word `subject:JavaScript` but then `pattern:+` backtracks character by character, to try to match the rest of the pattern, until it finally succeeds (when `pattern:\w+` matches `match:Java`).
2. In the second variant `pattern:(?=(\w+))` looks ahead and finds the word  `subject:JavaScript`, that is included into the pattern as a whole by `pattern:\1`, so there remains no way to find `subject:Script` after it.

We can put a more complex regular expression into `pattern:(?=(\w+))\1` instead of `pattern:\w`, when we need to forbid backtracking for `pattern:+` after it.

```smart
There's more about the relation between possessive quantifiers and lookahead in articles [Regex: Emulate Atomic Grouping (and Possessive Quantifiers) with LookAhead](http://instanceof.me/post/52245507631/regex-emulate-atomic-grouping-with-lookahead) and [Mimicking Atomic Groups](http://blog.stevenlevithan.com/archives/mimic-atomic-groups).
```

Let's rewrite the first example using lookahead to prevent backtracking:

```js run
let regexp = /^((?=(\w+))\2\s?)*$/;

alert( regexp.test("A good string") ); // true

let str = "An input string that takes a long time or even makes this regex to hang!";

alert( regexp.test(str) ); // false, works and fast!
```

Here `pattern:\2` is used instead of `pattern:\1`, because there are additional outer parentheses. To avoid messing up with the numbers, we can give the parentheses a name, e.g. `pattern:(?<word>\w+)`.

```js run
// parentheses are named ?<word>, referenced as \k<word>
let regexp = /^((?=(?<word>\w+))\k<word>\s?)*$/;

let str = "An input string that takes a long time or even makes this regex to hang!";

alert( regexp.test(str) ); // false

alert( regexp.test("A correct string") ); // true
```

The problem described in this article is called "catastrophic backtracking".

We covered two ways how to solve it:
- Rewrite the regexp to lower the possible combinations count.
- Prevent backtracking.
