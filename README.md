# My-Test-Repo
**「RFC違反の判定」列を追加しました。** 根拠条項と判定理由も、各項目に追記しています。

- [更新した詳細一覧（Markdown）](</C:/Users/ryuuji/Documents/Codex/2026-10-14/referenced-chatgpt-conversation-this-is-an/outputs/OpenJDK21_RFC9846_TLS_Error_Alerts_調査一覧.md>)
- [違反判定列付きの26種類一覧（CSV）](</C:/Users/ryuuji/Documents/Codex/2026-10-14/referenced-chatgpt-conversation-this-is-an/outputs/OpenJDK21_TLS_alerts_RFC違反判定.csv>)

判定対象は **OpenJDK 21.0.12+8の送信処理**です。「違反あり」は具体的な条件でMUSTとの不一致があるという静的判定です。SHOULDには合理的な例外があり得るため、理由を評価していないものは断定を保留しています。[RFC 2119 §3](https://www.rfc-editor.org/rfc/rfc2119.html#section-3)

| 番号・アラート | RFC違反の判定 | 主な判定理由 |
|---|---|---|
| 10 `unexpected_message` | **違反あり** | content type欠落時、§5.4指定の10ではなく40／20を選択。 |
| 20 `bad_record_mac` | **違反あり** | 短過ぎるAEAD暗号文の処理で、§5.2指定の20ではなく40になる分岐あり。 |
| 22 `record_overflow` | **違反あり** | 長さ超過に22を要求する§5.1／§5.2に対し、22を選ばず10へ進む分岐あり。 |
| 40 `handshake_failure` | **違反あり** | CertificateVerify不一致に40を使用。§4.5.2は51を指定。 |
| 42 `bad_certificate` | **条件付き違反** | MD5を理由にクライアント証明書を拒否する場合、§4.5.1.3指定の42ではなく46へ集約。 |
| 43 `unsupported_certificate` | 違反未確認 | 確認した43の使用や、任意の証明書形式の未提供だけでは違反といえない。 |
| 44 `certificate_revoked` | 断定保留（SHOULDとの差） | クライアント証明書の失効を46へ集約。常に44を要求する個別MUSTとは区別。 |
| 45 `certificate_expired` | 断定保留（SHOULDとの差） | `NOT_YET_VALID`を42へ変換するなど、45の発生条件との相違あり。 |
| 46 `certificate_unknown` | **条件付き違反** | MD5拒否まで46へ集約する場合、42指定に反する。42行と同じ事象。 |
| 47 `illegal_parameter` | **違反あり** | 長さ不正などの構文エラーにも47を使用。§6は50を指定。 |
| 48 `unknown_ca` | 断定保留（SHOULDとの差） | CA不明を48にせず46へ集約。具体的番号の推奨との相違。 |
| 49 `access_denied` | 未実装だけでは違反といえない | 対応するアクセス制御・拒否条件の実施を確認しない限り断定できない。 |
| 50 `decode_error` | **違反あり** | §6で50が指定される構文エラーを47にする分岐あり。 |
| 51 `decrypt_error` | **違反あり** | CertificateVerify不一致で、§4.5.2指定の51ではなく40を使用。 |
| 70 `protocol_version` | **違反あり** | `supported_versions`の未提示版に、§4.3.1指定の47ではなく70を選択。 |
| 71 `insufficient_security` | 断定保留（SHOULDとの差） | クライアント側の制約違反に使用し、§6.2の条件・役割と異なる。 |
| 80 `internal_error` | 違反未確認 | 全内部障害で必ず80を送る義務は確認していない。 |
| 86 `inappropriate_fallback` | **条件付き違反** | RFC 7507 §3の適用条件が成立すると86が必要だが、検出・送信経路なし。 |
| 109 `missing_extension` | 断定保留（SHOULDとの差） | PSK modes欠落で47を使用。§4.3.9の中止MUSTは満たすが、番号選択に相違。 |
| 110 `unsupported_extension` | **違反あり** | 未要求ALPN応答に、§4.3指定の110ではなく10を使用。 |
| 112 `unrecognized_name` | 違反未確認 | 設定されたSNIMatcherの不一致で112を送る経路は整合。 |
| 113 `bad_certificate_status_response` | 断定保留（SHOULDとの差） | サーバー側の応答生成失敗にも使用し、§6.2の条件・役割と異なる。 |
| 115 `unknown_psk_identity` | **省略は違反でない** | §6.2で送信がOPTIONALと明記されている。 |
| 116 `certificate_required` | 違反未確認 | 21.0.12の空クライアント証明書拒否は116で整合。21 GAの同分岐の42は違反。 |
| 117 `general_error` | **未定義だけでは違反といえない** | 必ず117を実装・送信するという明示的義務は確認していない。 |
| 120 `no_application_protocol` | 違反未確認 | 確認したALPN交渉失敗時の120送信は整合。 |

例えばCertificateVerifyについては、[RFC §4.5.2が51を要求](https://www.ietf.org/ietf-ftp/rfc/rfc9846.html#section-4.5.2)する一方、[実装は40を選択](https://github.com/openjdk/jdk21u/blob/9de4f68c88a0a1510373f291d1a95b1f6b0db8c8/src/java.base/share/classes/sun/security/ssl/CertificateVerify.java#L1014)しているため、「違反あり」と判定できます。

**実行テストは未実施です。** 「違反未確認」は全条件での適合を意味しません。また、同じ事象を期待番号・実装番号の両行に記載しているため、違反行数を独立した不具合件数として合算できません。
