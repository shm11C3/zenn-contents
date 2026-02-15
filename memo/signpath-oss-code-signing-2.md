---
title: "OSS プロジェクトのコード署名を SignPath + GitHub Actions で自動化する"
emoji: "🔏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "codesigning", "oss", "cicd", "security"]
published: false
---

Windows 向けに OSS のバイナリ（exe / msi など）を配布する際、コード署名がないと SmartScreen 警告が表示されます。
この記事では、SignPath Foundation の OSS 向け無償プログラムと GitHub Actions を組み合わせて、署名済み成果物を GitHub Releases へ載せるまでの手順を整理します。

## この記事の対象と前提

**対象読者：**

- Windows 向けに OSS のバイナリを配布している
- GitHub Actions でビルドしていて、署名まで自動化したい

**この記事で扱うこと：**

- SignPath Foundation への申請から GitHub Actions での署名自動化までの一連の手順
- 署名済み成果物を GitHub Releases へアップロードする導線

**この記事で扱わないこと：**

- コード署名の理論的な解説（必要最低限の説明に留めます）
- SmartScreen 警告を完全に消すための議論（EV 証明書でも即座には消えないため、注釈程度）
- Windows 以外のプラットフォーム向け署名

## SignPath とは

### 選定理由

筆者は当初 Azure Trusted Signing（旧 Azure Code Signing）の導入を検討していましたが、日本リージョンでの利用が難しい状況でした。
代替として、以下の要件を満たすサービスとして SignPath を選定しました。

- OSS 向けに無償で利用できる
- 秘密鍵を CI に置かずに署名運用できる（HSM でのキー管理）
- GitHub Actions とネイティブに連携できる

https://signpath.org/

### 仕組み（導入に必要な最低限）

SignPath による署名フローは以下のとおりです。

1. CI（GitHub Actions）が未署名の成果物を生成し、GitHub Artifact としてアップロード
2. SignPath GitHub Action が署名リクエストを送信
3. SignPath 側でマルウェアスキャン → 承認（手動 or 自動） → 署名
4. 署名済み成果物がダウンロードされる

ポイントとして、秘密鍵は SignPath 側の HSM（Hardware Security Module）で管理されます。CI 環境にもローカル環境にも秘密鍵は存在しません。
また、OSS プログラムでは証明書が **SignPath Foundation 名義** で発行されます（プロジェクト名義ではありません）。

## 導入手順

### 前提・準備

- 署名したい成果物がある（例：`app.exe`、`installer.msi`）
- GitHub Actions でビルドできる状態になっている
- リポジトリが OSI 承認のオープンソースライセンスで公開されている

### SignPath Foundation への申請

SignPath Foundation の OSS プログラムに申請します。

**申請ページ：** <https://signpath.org/apply.html>

申請時に必要な情報：

- リポジトリの URL
- 使用しているライセンス（OSI 承認であること）
- プロジェクトの概要

申請後、SignPath Foundation による審査が行われます。承認されるとメールで連絡が届き、SignPath のダッシュボードにアクセスできるようになります。

**申請時の注意点：**

- プロプライエタリなコンポーネントを含むプロジェクトは対象外です
- セキュリティ診断ツール等も対象外となる場合があります
- プロジェクトがアクティブにメンテナンスされている必要があります

### SignPath 側の設定

#### プロジェクトの作成

SignPath ダッシュボードにログイン後、プロジェクトを作成します。

- プロジェクト名（slug）を決める（例：`my-app`）
- 対象の GitHub リポジトリを紐付ける

#### Artifact Configuration の作成

署名対象のファイル構成を XML で定義します。GitHub Actions の `upload-artifact` はファイルを ZIP 化してアップロードするため、`<zip-file>` をルート要素にします。

**最小構成の例（exe 単体）：**

```xml
<?xml version="1.0" encoding="utf-8"?>
<artifact-configuration xmlns="http://signpath.io/artifact-configuration/v1">
  <zip-file>
    <pe-file path="MyApp.exe">
      <authenticode-sign/>
    </pe-file>
  </zip-file>
</artifact-configuration>
```

**MSI + 内部の exe/dll も署名する例：**

```xml
<?xml version="1.0" encoding="utf-8"?>
<artifact-configuration xmlns="http://signpath.io/artifact-configuration/v1">
  <zip-file>
    <msi-file path="MyApp.msi">
      <directory path="application">
        <pe-file-set>
          <include path="MyApp.exe" />
          <include path="MyApp.dll" />
          <for-each>
            <authenticode-sign/>
          </for-each>
        </pe-file-set>
      </directory>
      <authenticode-sign/>
    </msi-file>
  </zip-file>
</artifact-configuration>
```

