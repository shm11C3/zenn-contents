## この記事でやりたいこと（ブレ防止）

- SignPath の宣伝記事ではない
- 特定のプロジェクトへの導入レポートではない
- **OSS プロジェクトでコード署名を自動化する手順を、再現可能な形で整理する**

---

## コアメッセージ（1 文で）

> OSS プロジェクトのコード署名は SignPath Foundation + GitHub Actions で無償かつ安全に自動化できる。

---

## 理屈より手順

- 読者がこの記事を見ながら実際にセットアップできることを最優先
- 「なぜコード署名が必要か」は最小限にとどめる
- YAML は省略せず完全な形で掲載する

---

## 選定背景

- Azure Artifact Signing（Azure Trusted Signing）を使いたかったが、日本リージョンでは利用が難しい
- 代替として SignPath Foundation の OSS 向け無償プログラムを選定
- 要件：OSS 向けに導入コスト現実的、秘密鍵を CI に置かない、GitHub 連携

---

## 技術的事実（検証済み）

- GitHub Action: `signpath/github-action-submit-signing-request@v2`（2025年10月リリース）
- OSS では GitHub hosted runner 必須（セルフホストランナー不可）
- origin verification は OSS tier で必須
- Artifact Configuration は `<zip-file>` をルートにする（GitHub が artifact を ZIP 化するため）
- SIGNPATH_API_TOKEN は Secret、SIGNPATH_ORGANIZATION_ID は Variable として登録
- 証明書は SignPath Foundation 名義で発行される（プロジェクト名義ではない）
- 秘密鍵は HSM で管理（CI にもローカルにも秘密鍵は存在しない）

---

## 書かないこと（明確に除外）

- 有償プランとの詳細比較
- SignPath の内部アーキテクチャ
- Windows 以外の署名（対象外であることを明記するのみ）
- 他のコード署名サービスとの詳細比較
- SmartScreen の完全な消し方（注釈程度に触れるのみ）

---

## 書き終えた後のセルフチェック

- 手順通りに進めて再現できるか
- リンク先がすべて有効か
- YAML / XML にシンタックスエラーがないか
- 180 文字/行の制限を超えていないか
- 事実と推測を混ぜていないか

---

## 参考リンク

- https://github.com/SignPath/github-action-submit-signing-request
- https://github.com/SignPath/github-actions-demo
- https://signpath.org/
- https://about.signpath.io/documentation/trusted-build-systems/github
- https://docs.signpath.io/artifact-configuration/syntax
