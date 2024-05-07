---
title: "GitHub Actions プルリクエストを作成するシンプルなジョブのサンプル"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [github,githubactions,githubcli,ci,cicd]
published: false
---

## 概要
プルリクエストを作成する GitHub Actions のジョブを作成しました。

## 仕様
仕様は以下の通りシンプルです。
- マージ先ブランチは`main`とする
- コミットメッセージをプルリクエストタイトルに使用する
- Assigneesはワークフローの実行を最初にトリガーしたユーザーを設定する
- ワークフローを実行したブランチのプルリクエストが既に存在している場合、プルリクエストを作成しない

### 実行結果
- [ワークフロー](https://github.com/mskasa/sample-actions/actions/runs/8979599756)
- [既にプルリクエストが存在する場合の実行結果](https://github.com/mskasa/sample-actions/actions/runs/8979646254/job/24661984497#step:3:16)
- [作成されたプルリクエスト](https://github.com/mskasa/sample-actions/pull/1)

## 実装
- 全体は[こちら](https://github.com/mskasa/sample-actions/blob/main/.github/workflows/create-pull-request.yml)
```yml
jobs:
  create_pull_request:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Check and Create Pull Request
        env:
          GH_TOKEN: ${{ secrets.ACTIONS_TOKEN }}
        run: |
          PR_COUNT=$(gh pr list --head "${{ github.ref_name }}" --json number --jq '. | length')
          if [[ $PR_COUNT == 0 ]]; then
            gh pr create \
              -B main \
              -t "${{ github.event.head_commit.message }}" \
              -a "${{ github.actor }}"  \
              -F ./.github/workflows/PR_template.md
          else
            echo "PR already exists."
          fi
```

### 詳細
#### トークンの設定
```yml
env:
  GH_TOKEN: ${{ secrets.ACTIONS_TOKEN }}
```
GitHub Actions で GitHub CLI を使用したい場合、環境変数`GH_TOKEN`を設定します（[参考](https://docs.github.com/ja/actions/security-guides/automatic-token-authentication#example-1-passing-the-github_token-as-an-input)）。
`ACTIONS_TOKEN`は任意の名前です。必要な権限を設定したトークンを作成して、シークレットに設定してください。

:::message
- GitHub Enterpriseの場合、設定する環境変数は以下になるので注意してください
```yml
env:
  GH_ENTERPRISE_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  GH_HOST: github.example.com
```
:::

#### 該当ブランチのプルリクエスト数を取得
```sh
gh pr list --head "${{ github.ref_name }}" --json number --jq '. | length'
```
- コマンド[gh pr list](https://cli.github.com/manual/gh_pr_list)を使用
  - オプション`--json number` で JSON形式で出力（フィールドは適当に`number`のみ指定）
- `github.ref_name`でワークフロー実行中のブランチ名を取得（[コンテキスト一覧](https://docs.github.com/ja/actions/learn-github-actions/contexts)）
- jqのフィルター`. | length`で、JSONオブジェクトの配列の要素数を取得
  - `.`はJSONデータそのものであり、jq の length関数に渡している

#### プルリクエストの作成
```sh
gh pr create \
  -B main \
  -t "${{ github.event.head_commit.message }}" \
  -a "${{ github.actor }}"  \
  -F ./.github/workflows/PR_template.md
```
- コマンド[gh pr create](https://cli.github.com/manual/gh_pr_create)を使用
- `github.event.head_commit.message`や`github.actor`は[コンテキスト一覧](https://docs.github.com/ja/actions/learn-github-actions/contexts)を参照
- `-F`オプションでプルリクエストの本文を設定可能

## まとめ
シンプルなものが無かったので共有してみました。
本サンプルをベースにして、プロジェクトに合わせて色々カスタマイズしてみてください。

## 参考
https://cli.github.com/manual/

https://docs.github.com/ja/actions/learn-github-actions/contexts

https://github.com/mskasa/sample-actions/actions