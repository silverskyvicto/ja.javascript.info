<<<<<<< HEAD
コメントの先頭 `match:<!--` を見つけ、その後、 `match:-->` で終わるまでのすべてを見つける必要があります。

最初のアイデアは `pattern:<!--.*?-->` です -- 怠惰な量指定子は `match:-->` の直前でドットを停止させます。

しかし、JavaScriptのドットは "改行以外の任意の文字" を意味するので、複数行のコメントは見つかりません。

"なんでも" マッチさせるために、ドットの代わりに `pattern:[\s\S]` を使います。:

```js run
let reg = /<!--[\s\S]*?-->/g;
=======
We need to find the beginning of the comment `match:<!--`, then everything till the end of `match:-->`.

An acceptable variant is `pattern:<!--.*?-->` -- the lazy quantifier makes the dot stop right before `match:-->`. We also need to add flag `pattern:s` for the dot to include newlines.

Otherwise multiline comments won't be found:

```js run
let regexp = /<!--.*?-->/gs;
>>>>>>> cd2c7ce3c8f033e6f7861ed1b126552e41ba3e31

let str = `... <!-- My -- comment
 test --> ..  <!----> ..
`;

<<<<<<< HEAD
alert( str.match(reg) ); // '<!-- My -- comment \n test -->', '<!---->'
=======
alert( str.match(regexp) ); // '<!-- My -- comment \n test -->', '<!---->'
>>>>>>> cd2c7ce3c8f033e6f7861ed1b126552e41ba3e31
```