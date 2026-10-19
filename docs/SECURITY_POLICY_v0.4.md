# KEIRIN AI 公開安全ポリシー v0.4

状態: 設計・審査基準。検査機構自体が稼働したことを意味しない。
目的: PublicのGitHubを利用しつつ、認証情報・私的個人情報・許諾条件に反するデータの公開を極力防ぐ。

## 脅威モデル

- 公開コミット/PR/フォーク/Actionsログ/成果物/スクリーンショットからの秘密漏洩。
- 誤入力・AI生成コードによるAPIキー、DBダンプ、実ユーザー情報の混入。
- 外部PRに紛れた悪意あるテスト・ワークフロー変更による秘密流出。
- 公開サイトから取得したレースデータ・動画の再配布権と学習権の誤判定。
- 公開バックエンドの権限不備、ログ出力、無料枠の誤課金。

## SEC-001: GitHub送信前ゲート

1. ファイル作成後、GitHub APIやgit pushなどを呼ぶ前に必ず非公開作業領域に保存し、送信予定ファイルを検査。
2. シークレット検出器（独立した専用スキャナー）、禁止ファイルパターン、PII/契約資料判定、バイナリ・アーカイブの中身/画像メタデータの確認、許諾台帳を全て利用。
3. 追加・変更ファイルだけでなく、意図しない生成物、画像、ログ、fixture、CI artifact指定も調べる。
4. 本物の秘密文字列はテスト用にも作らず、架空の非使用可能なサンプルだけで拒否試験。
5. どれかの検査が実行不能・高危険度・不明なら SEND_DENIED。検査をスキップしてPublishしてはいけない。
6. 合格した正確なバイト列のSHA256、対象ファイルリスト、検査バージョン、検査時刻を記録し、送信直前に一致を確認する。
7. iPhoneのGitHub Web UIでの直接ファイルアップロードは、この検査を迂回できるため正式フローにしない。

重要: GitHubのCIは送信後に走るため、漏洩防止の第一防壁ではない。

## SEC-002: GitHubに設定する防御

- GitHub Secret scanning / Push protection / Dependabot alertsを利用可能な範囲で有効化する。
- 専用の秘密検出器、PII候補検査、ライセンス検査をPRとmainで追加実行する。
- mainにrequired PR、独立レビュー、必須status checks、force push/削除禁止を設定する。
- セキュリティ検査に使うワークフロー・ポリシー変更は、オーナーの独立レビューを必要とする。
- セキュリティチェックがSKIPPED/NEUTRALになっても安心と判断しない。ワークフローと実行結果を検査して完了判定する。
- 管理者がブランチ保護を変更できる可能性がある。運用上は手動バイパスを禁止する。
- GitHubの標準検出には対象パターンやファイルサイズ等の限界があるので、100%保証と記載しない。

## SEC-003: GitHub Actionsの権限

- 外部のfork PRにSecretsを渡さない。通常のpull_requestから始め、GITHUB_TOKENは原則contents:readのみ。
- pull_request_target等の高権限文脈で未信頼PRをチェックアウトしてビルド・実行しない。
- 外部Actionのバージョン固定、権限最小化、依存関係確認、不要なartifact出力禁止を行う。
- 機密が必要なデプロイジョブは、保護されたmainとオーナー承認済み環境のみで実行。
- ログや失敗時のエラーページに秘密を出さず、メッセージはマスキングとリダクションを行う。

## SEC-004: データ分類

PUBLIC-CODE: 一般的コード・合成データは審査後公開可。
LICENSED-DATA: 出走表・オッズ・選手データも契約/利用条件によって制限される。許諾がある範囲のみ。
RESTRICTED: 映像原本・内部分析・契約・原本データは公開禁止。
PRIVATE: 私的氏名、住所、連絡先、位置、口座、実スクリーンショットは公開禁止。
SECRET: APIキー、パスワード、PAT、SSH鍵、DB接続資格情報は公開禁止。
公開できない情報はGitHub Issues/PR本文、AIの指示文、ログ・添付にも貼らない。

## SEC-005: 事故時の必須対応

1. 影響するキー・パスワード等を即時失効・ローテーション。アクセス遮断、関係ジョブ停止。
2. 公開コミット、履歴、fork、キャッシュ、Actionsログ/アーティファクト、パブリックなビルド出力を調査。
3. 履歴削除は副作用と限界を理解したうえで検討。既に第三者がコピーした情報は回収できない。
4. 個人情報や契約資料の場合、本人/関係事業者への連絡や法令・契約上の通知義務を確認。
5. 侵入原因に対応した再現テストを追加し、レビュー完了まで復旧を宣言しない。

## 必須の受入テスト

| ID | テスト | 合格条件 |
|---|---|---|
| SEC-T01 | 架空の秘密をコードに混入 | GitHub送信前に拒否 |
| SEC-T02 | 架空の氏名・連絡先をCSV/JSON/画像へ混入 | GitHub送信前に拒否 |
| SEC-T03 | 実行不能になったスキャナー | 検査を飛ばさず拒否 |
| SEC-T04 | バイナリや圧縮ファイルを追加 | 解析/許諾が確認できるまで拒否 |
| SEC-T05 | 規約未確認の動画URLから自動取得 | HTTP通信自体を行わない |
| SEC-T06 | ワークフロー変更で検査をスキップ | 審査が止まりmainへ統合されない |
| SEC-T07 | 外部PRがSecretsにアクセスしようとする | 権限を取得できない |
| SEC-T08 | わざと失敗ログに秘密を出そうとする | 公開ログに原文を残さずジョブ拒否 |
| SEC-T09 | 対象のSHAと送信内容が異なる | 改ざん/競合として送信拒否 |
| SEC-T10 | main保護なし/Required checkなし | Release gateで未完成判定 |

これらは**将来実証すべきテスト**で、現在PASSしたとは言わない。

## プロジェクト進行の前提

- Phase S（公開安全基盤）が完了するまで公開コード投入は最小限。設計文書で成功扱いしない。
- source_idごとの利用権限ゲートが完成するまで本物の自動収集をしない。
- 漏洩リスクが発見されたら安全性を優先し、開発速度やPR処理速度より停止を選ぶ。
- 本番予測と購入推奨はモデルの未知期間検証が完了するまでSHADOW_ONLY/NO_BET。

## 根拠資料

- https://docs.github.com/ja/code-security/reference/secret-security/secret-scanning-scope
- https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches
- https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target
- https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