MSI の場合、内部の PE ファイルを先に署名し、その後 MSI 自体を署名する構成にします。`<pe-file-set>` で複数ファイルをまとめて処理できます。

#### Signing Policy の設定

Signing Policy は「どの条件で」「どの証明書で」署名するかを定義します。

| Policy          | 用途           | 証明書       | ブランチ制限        |
| --------------- | -------------- | ------------ | ------------------- |
| test-signing    | テスト・検証用 | テスト証明書 | なし                |
| release-signing | リリース配布用 | 本番証明書   | `main`, `release/*` |

- **test-signing** はすべてのブランチ・PR で使用し、署名フロー自体の動作確認に使います
- **release-signing** は `main` や `release/*` ブランチに限定し、実際の配布物に使用します

#### Origin Verification の設定

OSS プログラムでは origin verification が必要です。これにより、署名リクエストが実際に GitHub Actions ワークフローから送信されたことを検証できます。

設定手順：

1. SignPath GitHub App をリポジトリにインストール
2. SignPath ダッシュボードで GitHub connector を設定
3. 対象リポジトリのブランチ保護ルール（またはルールセット）を設定

ブランチ保護ルールは、origin verification でビルドの信頼性を担保するために必要です。

#### API トークンの発行と GitHub への登録

SignPath ダッシュボードから API トークンを発行し、GitHub リポジトリに登録します。

- **Secret:** `SIGNPATH_API_TOKEN`（API トークン）
- **Variable:** `SIGNPATH_ORGANIZATION_ID`（Organization ID）

`SIGNPATH_ORGANIZATION_ID` は機密情報ではないため、Variable（Actions variables）として登録します。

### GitHub Actions ワークフローの実装

#### 全体フロー

```text
ビルド → 未署名成果物をアップロード → SignPath に署名依頼
→ 署名済み成果物を取得 → GitHub Releases へアップロード
```

#### ワークフロー YAML（最小構成サンプル）

```yaml
name: build-and-sign

on:
  push:
    tags:
      - "v*"
  workflow_dispatch:

jobs:
  build_and_sign:
    runs-on: windows-latest
    permissions:
      id-token: write
      contents: write
      actions: read

    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: dotnet build --configuration Release --output ./build

      - name: Upload unsigned artifact
        id: upload-unsigned
        uses: actions/upload-artifact@v4
        with:
          name: my-app-unsigned
          if-no-files-found: error
          path: ./build/MyApp.exe

      - name: Sign with SignPath
        uses: signpath/github-action-submit-signing-request@v2
        env:
          SIGNPATH_SIGNING_POLICY_SLUG: >-
            ${{ (github.ref == 'refs/heads/main'
              || startsWith(github.ref, 'refs/tags/v'))
              && 'release-signing'
              || 'test-signing' }}
        with:
          api-token: "${{ secrets.SIGNPATH_API_TOKEN }}"
          organization-id: "${{ vars.SIGNPATH_ORGANIZATION_ID }}"
          project-slug: "my-app"
          signing-policy-slug: "${{ env.SIGNPATH_SIGNING_POLICY_SLUG }}"
          github-artifact-id: "${{ steps.upload-unsigned.outputs.artifact-id }}"
          wait-for-completion: true
          output-artifact-directory: "./signed"

      - name: Upload signed artifact
        uses: actions/upload-artifact@v4
        with:
          name: my-app-signed
          path: ./signed/
          if-no-files-found: error

      - name: Upload to GitHub Release
        if: startsWith(github.ref, 'refs/tags/v')
        env:
          GH_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
        run: |
          gh release upload "${{ github.ref_name }}" ./signed/MyApp.exe --clobber
```

#### 各ステップの補足

**トリガー：**

- `push.tags` でタグプッシュ時に自動実行します。リリース時のみ署名を実行する構成です
- `workflow_dispatch` を入れておくと、手動実行でテスト署名の動作確認ができます

**permissions：**

- `id-token: write` — SignPath の origin verification に必要
- `contents: write` — GitHub Releases へのアップロードに必要
- `actions: read` — SignPath Action がワークフロー情報を参照するために必要

**signing-policy-slug の動的切り替え：**

`env` で条件分岐させ、`main` ブランチやタグプッシュでは `release-signing`、それ以外では `test-signing` を使用します。

