---
title: "github private packages を利用した .npmrcの扱い方"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [tech,githubactions,github,npm]
published: true
---

## モチベーション

github packages を利用した .npmrc ファイルの扱いについてちょっと詰まったので覚書兼メモになります。

また、下記のような背景で困っている方の一助になれば幸いです。
- 社内でプライベートリポジトリを運用している / これから検討している
  - github packages を利用している / 検討している
- .npmrc を コミットせずに管理したい

※ github packages 、.npmrc がどういうものであるかについては割愛します。
※ 今回は**プライベートパッケージを利用する**ことに焦点を当てています。

## 実現したいこと

- GitHub Packages に公開しているプライベートなパッケージを利用する際に必要な **.npmrc ファイルを、GitHub 上にコミットせずに扱う**方法を解説します。
  - この方法を使うことで、Personal Access Token (以下 PAT) が載った .npmrc ファイルを GitHub 上にコミットする必要がなくなります。これにより、セキュリティ上の問題を回避しつつ、パッケージを利用できます。

## 流れ

**手順1：Personal Access Token (PAT) を生成する**

1. github のsettings より PAT を生成してください。
    1. `repo` と `read:packages` にチェックの入ったトークンが必要になります。
2. トークンをコピーしておきます。
    1. ローカルでパッケージを利用する際は、使用する人に応じて各々PATが必要になります。

**手順2：packages.json に preinstall コマンドを登録**

1. packages.json にpreinstallコマンドを登録。

    ```json
    "scripts": {
        "preinstall": "sh ./scripts/preinstall.sh",
    ...
    ```

**手順3：必要ファイルを作成**

1. プロジェクトディレクトリに下記ファイルを作成。
    1. .npmrc.example（`.npmrc` の雛形）

        ```tsx
        //npm.pkg.github.com/:_authToken=PERSONAL_ACCESS_TOKEN
        @[組織名]:registry=https://npm.pkg.github.com
        ```

    2. .env

        ```tsx
        PERSONAL_ACCESS_TOKEN= 手順1で発行したPAT
        ```

        1. 併せて .env.example のような雛形ファイルがあると良いです

    3. preinstall.sh（今回は scriptsディレクトリに配置します。）

        ```bash
        # ./scripts/preinstall.sh
        
        # ローカル環境：
        if [ -f "./.env" ]; then
          echo ".env ファイルを確認しました。"
          source "./.env"
        else
        # リモート環境：
          echo ".env ファイルが見つかりませんでした。"
        fi
        
        \cp -r ./.npmrc.example ./.npmrc
        sed -i "" "s|PERSONAL_ACCESS_TOKEN|${PERSONAL_ACCESS_TOKEN}|g" ./.npmrc
        ```

        :::details 上記処理について
        1. プロジェクトルートに envファイルの有無を確認
            1. あったら env ファイルの環境変数を取得する
            2. なければスキップ
        2. プロジェクトルートにある .npmrc.example ファイルを元に.npmrc ファイルを生成する。
        3. .npmrc 内の「PERSONAL_ACCESS_TOKEN」の値を1で取得した内容に書き換える。
        :::

**手順4：実際にnpm install を実行してみる（ローカル上でパッケージを利用する場合）**

前述の手順を踏んで、.npmrcが正しく設定されているか確認するために、実際にnpm installを実行してみます。

```
npm install
```

npm installを実行すると、packages.jsonに記載された依存関係のパッケージがインストールされます。この際、.npmrcファイルに記述された認証情報が使われます。

### Github Actions を経由してプロジェクトをビルド（or デプロイ）する方法

Github Actions を経由して プロジェクトをデプロイする場合はリポジトリ内で設定できる secrets を利用すると良いです。

https://docs.github.com/ja/actions/security-guides/encrypted-secrets

**手順1：Personal Access Token (PAT) を生成 > secret に登録**

1. 先で生成したPAT を 対象のリポジトリの secrets に登録する。
    1. setting > secrets and actions > New repository secret
    2. こちらのケースではローカルでの運用と違い、個々人でPATを生成する必要がないため、チーム内で誰かが生成したPAT一つを secret として登録するだけでOKです。

**手順2：ワークフローを作成**

1. 下記のワークフローの内容を含んだyamlファイルを作成する。

    ```yaml
    ./github/workflows/deploy.sh
    
    ...
    - name: Create .npmrc with replace token
            shell: bash
            env: 
              PERSONAL_ACCESS_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }} ## secrets に登録したトークン
            run: |
              \cp -r ./.npmrc.example ./.npmrc
              sed -i "" "s|PERSONAL_ACCESS_TOKEN|${PERSONAL_ACCESS_TOKEN}|g" ./.npmrc
    
          - name: Check .npmrc
            run: |
              cat ./.npmrc
    
          - name: [ビルド処理 etc]
    ...
    ```

トークンの取得：

```json
env: 
  PERSONAL_ACCESS_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
```

上記処理により secret に登録されている値を参照するため、ローカルで利用する場合と違い、一つのPATで.npmrcファイルの生成までが実現可能になります。

**手順3：ワークフローを実行する**

1. ビルド or ホスティング先へデプロイ。ワークフローが無事に実行されているかを確認
    1. 実際にNext.jsのプロジェクトを Cloud Flare Pages にデプロイするケースがあったため、参考記事を元にデプロイ処理を別途追加しました。（[こちらの記事](https://zenn.dev/nekoshita/articles/cd39383254d55d)を参考にさせていただきました。）

## まとめ

以上で、.npmrcファイルをgithubにコミットせずにgithub private packages を利用する方法について説明しました。この方法を用いることで、パッケージの利用に必要な認証情報を外部に漏らさずに済むため、セキュリティの観点からも安全です。
shellコマンドめっちゃ便利だなあと改めて感じた今日この頃。

## 参考文献考：
https://www.mizdra.net/entry/2021/03/31/235055
https://qiita.com/marumaru0113/items/21b600c21caf5d9b9775
