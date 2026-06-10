# 課題一覧

## 1. PostgreSQL固有演算子が整形で分断される

- ステータス: 完
- 種別: 既存
- 重要度: 高
- 問題: PostgreSQLのJSON/配列系演算子を含むSQLを整形すると、`->>`, `@>` などが分断され、実行できないSQLに変わる。
- 根拠: `BindSQLJavaLogs.html:1076` で `#>>`, `->>`, `@>`, `<@`, `?|`, `?&`, `&&` などの複合演算子を定義し、`BindSQLJavaLogs.html:2457` から `BindSQLJavaLogs.html:2475` で空白付与前に保護している。実行確認でも `data->>'name'` と `attrs @> ?` は分断されず、バインド後も演算子として残った。
- 現状: PostgreSQL複合演算子は整形時に保護されており、確認した範囲では既存の分断不具合は解消済み。
- 影響: 現在のコードでは、この既存課題によるSQL構文破壊は確認されない。
- 次の対応: なし。

## 2. 行コメント後の改行が失われてSQLの意味が変わる

- ステータス: 完
- 種別: 既存
- 重要度: 高
- 問題: `--` 行コメントを含むSQLを整形すると、コメント終端の改行が消え、後続トークンがコメント内に取り込まれる。
- 根拠: `BindSQLJavaLogs.html:1459` から `BindSQLJavaLogs.html:1461` で `--` コメントの保護範囲に終端改行を含めている。実行確認でも `select -- first column\n  * from users where id = ?` の `* from users` はコメント外に残った。
- 現状: 行コメントの改行は保護・復元されており、後続SQLがコメント化される挙動は再現しない。
- 影響: 現在のコードでは、この既存課題による出力SQLの意味変化は確認されない。
- 次の対応: なし。

## 3. 複数行のJavaログSQLをログ入力として解析できない

- ステータス: 完
- 種別: 既存
- 重要度: 高
- 問題: HibernateやMyBatisで整形済みSQLが複数行に出力されるログを貼り付けると、SQL本文とバインド行が正しく分離されない。
- 根拠: `BindSQLJavaLogs.html:1971` から `BindSQLJavaLogs.html:2009` のMyBatisパーサ、`BindSQLJavaLogs.html:2033` から `BindSQLJavaLogs.html:2071` のHibernateパーサはいずれもSQL継続行を収集する実装になっている。複数行の `==> Preparing:` と `Hibernate:` 入力でもバインド値が置換された。
- 現状: MyBatis/Hibernateの複数行SQLは、確認した範囲ではSQL本文として取り込まれる。
- 影響: 現在のコードでは、この既存課題によるログ文字列混入や未置換は確認されない。
- 次の対応: なし。

## 4. `BETWEEN` や `LIMIT` の位置バインドが `?` 演算子扱いされる

- ステータス: 完
- 種別: 既存
- 重要度: 高
- 問題: `BETWEEN ? AND ?`、`LIMIT ? OFFSET ?`、`LIKE ? ESCAPE ?` などの通常の位置バインドがPostgreSQLの `?` 演算子として誤判定され、置換されない。
- 根拠: `BindSQLJavaLogs.html:2265` から `BindSQLJavaLogs.html:2357` の直前キーワード集合と `?` 判定で、`BETWEEN`, `LIKE`, `ILIKE`, `LIMIT`, `OFFSET`, `FETCH`, `ESCAPE` などが通常の位置バインドとして扱われる。実行確認でも `BETWEEN ? AND ?`、`LIKE ? ESCAPE ?`、`LIMIT ? OFFSET ?` は位置バインドとして置換された。
- 現状: 通常句に続く `?` は演算子扱いされず、位置バインドとして処理されている。
- 影響: 現在のコードでは、この既存課題による未置換やバインド値ずれは確認されない。
- 次の対応: なし。

## 5. 空文字のMyBatis/Hibernateログバインド値が未置換になる

- ステータス: 完
- 種別: 既存
- 重要度: 中
- 問題: MyBatisやHibernateのログで空文字の文字列バインドが出ると、バインド行が `1 = ` になり、位置バインドとして解析されず `?` が未置換になる。
- 根拠: `BindSQLJavaLogs.html:1297` から `BindSQLJavaLogs.html:1298` で文字列系型を判定し、`BindSQLJavaLogs.html:1375` から `BindSQLJavaLogs.html:1380` でログパラメータの型を反映している。`BindSQLJavaLogs.html:1181` から `BindSQLJavaLogs.html:1185` で文字列型の空値を `''` にしている。実行確認でも `==> Parameters: (String)` とHibernateの空文字は `name = ''` に置換された。
- 現状: MyBatis/Hibernateの文字列型空値は `''` として扱われ、既存課題の入力では未置換にならない。
- 影響: 現在のコードでは、この既存課題による空文字バインドの未置換は確認されない。
- 次の対応: なし。

