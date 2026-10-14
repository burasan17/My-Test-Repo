# My-Test-Repo

## 結論：OpenJDK 21はRFC 9846 §6.2の全エラーアラートに完全対応しているわけではありません

OpenJDK 21の公式ソースコードをRFC 9846と照合した結果、まず以下の点が確認できました。

- RFC 9846 §6.2には、26種類のエラーアラートが定義されています。
- OpenJDK 21のSunJSSEには、そのうち25種類の定数が実装されています。
- `general_error`（Alert 117）が未定義です。
- さらに、既知のエラーアラートを`warning`レベルで受信した場合の処理にも、RFCとの相違が認められます。

したがって、単純に「RFC 9846 §6.2に完全準拠している」と評価することはできません。

ただし、Alert 117が未定義であることと、TLS 1.3の通信が正常にできないことは別問題です。未認識のAlertに対しても接続を終了させる共通処理は実装されています。

### 比較した一次資料

- RFC 9846 §6.2 — Error Alerts（IETF公式）
- OpenJDK 21 GA — Alert.java
- OpenJDK 21更新版開発ブランチ — Alert.java

`Alert.java`については、OpenJDK 21 GAと更新版開発ブランチの双方で、Alert 117が存在しないことを確認しました。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub

+2



## 1. RFC 9846 §6.2とOpenJDK 21の全26種類を比較

まず、Alert番号の定義有無を全件比較した結果です。

| 番号  | RFC 9846のAlert                    | OpenJDK 21の定義 |
| --- | --------------------------------- | ------------- |
| 10  | `unexpected_message`              | 〇             |
| 20  | `bad_record_mac`                  | 〇             |
| 22  | `record_overflow`                 | 〇             |
| 40  | `handshake_failure`               | 〇             |
| 42  | `bad_certificate`                 | 〇             |
| 43  | `unsupported_certificate`         | 〇             |
| 44  | `certificate_revoked`             | 〇             |
| 45  | `certificate_expired`             | 〇             |
| 46  | `certificate_unknown`             | 〇             |
| 47  | `illegal_parameter`               | 〇             |
| 48  | `unknown_ca`                      | 〇             |
| 49  | `access_denied`                   | 〇             |
| 50  | `decode_error`                    | 〇             |
| 51  | `decrypt_error`                   | 〇             |
| 70  | `protocol_version`                | 〇             |
| 71  | `insufficient_security`           | 〇             |
| 80  | `internal_error`                  | 〇             |
| 86  | `inappropriate_fallback`          | 〇             |
| 109 | `missing_extension`               | 〇             |
| 110 | `unsupported_extension`           | 〇             |
| 112 | `unrecognized_name`               | 〇             |
| 113 | `bad_certificate_status_response` | 〇             |
| 115 | `unknown_psk_identity`            | 〇             |
| 116 | `certificate_required`            | 〇             |
| 117 | `general_error`                   | ×             |
| 120 | `no_application_protocol`         | 〇             |

〇はAlert.javaで定数と番号が定義されていることを意味します。各エラー条件の検出や、そのAlertの実際の送信まで検証済みという意味ではありません。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub

+2

### 未定義のAlert 117について

`general_error`はRFC 9846で追加された汎用エラーです。

特定のエラーに分類できない場合や、詳細なエラー理由を隠したい場合に利用できます。

OpenJDK 21には以下の並びがあります。

```
UNKNOWN_PSK_IDENTITY ((byte)115, "unknown_psk_identity", true),
CERTIFICATE_REQUIRED ((byte)116, "certificate_required", true),
NO_APPLICATION_PROTOCOL ((byte)120, "no_application_protocol", true);
```

つまり117が存在しません。

もっとも、RFCは可能な場合には具体的なエラーを使用することを推奨しているため、117を自ら送信しないことだけで仕様違反とは断定できません。

## 2. エラーアラートの受信処理にRFCとの相違がある

こちらの方が実装上、重要です。

RFC 9846 §6では、§6.2に列挙されたエラーアラートは、受信したAlertLevelにかかわらずエラーとして扱わなければならないと規定しています。

