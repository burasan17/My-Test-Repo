# My-Test-Repo

結論：いいえ。ECPGで-r no_indicatorを指定していても、全スペースのホスト変数をINSERT文の中でRTRIMした場合、NULL扱いにはなりません。

PostgreSQLの公式仕様とECPGのソースコードを確認しました。

1. 具体的な動作

例えば、次のECPGプログラムです。

EXEC SQL BEGIN DECLARE SECTION;
    char val[6] = "     ";
EXEC SQL END DECLARE SECTION;
EXEC SQL INSERT INTO TABLE_B (COL1)
    VALUES (RTRIM(:val));

-r no_indicatorを指定している場合、次の順序で処理されます。

順序	処理	値
①	C言語ホスト変数 val	’ ’
②	ECPGがNULL判定	NULLではない
③	PostgreSQLにパラメータを渡す	’ ’
④	PostgreSQLがRTRIM実行	‘’
⑤	INSERTされる値	‘’（空文字列）

最終的な結果はNULLではなく、空文字列です。

したがって、登録先がVARCHARやTEXTのNOT NULLカラムなら、NOT NULL制約違反にはなりません。

登録先がCHAR(n)なら、空文字列が空白埋めされて格納される点にも注意してください。

2. なぜno_indicatorが作用しないのか

-r no_indicatorのNULL判定は、SQLを実行する前、ECPGがホスト変数を入力パラメータに変換する段階で行われます。

ECPGの内部関数ECPGis_noind_null()では、文字配列について次のような判定をしています。

if (*((const char *) ptr) == '\0')
    return true;

つまり、ホスト変数の先頭バイトが\0かどうかを判定しています。

全スペースの場合、先頭バイトは0x20なので、NULLと判定されません。

その後、PostgreSQL側でRTRIMによって空文字列になっても、ECPGによるNULL判定は再実行されません。

これはECPGのソースコードで確認できます。

3. NULLになる場合との比較

INSERT文	登録される値
VALUES (:val) ※全スペース	全スペース
VALUES (RTRIM(:val)) ※全スペース	空文字列
VALUES (:val) ※先頭が\0	NULL
VALUES (RTRIM(:val)) ※先頭が\0	NULL

※インジケータ変数なし、-r no_indicator指定、通常の文字列ホスト変数を前提とします。

4. 今回の障害について

ここまでの情報から考えると、ホスト変数が本当に全スペースで、INSERT文内でRTRIM(:val)しているだけなら、-r no_indicatorが原因でNULL制約違反になることはありません。

もしNOT NULL制約違反が発生しているなら、疑うべきは次の2点です。

* INSERT直前のホスト変数が全スペースではなく、先頭バイトが\0になっている。
* インジケータ変数や別のSQL式によって、NULLが渡されている。

特に今回は、INSERT直前のホスト変数の先頭バイトを16進数で確認するのが有効です。

0x20ならスペース、0x00ならno_indicatorによってNULL判定される可能性があります。

公式資料：PostgreSQL ECPG — ホスト変数とNULLインジケータ⁠￼。
