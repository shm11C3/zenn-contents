---
title: "無料でコード署名ができるOSS向けサービス SignPath Foundation を申請してみた①（申請編）"
emoji: "📝"
type: "tech"
topics: ["signpath", "codesigning", "oss", "security", "github"]
published: true
---

Windows 向けのコード署名サービスとして、今までは「Azure Artifact Signing（旧 Trusted Signing）」が有力な選択肢でした。  
しかし、2026年2月時点で、Azure Artifact Signing の Public Trust は「米国・カナダ・EU・英国の組織」と「米国・カナダの個人開発者」に限定されています。  
日本の組織/個人は Public Trust を前提にすると利用が難しいため、代替として OSS 向け無料署名の SignPath Foundation を検討しました。

この記事では、SignPath Foundation への申請手順（準備〜申請フォーム送信）を整理します。

この記事は、筆者が [HardwareVisualizer](https://github.com/shm11C3/HardwareVisualizer)（PC向けハードウェアモニターソフト）のコード署名化を進めるなかで、実際に SignPath Foundation へ申請した体験をもとに書いています。

:::message
この記事は「申請まで」がゴールです。
承認後の SignPath 設定や GitHub Actions 連携は承認が通り次第、別記事で扱います。
:::

## 対象読者

- OSS の Windows 向け配布物を出している（または出す予定）
- GitHub などでソースコードを公開している
- まずは SignPath Foundation に申請してみたい

## 申請前に押さえること

SignPath Foundation は「OSS の配布物を安全に署名する」ためのプログラムなので、プロジェクトが条件を満たしている必要があります。

申請ページ：<https://signpath.org/apply.html>

（要件は更新される可能性があるので、最終的には申請ページの記載を優先してください）

最低限チェックしたいポイント：

- OSI 承認のオープンソースライセンスである
- ソースコードが公開リポジトリにある
- 配布物が無料でダウンロードできる
- 悪意のある挙動を含まない（マルウェア、セキュリティ回避用途などでない）

## 申請を通しやすくするための準備チェックリスト

申請フォーム自体は難しくありませんが、審査側が判断しやすい状態にしておくと承認されやすくなります。
今回は以下の準備を行いました。

- コード署名ポリシーの作成 <https://github.com/shm11C3/HardwareVisualizer/pull/1123>
- CODE_OF_CONDUCT の作成 <https://github.com/shm11C3/HardwareVisualizer/pull/1124>

今回作成したコード署名ポリシー

```md
# Code signing policy

This project signs and distributes release artifacts. The signing method differs by platform.

## Windows — SignPath Foundation (pending)

We are applying to the SignPath Foundation program.

Planned statement (required by the program, if approved):
"Free code signing provided by SignPath.io, certificate by SignPath Foundation"

Status: Pending approval.

### What will be signed

- Windows installer packages (e.g. .exe, .msi) published on GitHub Releases.

### Build and signing process

- Artifacts are built from this repository using CI.
- Only CI-built artifacts will be submitted to SignPath for signing.
- The private key is held by SignPath (HSM-backed). This project does not store the private key.

### Team roles (single-maintainer project)

- Authors (commit access, can modify the repository without additional reviews):
  - <https://github.com/shm11C3>

- Reviewers (review required for changes proposed by non-committers, e.g. pull requests):
  - <https://github.com/shm11C3>
  - Policy: All external pull requests are reviewed by the maintainer before merge.

- Approvers (approve each signing request):
  - <https://github.com/shm11C3>
  - Policy: Each signing request requires explicit approval by the maintainer.

## macOS

- Signed with Apple Developer ID and notarized by Apple.

## Linux (currently unsigned)

Status: Not implemented yet.

### What is distributed

- Linux artifacts (e.g. AppImage, .deb, .rpm) published on GitHub Releases.

### Verification

- At this time, Linux artifacts are not cryptographically signed by this project.
- Users should obtain artifacts only from the official GitHub Releases page.

### Future plan (non-binding)

- We may add artifact signing (e.g. Sigstore/cosign or GPG) in a future release.

## Distribution locations

- <https://github.com/shm11C3/HardwareVisualizer/releases>

## Privacy policy

This program will not transfer any information to other networked systems unless specifically requested by the user.
```

コード署名ポリシーの中身はあくまでも例です。  
プロジェクトの状況や方針に合わせて、必要な内容を記載してください。

`CODE_OF_CONDUCT` は OSS プロジェクトとしての健全な運営方針を明示的にするために作成しました。  
中身は [Contributor Covenant](https://www.contributor-covenant.org/) をベースに作成しています。

### リポジトリの見え方を整える

- `LICENSE`（または `LICENSE.txt`）がリポジトリ直下にある
- README に「このプロダクトが何か」「どこからダウンロードできるか」が書かれている
- ビルド方法が README にある（手動でも CI でもよい）

### 配布物の出どころを説明できるようにする

審査側が知りたいのは、ざっくり言うと次の2点です。

1. 署名対象のバイナリが「この OSS プロジェクト」由来である
2. バイナリのビルドが再現可能で、改ざん混入しにくい

そのため、以下が揃っていると説明が簡単になります。

- GitHub Releases（または公式配布ページ）がある
- CI（GitHub Actions など）でビルドしているなら、ワークフローを公開している

### 連絡先（メール）を用意する

申請後のやり取りはメールが中心になります。

- 返信できるメールアドレス（個人でも可）
- プロジェクトのメンテナとして連絡が取れること

## 申請フォームの書き方

申請ページ：<https://signpath.org/apply.html>

フォームの文言は変更される可能性がありますが、基本的には次の情報を求められます。

### Project / Repository URL

- GitHub リポジトリの URL（公開）を指定します
- ミラーが複数ある場合は、開発の一次情報がある場所を選びます

### License

- OSI 承認ライセンス名（MIT / Apache-2.0 / GPL など）を明記します
- `LICENSE` ファイルがあることも合わせて示せるとスムーズです

### Download / Release URL

- ユーザーが入手する配布物の URL を指定します
  - GitHub Releases
  - 公式サイトのダウンロードページ

「配布物が無料でダウンロードできる」ことが読み取れるリンクにします。

### Project description

短くてもよいので、審査側が誤解しないように以下を入れるのが安全です。

- 何のアプリ/ツールか
- どんなユーザーが使うか
- 配布している成果物の種類（exe / msi / zip など）

例（雰囲気）：

```text
This project provides a Windows desktop application for ...
We distribute binaries (exe/msi) via GitHub Releases.
The binaries are built using GitHub Actions from the public repository.
```

:::message
英語で書くのが無難です（審査側の運用上、英語が前提になっていることが多いため）。
ただし、最低限伝わる内容なら簡潔で問題ありません。
:::

## 申請後に起きること

この記事の範囲は「申請を出すまで」ですが、次の流れだけ把握しておくと安心です。

- 申請内容の確認（追加質問が来る場合があります）
- 承認されると、SignPath 側で利用開始の案内が届く
- 以降は SignPath のプロジェクト作成、GitHub 連携、署名ポリシー設計へ進む

## 次にやること

申請が通ったら、次は「SignPath の設定」と「GitHub Actions での署名自動化」に進みます。

- 続編：SignPath + GitHub Actions で署名を自動化する（申請が通ったら公開します）
