---
title: "OSS向け無料コード署名サービス SignPath を使ってみた"
emoji: "🔏"
type: "tech"
topics: ["signpath", "codesigning", "githubactions", "windows", "oss"]
published: false
---

Windows 向けに OSS のバイナリ（exe / msi など）を配布していると、コード署名をどうするかという問題に直面します。
署名されていないバイナリは SmartScreen の警告が表示され、ユーザーに不安を与えます。

この記事では、OSS 向けに無料でコード署名を提供している **SignPath** を使い、GitHub Actions から署名済み成果物を GitHub Releases へ載せるまでの手順を紹介します。

## SignPath とは

[SignPath](https://signpath.io/) は、コード署名をクラウド上で完結させるサービスです。
秘密鍵は SignPath 側の HSM（Hardware Security Module）で生成・管理されるため、CI 環境に秘密鍵を置く必要がありません。

OSS プロジェクト向けには [SignPath Foundation](https://signpath.org/) が無料でコード署名証明書を提供しています。
個人の身元確認は不要で、「リポジトリからビルドされたバイナリである」ことを SignPath Foundation が検証し、その名義で署名されます。

### この記事の対象・ゴール

対象読者：

- Windows 向けに OSS のバイナリ（exe / msi など）を配布している
- GitHub Actions でビルドしていて、署名まで自動化したい

ゴール：

- GitHub Actions → SignPath → 署名済み成果物を GitHub Releases へ、の導線を通す

### 選定理由

筆者は当初 [Azure Artifact Signing（旧 Trusted Signing）](https://azure.microsoft.com/en-us/products/artifact-signing)の利用を検討していました。
しかし、Public Trust 証明書の Identity Validation は現時点で米国・カナダ・EU・英国の組織に限定されており、日本の組織や個人開発者では利用が難しい状況です。

https://github.com/Azure/artifact-signing-action/issues/81

そこで、以下の要件を満たす代替として SignPath を選定しました。

- OSS 向けに導入コストが現実的（無料）
- 秘密鍵を CI に置かずに署名運用できる
- GitHub と連携して CI に組み込める

### SignPath の仕組み（最低限）

署名フローは以下のとおりです。

1. CI（GitHub Actions）がビルドした **未署名の成果物** を GitHub Artifact としてアップロード
2. SignPath の GitHub Action が、その Artifact を SignPath へ送信
3. SignPath 側で署名処理が実行される
4. **署名済み成果物** がダウンロード可能になる

署名にはポリシーベースの承認フロー（手動 / 自動）があり、運用設計に関わります。

## 導入手順

### 前提・準備

- 署名したい成果物（例：`app.exe` / `installer.msi`）がある
- GitHub Actions でビルドできる状態になっている
- GitHub リポジトリが公開されている（OSS プラン利用のため）

この記事でやること：

- 署名フローを CI に組み込み、署名済み成果物を Release に載せる

この記事でやらないこと：

- 署名の理論解説（必要最低限の説明に留めます）
- SmartScreen の警告を完全に消すための議論（EV 証明書でも即座には消えない場合があります）

### SignPath 側の設定

#### 1. SignPath Foundation への申請

[SignPath Foundation](https://signpath.org/) に OSS プロジェクトとして申請します。

申請要件：

- OSI 承認のオープンソースライセンスであること
- ソースコードが公開リポジトリにあること
- 無料でダウンロード可能であること
- 悪意のあるコードを含まないこと
- セキュリティ対策を回避するような機能を含まないこと

申請が承認されると、SignPath のアカウントと OSS 用証明書が発行されます。

#### 2. プロジェクト作成

SignPath のダッシュボードでプロジェクトを作成します。

- プロジェクト名を設定
- GitHub リポジトリとの連携を設定（対象リポジトリを紐付け）

#### 3. Artifact Configuration の作成

署名対象のファイル構成を定義します。Artifact Configuration は XML 形式で記述します。

exe 単体の場合：

```xml
<artifact-configuration xmlns="http://signpath.io/artifact-configuration/v1">
  <pe-file>
    <authenticode-sign/>
  </pe-file>
</artifact-configuration>
```

zip 内の exe を署名する場合：

```xml
<artifact-configuration xmlns="http://signpath.io/artifact-configuration/v1">
  <zip-file>
    <pe-file path="*.exe">
      <authenticode-sign/>
    </pe-file>
  </zip-file>
</artifact-configuration>
```

msi の場合（内部の exe も署名する deep signing）：

```xml
<artifact-configuration xmlns="http://signpath.io/artifact-configuration/v1">
  <msi-file>
    <authenticode-sign/>
    <pe-file path="*.exe">
      <authenticode-sign/>
    </pe-file>
  </msi-file>
</artifact-configuration>
```

Deep signing を使うと、インストーラーと内部のファイルを一度のリクエストで署名できます。

#### 4. Signing Policy の設定

署名ポリシーを作成します。典型的には以下の 2 つを用意します。

- **test-signing**: テスト用の自己署名証明書を使用。開発中のビルドに使用
- **release-signing**: 本番用証明書を使用。リリース時に使用

各ポリシーでは以下を設定します。

- 使用する証明書
- 承認フロー（自動承認 or 手動承認）
- ビルド元のブランチ制限（例：`main` や `release/*` のみ）

:::message
最初は手動承認で運用を開始し、フローに慣れてから自動承認への移行を検討するのが安全です。
:::

#### 5. GitHub 連携の設定

SignPath ダッシュボードから GitHub 連携を有効にし、以下を GitHub リポジトリに設定します。

GitHub Secrets に登録する値：

| Secret 名 | 値 |
| --- | --- |
| `SIGNPATH_API_TOKEN` | SignPath で発行した API トークン |

GitHub Variables に登録する値：

| Variable 名 | 値 |
| --- | --- |
| `SIGNPATH_ORGANIZATION_ID` | SignPath の組織 ID |

API トークンは、SignPath のダッシュボードから発行できます。
トークンを発行するユーザーは、対象の Signing Policy で Submitter ロールを持っている必要があります。

### GitHub Actions 側：ワークフロー実装

#### 全体フロー

```text
ビルド → Artifact アップロード → SignPath に署名依頼 → 署名済み成果物取得 → Release へアップロード
```

#### YAML（最小構成サンプル）

```yaml
name: Build and Sign

on:
  release:
    types: [created]

permissions:
  id-token: write
  contents: write
  actions: read

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4

      # ビルド（プロジェクトに合わせて変更）
      - name: Build
        run: dotnet publish -c Release -o ./publish

      # 未署名成果物を zip 化
      - name: Package
        run: Compress-Archive -Path ./publish/* -DestinationPath ./app-unsigned.zip

      # GitHub Artifact としてアップロード
      - name: Upload unsigned artifact
        id: upload
        uses: actions/upload-artifact@v4
        with:
          name: app-unsigned
          path: ./app-unsigned.zip

  sign:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # SignPath に署名リクエストを送信
      - name: Submit signing request
        id: sign
        uses: signpath/github-action-submit-signing-request@v1
        with:
          api-token: ${{ secrets.SIGNPATH_API_TOKEN }}
          organization-id: ${{ vars.SIGNPATH_ORGANIZATION_ID }}
          project-slug: "my-project"
          signing-policy-slug: "release-signing"
          artifact-configuration-slug: "default"
          github-artifact-id: ${{ needs.build.outputs.artifact-id }}
          wait-for-completion: true
          output-artifact-directory: ./signed

      # 署名済み成果物を Release にアップロード
      - name: Upload to Release
        uses: softprops/action-gh-release@v2
        with:
          files: ./signed/*
```

:::message alert
`upload-artifact` の出力から `artifact-id` を取得する方法は、`actions/upload-artifact@v4` の `output` を参照してください。
ジョブをまたいで値を渡す場合は `jobs.<job_id>.outputs` の設定が必要です。
:::

**build ジョブに outputs を追加する例：**

```yaml
jobs:
  build:
    runs-on: windows-latest
    outputs:
      artifact-id: ${{ steps.upload.outputs.artifact-id }}
    steps:
      # ...（上記と同じ）
```

#### Secrets / Permissions の注意点

- `permissions` に `id-token: write` と `actions: read` が必要（SignPath の GitHub Connector が使用）
- `contents: write` は Release へのアップロードに必要
- **Fork からの PR では署名ワークフローを実行しない**こと（Secrets が漏洩するリスクがあるため）
- OSS プランでは、署名リクエストに至るまでのすべてのジョブが **GitHub-hosted runner** で実行されている必要がある

### 動作確認

#### 署名の検証

Windows のファイルプロパティで確認：

署名済みの exe / msi を右クリック → プロパティ → 「デジタル署名」タブで、SignPath Foundation の署名が表示されることを確認します。

signtool で確認：

```powershell
signtool verify /pa /v app.exe
```

#### GitHub Releases での確認

- Release 上の成果物が署名済みファイルに差し替わっていることを確認
- unsigned な成果物が残っていないことを確認

:::message
署名しても SmartScreen の警告がすぐに消えるとは限りません。
SmartScreen はダウンロード数やレピュテーションも考慮するため、署名直後は警告が出ることがあります。
:::

### ハマりどころ

実際に導入する際につまずきやすいポイントをまとめます。

#### 署名対象の取り違え

exe だけ署名してインストーラー（msi）の署名を忘れる、またはその逆のパターンがあります。
Artifact Configuration で署名対象を漏れなく定義してください。
Deep signing を使えば、msi と内部の exe を一度に署名できます。

#### zip のディレクトリ構造

SignPath はアップロードされた Artifact の構造を Artifact Configuration と照合します。
zip 内のディレクトリ構造やファイル名が Configuration と一致しない場合、署名が失敗します。

よくある原因：

- zip 作成時にルートに余計なディレクトリが入る
- ワイルドカードの `max-matches` / `min-matches` とファイル数が合わない

#### 成果物パスの不一致（matrix ビルド時）

matrix ビルドで複数 OS / アーキテクチャの成果物を生成する場合、パスやファイル名がずれることがあります。
Artifact 名を `app-unsigned-${{ matrix.arch }}` のように分けるか、Artifact Configuration 側でワイルドカードを適切に設定します。

#### 手動承認での CI 停止

Signing Policy で手動承認を設定している場合、CI は承認待ちで停止します。
`wait-for-completion-timeout-in-seconds`（デフォルト 600 秒）を超えるとタイムアウトします。

手動承認を使う場合は、タイムアウト値を十分に大きく設定するか、`wait-for-completion: false` にして別のワークフローで署名完了を確認する運用を検討してください。

#### リリースへの成果物差し替え方針

署名前の成果物がユーザーに配布されないよう、以下のいずれかの運用を推奨します。

- **Draft Release → 署名後に Publish**: Release を draft 状態で作成し、署名済み成果物のアップロード後に publish する
- **署名済みのみアップロード**: 未署名成果物は GitHub Artifact のみに保存し、Release には署名済みファイルのみをアップロードする

## まとめ

### 今回やったこと

- SignPath を使って CI から署名を実行した
- 署名済み成果物を GitHub Releases へ載せた
- 鍵を CI に置かない形で署名運用できた

### 運用の指針

- 署名は tag / release 時のみに絞るのが安全で現実的
- 最初は手動承認で開始し、慣れたら自動承認を検討
- unsigned な成果物を配布物に混ぜない（事故防止）

### 次にやると良いこと

- SBOM の生成・公開（SignPath のワークフローにも組み込める）
- リリースフローの整備（draft → 署名 → publish の自動化）
- ハッシュ値の公開やアーティファクトの provenance 対応
