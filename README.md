# My-Test-Repo
**「26種類すべてについて、全発生条件でRFCどおりの番号を送信する」とは結論できません。具体的な番号選択の相違が見つかりました。**

OpenJDK公式の **21.0.12+8**（コミット `9de4f68c88a0a1510373f291d1a95b1f6b0db8c8`）を固定して調査し、関連部分を21 GAとも比較しました。

- [日本語の詳細一覧・条件別評価・全参照箇所の索引（Markdown）](</C:/Users/ryuuji/Documents/Codex/2026-10-14/referenced-chatgpt-conversation-this-is-an/outputs/OpenJDK21_RFC9846_TLS_Error_Alerts_調査一覧.md>)
- [根拠URL・ソース前後の文脈付き索引（CSV、484件）](</C:/Users/ryuuji/Documents/Codex/2026-10-14/referenced-chatgpt-conversation-this-is-an/outputs/OpenJDK21_TLS_alert_source_index.csv>)

以下が26種類の要約です。発生条件は [RFC 9846 §6.2](https://www.ietf.org/ietf-ftp/rfc/rfc9846.html#section-6.2)、実装上の番号定義は [Alert.java](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/Alert.java#L42) に基づきます。**117以外の25種類は定義され、定義番号は一致しています。**

| 番号・アラート | 送信条件とRFCとの一致に関する静的調査結果 |
|---|---|
| 10 `unexpected_message` | 状態・順序違反に送信経路あり。ただし復号後のcontent type欠落は40／20へ変換され、RFCが指定する10と異なる。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/SSLCipher.java#L1928) |
| 20 `bad_record_mac` | AEADタグ不一致は20。ただし短過ぎる暗号文などの`BadPaddingException`は、`handshakeContext`があると40になる。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/SSLTransport.java#L122) |
| 22 `record_overflow` | **定義のみで送信側参照なし。** 受信長検査は大きな互換上限を使用し、超過時も22を選ばない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/SSLSocketInputRecord.java#L225) |
| 40 `handshake_failure` | パラメーター交渉失敗に経路あり。ただしCertificateVerify署名不一致にも40を使用し、指定される51と異なる。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateVerify.java#L1014) |
| 42 `bad_certificate` | 証明書パース失敗などに経路あり。サーバー側のクライアント証明書検証失敗は別途46へ集約される。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateMessage.java#L1187) |
| 43 `unsupported_certificate` | アルゴリズム制約違反の例外変換で選択。あらゆる未対応証明書種別を43にする実装ではない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateMessage.java#L1325) |
| 44 `certificate_revoked` | クライアント側で認識した失効原因を44へ変換。stapling有効時は113、クライアント証明書の検証失敗は46になり得る。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateMessage.java#L1312) |
| 45 `certificate_expired` | `EXPIRED`は45。一方、現在まだ有効でない`NOT_YET_VALID`は42であり、全有効期間エラーが45にはならない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateMessage.java#L1320) |
| 46 `certificate_unknown` | その他の証明書エラーに使用。サーバー側では具体的な失効・期限等を含む検証例外を一律46に集約する。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateMessage.java#L1231) |
| 47 `illegal_parameter` | 意味上の不整合に経路あり。ただしKeyUpdateの長さ不正など、RFC上50に対応する構文エラーにも使用。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/KeyUpdate.java#L80) |
| 48 `unknown_ca` | **定義のみで送信側参照なし。** 証明書例外変換にもCA不明を48へ変換する分岐がない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateMessage.java#L1303) |
| 49 `access_denied` | **定義のみで送信側参照なし。** アクセス制御による拒否を49へ変換する経路は確認できない。[定義](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/Alert.java#L57) |
| 50 `decode_error` | 長さ不整合・バッファ不足などに経路あり。ただし明示的な長さ検査で47を使う箇所もあり、構文エラー全体では統一されていない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/HandshakeContext.java#L482) |
| 51 `decrypt_error` | Finished不一致は51。しかしCertificateVerify不一致は40、PSK binder不一致は47。[Finished](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/Finished.java#L123)、[binder](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/PreSharedKeyExtension.java#L565) |
| 70 `protocol_version` | 対応版不一致に経路あり。ただし未提示版の選択に対して47を要求する条件でも、70へ進む分岐がある。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/ServerHello.java#L965) |
| 71 `insufficient_security` | TLS 1.3ではクライアントがサーバーのkey_shareを制約違反として拒否する経路。§6.2のサーバーによる拒否という記述と役割が異なる。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/KeyShareExtension.java#L672) |
| 80 `internal_error` | 鍵導出等の内部例外に多数の経路あり。ただしメモリ不足など、全内部障害で80を送る保証はない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/PreSharedKeyExtension.java#L797) |
| 86 `inappropriate_fallback` | **定義のみで送信側参照なし。** `TLS_FALLBACK_SCSV`は値が登録されているが、86を選ぶ検出処理は見つからない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CipherSuite.java#L693) |
| 109 `missing_extension` | 必須拡張欠落に複数の経路あり。ただしPSKがあるのにPSK modesがない場合は47へ集約する。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/PreSharedKeyExtension.java#L352) |
| 110 `unsupported_extension` | 共通拡張処理に経路あり。ただし未要求ALPN応答には110ではなく10を使用する。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/AlpnExtension.java#L475) |
| 112 `unrecognized_name` | 設定されたSNIMatcherに一致しない場合に112。Matcher未設定やSNI欠落を自動的に112とはしない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/ServerNameExtension.java#L305) |
| 113 `bad_certificate_status_response` | クライアント側のOCSP関連例外変換に経路あり。一方、サーバー側の応答生成失敗にも113を使用している。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertStatusExtension.java#L1168) |
| 115 `unknown_psk_identity` | **定義のみで送信側参照なし。** 不一致なら再開を断念する経路がある。115の送信はOPTIONALなので、未送信だけでは違反ではない。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/PreSharedKeyExtension.java#L416) |
| 116 `certificate_required` | クライアント認証必須かつ空のCertificateで116。**21 GAでは同条件が42だったため、更新版との区別が必要。** [根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateMessage.java#L1126) |
| 117 `general_error` | **未定義・送信未実装。** RFC 9846で追加された番号であり、既存の80とは異なる。[定義一覧](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/Alert.java#L42) |
| 120 `no_application_protocol` | 設定されたALPNに共通プロトコルがない場合などに120。該当設定下の送信経路はRFC条件に対応する。[根拠](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/AlpnExtension.java#L322) |

**送信呼出しがあっても、実際の送信・到達は保証されません。** 共通の`fatal()`処理には、接続状態・出力側閉鎖・受信fatalへの応答抑止などの条件があります。SSLEngineではアラートをキューに入れ、その後の出力転送をアプリケーションが担います。[共通処理](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/TransportContext.java#L412)、[SSLEngine出力](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/SSLEngineOutputRecord.java#L74)

今回網羅したのは、**26種類の一覧と、対象133ファイル内の明示参照484件**です。件数にはTLS 1.2以前・DTLSの共通ソースも含みます。**ビルド、実接続、異常入力テスト、パケットキャプチャは未実施**であり、全入力・全例外・全実行経路の適合性を証明した調査ではありません。詳細資料では、任意送信の省略と、RFCが特定番号を要求する条件での相違を分けています。