[image](https://www.google.com/s2/favicons?domain=https://datatracker.ietf.org\&sz=32)

IETF Datatracker



ところが、OpenJDK 21の`Alert.java`には次の処理があります。

```
} else if ((level == Level.WARNING) && (alert != null)) {
    if (alert.handshakeOnly && (tc.handshakeContext != null)) {
        // ハンドシェイク中の警告を処理
    }
    // Otherwise, ignore the warning
}
```

説明用に内部の詳細を省略。公式ソースの該当箇所&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



例えば、TLS 1.3の接続確立後に相手から次のAlertを受信した場合です。

```
AlertLevel       = warning (1)
AlertDescription = bad_record_mac (20)
```

RFC 9846では、Alert 20はレベルにかかわらずエラーとして扱う必要があります。

一方、OpenJDK 21のコードでは、`bad_record_mac`は既知のAlertで、`handshakeOnly`が`false`なので、この分岐では無視されることになります。

この受信条件については、RFC 9846の要求とOpenJDK 21の実装が一致していないと判断できます。

なお、これは相手が規定どおりの`fatal`ではなく、不適切な`warning`レベルでエラーアラートを送った場合の話です。通常の相手側実装では発生しにくい条件ですが、規格適合性を評価するうえでは無視できません。

## 3. エラー番号の定義と実際の送信処理にも違いがある

実際のOpenJDK 21では、エラー条件に応じて`TransportContext.fatal()`を呼び出し、Alertを送信します。

TransportContext.java — fatal()

ここでは、受信処理の終了、セッションの無効化、Fatal Alert送信、トランスポートの終了が実装されています。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



具体的な送信処理も確認しました。

| 発生条件                   | OpenJDK 21で確認した処理                  |
| ---------------------- | ---------------------------------- |
| ALPNで共通プロトコルがない        | `NO_APPLICATION_PROTOCOL`を送信する経路あり |
| SNI名が一致しない             | `UNRECOGNIZED_NAME`を送信する経路あり       |
| 証明書失効                  | `CERTIFICATE_REVOKED`等へ振り分ける処理あり   |
| 証明書期限切れ                | `CERTIFICATE_EXPIRED`へ振り分ける処理あり    |
| TLS 1.3で必須のクライアント証明書が空 | `BAD_CERTIFICATE`を送信               |

これらは、AlpnExtension.java、ServerNameExtension.java、CertificateMessage.javaから確認できます。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub

+3



ここで注意すべき実装差分がさらにあります。

RFC 9846 §4.5.1.3では、サーバーが空のCertificateメッセージを送信した場合、クライアントは`decode_error`でハンドシェイクを中止しなければならないと規定しています。

ところがOpenJDK 21のTLS 1.3用`CertificateMessage.java`では、この条件に`BAD_CERTIFICATE`を使用しています。

[image](https://www.google.com/s2/favicons?domain=https://datatracker.ietf.org\&sz=32)

IETF Datatracker

+1



これもエラーアラートの定数が存在するだけでは、RFCで要求された送信条件まで一致しているとは限らないことを示す具体例です。

## 4. 未定義のAlert 117を受信するとどうなるか

OpenJDK 21の`AlertConsumer.consume()`では、次の処理が確認できます。

```
if (alert == null) {
    alert = Alert.UNEXPECTED_MESSAGE;
    diagnostic = "Unknown alert description (" + am.id + ")";
}

throw tc.fatal(alert, diagnostic, true, null);
```

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub

したがって、相手から`general_error`（117）を受信した場合、OpenJDK 21はその名前を認識しませんが、未知のAlertとして致命的エラー処理に進み、接続を終了させます。

つまり、

- `general_error`という名前で識別する機能：未実装
- 未知のエラーアラートを受信して接続終了する機能：実装済み

という違いがあります。

## 5. 今回のAlert 90との関係

ここは切り離して考える必要があります。

Alert 90（`user_canceled`）はRFC 9846 §6.2ではなく、§6.1のClosure Alertsに分類されています。

したがって、今回確認した「エラーアラートを致命的エラーとして扱う」という規定は、Alert 90には適用されません。

OpenJDK 21の`AlertConsumer`にも、Alert 90を`warning`として受信すると`peerUserCanceled=true`を設定する専用の分岐が実装されています。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



これは、先方がAlert 90を一律に通信エラーと判定している問題を考える際に、重要な区別です。

## 最終評価

| 評価対象                          | 結果                |
| ----------------------------- | ----------------- |
| RFC 9846 §6.2の26種類のAlert番号の定義 | 25/26種類           |
| Fatal Alertの送信・受信・接続終了の共通処理   | 実装済み              |
| 未知のAlertに対する終了処理              | 実装済み              |
| Error Alertをwarningで受信した場合    | RFCとの差異あり         |
| 全26種類について、全発生条件で正しい番号を送信するか   | 完全な検証は未了。具体的な差異あり |
| RFC 9846への完全準拠                | 確認できず。実装差分が存在     |

今回の調査対象はOpenJDK 21のGAソースと21u更新版開発ソースです。RHELに導入されたRed Hatビルドについては、正確なパッケージバージョンと適用パッチの照合が別途必要です。

実務的な結論として、OpenJDK 21はTLS 1.3の主要なエラー処理を実装していますが、RFC 9846 §6.2を全項目・全条件で厳密に満たしているとは言えません。

そして、今回のJBossと先方APIサーバーの相互接続問題については、§6.2の実装差分よりも、§6.1の`user_canceled`を先方がClosure Alertとして扱っているかを優先して確認するのが妥当です。