## 6. SQL本文中の `1 = 1` 行がバインド行として削除される

- ステータス: 完
- 種別: 既存
- 重要度: 高
- 問題: 手入力の複数行SQLで `WHERE` 直下などに `1 = 1` の行があると、SQL条件ではなく位置バインド定義として扱われ、SQL本文から削除される。
- 根拠: `BindSQLJavaLogs.html:1887` から `BindSQLJavaLogs.html:1894` で曖昧な位置バインド風述語を判定し、`BindSQLJavaLogs.html:1942` から `BindSQLJavaLogs.html:1956` で `WHERE`/`AND`/`OR` 直後の `1 = 1` をSQL本文に残している。実行確認でも `WHERE 1 = 1 AND id = 7` として出力された。
- 現状: 確認した既存ケースでは `1 = 1` 条件はSQL本文に保持される。
- 影響: 現在のコードでは、この既存課題による条件削除や `WHERE AND` 化は確認されない。
- 次の対応: なし。

## 7. Spring JDBCログのSQL抽出が `]` を含むSQLで途中終了する

- ステータス: 完
- 種別: 既存
- 重要度: 中
- 問題: Spring JDBCログの `Executing prepared SQL statement [...]` 内にPostgreSQL配列添字などの `]` が含まれると、SQL抽出が最初の `]` で止まり、SQL本文が途中で切れる。
- 根拠: `BindSQLJavaLogs.html:2074` から `BindSQLJavaLogs.html:2094` で、同一行末またはパラメータ行手前の最後の `]` をSQL終端として扱っている。実行確認でも `select arr[1] from t where id = ?` のSQLは途中で切れず、`id = 7` まで出力された。
- 現状: Spring JDBCのSQL本文中に `]` がある既存ケースは解消済み。
- 影響: 現在のコードでは、この既存課題によるSQL本文の途中切断は確認されない。
- 次の対応: なし。

## 8. Spring JDBCのパラメータ値解析が値の `]` と文字列型を保持しない

- ステータス: 完
- 種別: 既存
- 重要度: 高
- 問題: Spring JDBCログの `parameter value [...]` で、値そのものに `]` が含まれると最初の `]` で切り詰められる。また `value class [java.lang.String]` を読んでいないため、`001` や `true` のような文字列値が数値・真偽値として出力される。
- 根拠: `BindSQLJavaLogs.html:2097` から `BindSQLJavaLogs.html:2123` でSpring JDBCのパラメータ行を構造的に解析し、`value class` を `BindSQLJavaLogs.html:2131` から `BindSQLJavaLogs.html:2136` で `_formatLogPositionalBindLine` に渡している。実行確認でも `parameter value [abc]def], value class [java.lang.String]` は `'abc]def'`、`parameter value [001], value class [java.lang.String]` は `'001'` と出力された。
- 現状: Spring JDBCのパラメータ値に `]` を含む文字列や、数値・真偽値に見える文字列は、文字列型として保持される。
- 影響: 現在のコードでは、この既存課題による値の切り詰めや文字列型の喪失は確認されない。
- 次の対応: なし。

## 9. 明示的なバインドセクションでもSQLキーワード名の名前付きバインドを解析できない

- ステータス: 完
- 種別: 既存
- 重要度: 中
- 問題: `params:` などの明示的なバインドセクション内でも、`limit = 10` のように名前がSQLキーワード扱いされるバインド行がSQL本文へ戻され、`:limit` が未置換になる。
- 根拠: `BindSQLJavaLogs.html:1861` から `BindSQLJavaLogs.html:1884` で明示的なバインドセクション中のbare nameはSQLキーワード名でもバインド定義として許可している。また `BindSQLJavaLogs.html:2181` から `BindSQLJavaLogs.html:2195` で、バインドセクション開始後はSQLプレフィックス判定よりバインド定義判定を優先している。実行確認でも `params:\nuser = 42\nlimit = 10\noffset = 5\nsql = abc` により `:user`, `:limit`, `:offset`, `:sql` がすべて置換された。
- 現状: `limit`, `offset`, `sql` などの名前付きバインドは、明示的なバインドセクション内で解析できる。
- 影響: 現在のコードでは、この既存課題による未置換やバインド行のSQL本文混入は確認されない。
- 次の対応: なし。

## 10. `sql=[...] params=[...]` 形式のパラメータ値がカンマ入り構造で分割される

