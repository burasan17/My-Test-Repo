# My-Test-Repo

はい。ただし、RTRIMの結果が空文字列になった後、ECPGのホスト変数を経由するかどうかで動作が異なります。

PostgreSQL公式マニュアルとECPGのソースコードで確認できました。

1. 動作の違い

処理パターン	結果
SQL内でRTRIM(’ ’)	空文字列’’
SQL内でRTRIMして直接INSERT	空文字列として登録
RTRIM結果をECPGのchar[]変数に取得	空文字列（先頭が\0）
その変数を-r no_indicatorで再度INSERT	NULLとして送信

つまり、-r no_indicatorはPostgreSQLのSQL関数には作用せず、ECPGがホスト変数をSQLの入力パラメータへ変換する際に作用します。

2. 具体的な例

ECPGを以下のオプションでコンパイルした場合です。

ecpg -r no_indicator sample.pgc

次の処理を実行します。

EXEC SQL BEGIN DECLARE SECTION;
    char val[10];
EXEC SQL END DECLARE SECTION;
EXEC SQL SELECT RTRIM('     ')
    INTO :val;
EXEC SQL INSERT INTO TABLE_B (COL1)
    VALUES (:val);

実際には次のように処理されます。

* PostgreSQLのRTRIMが5文字のスペースを空文字列に変換。
* ECPGがホスト変数valに空文字列を取得。
* INSERT時にECPGのECPGis_noind_null()が実行される。
* val[0] == '\0'のためNULLと判定される。
* PostgreSQLにNULLが送信される。
* COL1がNOT NULLなら制約違反になる。

ソースコードでは、ECPGis_noind_null()が文字列の先頭バイトを調べ、\0ならNULLと判定します。VARCHAR型についても同様の判定が実装されています。

3. 逆にNULLにならないパターン

次のようにSQL内で完結する場合です。

INSERT INTO TABLE_B (COL1)
VALUES (RTRIM('     '));

この場合、ECPGのホスト変数を経由しないため、-r no_indicatorの影響を受けません。

結果は空文字列''であり、NOT NULL制約違反にはなりません。

4. 今回の障害との関連

今回の現象について、次の条件が成立するなら原因として非常に有力です。

* ECPGのコンパイルオプションが-r no_indicator
* RTRIMの結果が全スペースから空文字列になる
* その結果を一度C言語のホスト変数に取得する
* インジケータを指定せず、別のSQLでINSERTする

この条件では、PostgreSQL上ではNULLではなかった空文字列が、ECPGを経由することでNULLとして再登録されます。

これは推測だけではなく、ECPGの公式仕様・実装と整合する動作です。

今回の調査では、RTRIMを実行してからINSERTするまでの間にホスト変数を経由しているかを確認するのが最も重要です。