**GitHub Releases へのアップロード：**

`gh release upload` で署名済み成果物をリリースに追加します。`--clobber` を付けると、同名ファイルがあれば上書きします。

#### Secrets / Permissions の注意点

- fork からの PR では署名ワークフローを走らせないでください。fork の PR は信頼境界の外にあるため、Secrets にアクセスさせるべきではありません
- OSS プログラムでは **GitHub hosted runner の使用が必須** です。セルフホストランナーは使用できません
- `GITHUB_TOKEN` の `contents: write` 権限は、Release へのアップロードに必要です

### 動作確認

#### Windows のファイルプロパティで確認

署名済みの exe / msi を右クリック → プロパティ → 「デジタル署名」タブが表示されていれば署名されています。
署名者名が「SignPath Foundation」となっていることを確認してください。

#### PowerShell で確認

```powershell
Get-AuthenticodeSignature .\MyApp.exe | Format-List
```

`Status` が `Valid`、`SignerCertificate` の Subject に `SignPath Foundation` が含まれていれば成功です。

#### signtool で確認

Windows SDK に含まれる `signtool` でも確認できます。

```powershell
signtool verify /pa .\MyApp.exe
```

#### GitHub Releases 上での確認

リリースページにアップロードされた成果物が署名済みのものに差し替わっていることを確認します。
unsigned のまま公開されないよう、ワークフローの順序に注意してください。

### ハマりどころ

この章では、導入時に遭遇しやすい問題をまとめます。

**署名対象の取り違え：**

exe だけ署名して installer（msi）を忘れる、またはその逆。Artifact Configuration で署名対象を漏れなく定義してください。
MSI の場合、MSI 自体と内部の PE ファイルの両方を署名する必要があります。

**ZIP のディレクトリ構造の不一致：**

`upload-artifact` でアップロードするファイルのパス構造と、Artifact Configuration で定義した `path` 属性が一致しない場合、署名に失敗します。
ローカルでアーティファクトの ZIP を展開して構造を確認するのが確実です。

**成果物パスが OS / アーキテクチャでズレる：**

matrix ビルドを使用している場合、OS やアーキテクチャごとに出力パスが変わることがあります。
パスをハードコードせず、ビルドツールの出力変数を利用してください。

**手動承認で CI が止まる：**

Signing Policy で手動承認を設定した場合、署名リクエストが承認待ちで止まります。
`wait-for-completion-timeout-in-seconds`（デフォルト 600 秒）を超えるとタイムアウトします。
運用に合わせてタイムアウト値を調整するか、自動承認の導入を検討してください。

**origin verification failed：**

ブランチ保護ルール（またはルールセット）が正しく設定されていない場合に発生します。
SignPath ダッシュボードの GitHub connector 設定と、リポジトリのブランチ保護設定を確認してください。

**リリース差し替え方針：**

unsigned な成果物がリリースに残る事故を防ぐため、以下の運用を推奨します。

- タグプッシュ時に draft リリースを作成
- 署名完了後に署名済み成果物をアップロード
- すべてのアセットが揃ったらリリースを publish

## まとめ

### やったこと

- SignPath Foundation の OSS プログラムを利用して、CI からコード署名を実行する手順を整理しました
- GitHub Actions のワークフローに署名ステップを組み込み、署名済み成果物を GitHub Releases へアップロードする導線を構築しました
- 秘密鍵を CI にもローカルにも置かない形で、署名運用を実現しました

### 運用の指針

- 署名は tag プッシュ / release 時のみに絞るのが安全で現実的です
- 最初は手動承認で運用し、フローが安定したら自動化を検討してください
- unsigned な成果物を配布物に混ぜないよう、リリースフローを整備してください

### 次にやると良いこと

- SBOM（Software Bill of Materials）の生成と署名（SignPath は CycloneDX の XML 署名にも対応しています）
- provenance やハッシュの公開による追加の信頼性担保
- リリースフローの整備（draft 化、承認担当の明確化、自動化の段階的導入）

## 参考リンク

- [SignPath Foundation](https://signpath.org/)
- [SignPath GitHub Action](https://github.com/SignPath/github-action-submit-signing-request)
- [SignPath GitHub Actions Demo](https://github.com/SignPath/github-actions-demo)
- [SignPath Docs - GitHub Integration](https://about.signpath.io/documentation/trusted-build-systems/github)
- [SignPath Docs - Artifact Configuration](https://docs.signpath.io/artifact-configuration/syntax)