- ステータス: 完
- 種別: 既存
- 重要度: 中
- 問題: `sql=[...] params=[...]` 形式で `ARRAY[1,2]` のようなカンマを含む値を渡すと、値の途中のカンマで別パラメータとして分割され、誤った値に置換される。
- 根拠: `BindSQLJavaLogs.html:1196` から `BindSQLJavaLogs.html:1270` の `_splitParameterEntries` で、引用符に加えて角括弧と丸括弧のネスト深度を追跡し、トップレベルのカンマだけで分割している。実行確認でも `params=[1 = ARRAY[1,2], 2 = 3]` は途中分割されず、2件のパラメータとして扱われた。
- 現状: パラメータブロック形式で、角括弧・丸括弧内のカンマはパラメータ区切りとして扱われない。
- 影響: 現在のコードでは、この既存課題によるカンマ入り値の途中分割は確認されない。
- 次の対応: なし。

## 11. PostgreSQL配列添字内のプレースホルダが置換されない

- ステータス: 未
- 種別: 新規
- 重要度: 高
- 問題: PostgreSQLの配列添字などで `arr[?]` や `data[:idx]` のように角括弧内へプレースホルダを書くと、バインド定義が検出されず、プレースホルダも置換されない。
- 根拠: `BindSQLJavaLogs.html:1474` から `BindSQLJavaLogs.html:1476` で角括弧全体を保護セグメントとして扱っている。さらに `BindSQLJavaLogs.html:1907` から `BindSQLJavaLogs.html:1924` の暗黙バインド検出と、`BindSQLJavaLogs.html:2372` から `BindSQLJavaLogs.html:2391` の置換処理はいずれも保護後のSQLを対象にする。実行確認では `select arr[?] from t\n1 = 2` が `_bindLines: []` になり、出力も `arr[?]` と `t 1 = 2` を含んだ。
- 現状: 角括弧内の位置バインド・名前付きバインドは、暗黙バインド検出と置換の対象外になっている。
- 影響: 配列添字をバインドするSQLで、未置換SQLやバインド行混入SQLが生成され、PostgreSQLで実行できない可能性が高い。
- 次の対応: 角括弧保護を引用識別子用途に限定する、または角括弧内の `?` / `:name` を検出・置換対象にする。

## 12. ヘッダーなしの名前付きバインド行がSQL本文に混入する

- ステータス: 未
- 種別: 新規
- 重要度: 中
- 問題: `:id` を含むSQLの直後に `id = 1` のようなbare nameのバインド行を置くと、`params:` ヘッダーや `:id = 1` のコロン付き表記がない限りバインド定義として扱われず、SQL本文に混入する。
- 根拠: `BindSQLJavaLogs.html:1861` から `BindSQLJavaLogs.html:1878` でbare nameのバインド定義は `_allowBareName` が真のときだけ許可されるが、`BindSQLJavaLogs.html:1942` から `BindSQLJavaLogs.html:1956` ではバインドセクション開始前に `_allowBareName` が真にならない。実行確認では `select * from t where id=:id\nid=1` が `_bindLines: []` になり、`:id` 未置換と `id = 1` のSQL本文混入が発生した。
- 現状: ヘッダーなしの名前付きバインドは、先頭に `:` を付けた場合か、明示的な `params:` セクション内だけ正常に処理される。
- 影響: READMEで名前付きバインド対応をうたっている一方、自然な貼り付け形式で未置換や無効SQLが発生し、ユーザー操作で想定外の出力になる。
- 次の対応: 未解決の名前付きプレースホルダがあり、残り行がバインド定義だけの場合は、SQL代入行や述語誤判定を避けつつbare nameもバインド行として扱う。

## 13. 指数表記の数値バインドが文字列リテラル化される

- ステータス: 未
- 種別: 新規
- 重要度: 中
- 問題: `1e-3` や `1E+6` のような指数表記の数値バインドが数値リテラルとして認識されず、`'1e-3'` や `'1E+6'` の文字列リテラルとして出力される。
- 根拠: `BindSQLJavaLogs.html:1173` から `BindSQLJavaLogs.html:1177` と `BindSQLJavaLogs.html:1854` の数値判定正規表現が指数部を許容していない。`BindSQLJavaLogs.html:1375` から `BindSQLJavaLogs.html:1380` のログパラメータ整形も同じ判定に依存する。実行確認では手入力の `1 = 1e-3`、MyBatisの `1E+6(BigDecimal)`、Hibernateの `[NUMERIC] - [1E+6]` がいずれも引用付きで出力された。
- 現状: 整数・小数は数値として扱われるが、指数表記の数値は文字列として扱われる。
- 影響: 数値列や数値関数に対して型不一致、暗黙キャスト依存、実行計画や比較結果の差異につながる可能性がある。
- 次の対応: 数値リテラル判定に指数表記を追加し、手入力・MyBatis・Hibernate・Spring JDBCの数値ログで引用されないことを確認する。
