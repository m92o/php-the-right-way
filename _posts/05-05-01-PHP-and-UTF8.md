---
title: PHPとUTF-8
isChild: true
anchor: php_and_utf8
---

## PHPとUTF-8 {#php_and_utf8_title}

_このセクションは、もともと[Alex Cabal](https://alexcabal.com/)が
[PHP Best Practices](https://phpbestpractices.org/#utf-8)向けに書いたものだ。ここでも共有する。_

### 一発で済ませる方法はない。注意して、きちんと一貫性を保つこと。

今のところPHPは、低レベルではUnicodeをサポートしていない。
PHPでUTF-8文字列をきちんと処理する方法もあるにはあるが、簡単ではない。さらに、ウェブアプリケーションのあらゆるレベル
（つまり、HTMLやSQLからPHPまで）に手を入れる必要がある。
ここでは、それらについて、現実的な範囲で手短にまとめた。

### PHPレベルでのUTF-8

文字列の連結や変数への代入などの基本操作については、UTF-8だからといって何か特別なことをする必要はない。
しかし、大半の文字列関数（`strpos()` や `strlen()` など）については、そういうわけにはいかない。
これらの関数には、対応する関数として `mb_*` が用意されていることが多い。
たとえば `mb_strpos()` や `mb_strlen()` だ。
これらの関数をひとまとめにして、マルチバイト文字列関数と呼ぶ。
マルチバイト文字列関数は、Unicode文字列を扱えるように設計されている。

Unicode文字列を扱う場合は、常に `mb_*` 関数を使う必要がある。
たとえば、UTF-8文字列に対して `substr()` を使うと、その結果の中に文字化けした半角文字が含まれてしまう可能性がある。
この場合、マルチバイト文字列のときに使うべき関数は `mb_substr()` だ。

常に `mb_*` 関数を使うように覚えておくのが大変なところだ。
たとえ一か所でもそれを忘れてしまうと、それ以降の Unicode 文字列は壊れてしまう可能性がある。

そして、すべての文字列関数に `mb_*` 版があるわけではない。
自分が使いたい関数にマルチバイト版がないだって？
ご愁傷様。

さらに、すべてのPHPスクリプトの先頭（あるいは、グローバルにインクルードするファイルの先頭）で `mb_internal_encoding()`
関数を使わないといけないし、スクリプトの中でブラウザに出力するつもりなら、それだけではなく
`mb_http_output()` 関数も使わなければいけない。
すべてのスクリプトでエンコーディングを明示しておけば、後で悩まされることもなくなるだろう。

最後に、文字列を操作する関数の多くには、文字エンコーディングを指定するためのオプション引数が用意されている。
このオプションがある場合は、常に UTF-8 を明示しておくべきだ。
たとえば `htmlentities()` には文字エンコーディングを設定する引数があるので、
UTF-8 文字列を扱うなら常にそう指定しておかないといけない。

PHP 5.4.0 以降では、 `htmlentities()` や `htmlspecialchars()` のデフォルトエンコーディングが UTF-8 に変わった。

### データベースレベルでのUTF-8

PHP スクリプトから MySQL に接続する場合は、上で書いた注意をすべて守ったにもかかわらず UTF-8
文字列がそれ以外のエンコーディングで格納されてしまう可能性がある。

PHP から MySQL に渡す文字列を確実に UTF-8 として扱わせるには、データベースとテーブルの文字セットや照合順序の設定を、すべて
`utf8mb4` にしておく必要がある。さらに、PDO の接続文字列にも、文字セットとして `utf8mb4` を指定する。
詳細は以下のコードを参照すること。 _これ、試験に出るよ。_

UTF-8 を完全にサポートするには、文字セット `utf8mb4` を使わないといけない。 `utf8` はダメ！！！
その理由が知りたければ「あわせて読みたい」を参照すること。
Further Reading for why.

### ブラウザレベルでのUTF-8

`mb_http_output()` 関数を使えば、PHP スクリプトからブラウザへの出力が UTF-8 文字列になることを保証できる。
HTML の中では、 `<head>` タグに [charset の `<meta>` タグ](http://htmlpurifier.org/docs/enduser-utf8.html) を指定すればいい。

{% highlight php %}
<?php
// PHP に対して、今後このスクリプトの中では UTF-8 文字列を使うことを伝える
mb_internal_encoding('UTF-8');
 
// PHP に対して、ブラウザに UTF-8 で出力することを伝える
mb_http_output('UTF-8');
 
// UTF-8 のテスト用文字列
$string = 'Êl síla erin lû e-govaned vîn.';
 
// 何らかのマルチバイト関数で文字列を操作する。
// ここでは、デモの意味も込めて、非ASCII文字のところで文字列をカットしてみた。
$string = mb_substr($string, 0, 15);
 
// データベースに接続し、この文字列を格納する。
// このドキュメントにある PDO のサンプルを見れば、より詳しい情報がわかる。
// ここでの肝は、 `set names utf8mb4` コマンドだ。
$link = new \PDO(   
                    'mysql:host=your-hostname;dbname=your-db;charset=utf8mb4',
                    'your-username',
                    'your-password',
                    array(
                        \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
                        \PDO::ATTR_PERSISTENT => false
                    )
                );
 
// 変換した文字列を、UTF-8としてデータベースに格納する。
// DBとテーブルの文字セットや照合順序が、ちゃんとutf8mb4になっているかな？
$handle = $link->prepare('insert into ElvishSentences (Id, Body) values (?, ?)');
$handle->bindValue(1, 1, PDO::PARAM_INT);
$handle->bindValue(2, $string);
$handle->execute();
 
// 今格納したばかりの文字列を取り出して、きちんと格納できているかどうかを確かめる
$handle = $link->prepare('select * from ElvishSentences where Id = ?');
$handle->bindValue(1, 1, PDO::PARAM_INT);
$handle->execute();
 
// 結果をオブジェクトに代入して、後でHTMLの中で使う
$result = $handle->fetchAll(\PDO::FETCH_OBJ);
?><!doctype html>
<html>
    <head>
        <meta charset="UTF-8" />
        <title>UTF-8 テストページ</title>
    </head>
    <body>
        <?php
        foreach($result as $row){
            print($row->Body);  // 変換したUTF-8文字列が、ブラウザに正しく出力されるはず
        }
        ?>
    </body>
</html>
{% endhighlight %}

### あわせて読みたい

* [PHPマニュアル: 文字列演算子](http://php.net/manual/ja/language.operators.string.php)
* [PHPマニュアル: String関数](http://php.net/manual/ja/ref.strings.php)
    * [`strpos()`](http://php.net/manual/ja/function.strpos.php)
    * [`strlen()`](http://php.net/manual/ja/function.strlen.php)
    * [`substr()`](http://php.net/manual/ja/function.substr.php)
* [PHPマニュアル: マルチバイト文字列関数](http://php.net/manual/ja/ref.mbstring.php)
    * [`mb_strpos()`](http://php.net/manual/ja/function.mb-strpos.php)
    * [`mb_strlen()`](http://php.net/manual/ja/function.mb-strlen.php)
    * [`mb_substr()`](http://php.net/manual/ja/function.mb-substr.php)
    * [`mb_internal_encoding()`](http://php.net/manual/ja/function.mb-internal-encoding.php)
    * [`mb_http_output()`](http://php.net/manual/ja/function.mb-http-output.php)
    * [`htmlentities()`](http://php.net/manual/ja/function.htmlentities.php)
    * [`htmlspecialchars()`](http://www.php.net/manual/ja/function.htmlspecialchars.php)
* [PHP UTF-8 Cheatsheet](http://blog.loftdigital.com/blog/php-utf-8-cheatsheet)
* [Stack Overflow: What factors make PHP Unicode-incompatible?](http://stackoverflow.com/questions/571694/what-factors-make-php-unicode-incompatible)
* [Stack Overflow: Best practices in PHP and MySQL with international strings](http://stackoverflow.com/questions/140728/best-practices-in-php-and-mysql-with-international-strings)
* [How to support full Unicode in MySQL databases](http://mathiasbynens.be/notes/mysql-utf8mb4)
* [Brining Unicode to PHP with Portable UTF-8](http://www.sitepoint.com/bringing-unicode-to-php-with-portable-utf8/)
