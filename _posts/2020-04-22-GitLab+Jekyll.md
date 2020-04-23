---
title: GitLab+JekyllでGitHub Pagesクローンを立てる
---

# GitLab + Jekyll で GitHub Pagesクローンを立てる

GitHub Pagesは静的ウェブサイトをMarkdownを使って簡単に公開できるので非常に便利です。
しかし、非公開のサイトを作ることはできません（頑張れば不可能ではないらしいですが）。  
そこで、オンプレミスにGitHub Pagesのクローンを立てたときの記録です。  
レポジトリもオンプレミスにあるので、組織外持ち出し禁止情報も安心して掲載できます。


* 目次
{:toc}

## 環境
- Centos 7
- GitLab 12.9
- Jekyll 3.8


## GitLab Community Editionのインストール
[GitLab](https://about.gitlab.com/stages-devops-lifecycle/)
([Wikipedia](https://ja.wikipedia.org/wiki/GitLab))は、
GitHubのようなウェブサイトを提供するサーバソフトウェア  
オープンソースのCommunity Editionをインストールする。

[公式ドキュメント](https://about.gitlab.com/install/?version=ce#centos-7)に従って
以下の手順でインストール

### 依存パッケージのインストールや設定など
- 必要パッケージをインストール  
- sshdの起動
- firewallでHTTP, HTTPSへのアクセスを許可
- メール送信のためのPostfixのインストール・起動

### GitLabのpackage repositoryの追加
```shell
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```

### gitlab-ceパッケージのインストール
```shell
sudo EXTERNAL_URL="https://gitlab.example.com" yum install -y gitlab-ce
```
- `EXTERNAL_URL`の内容は公開URL
- gitlab-ceの中にnginx, postgresqlなど諸々が含まれている。
- この時点でEXTERNAL_URLにインターネットからHTTPSでアクセス可能だと、
  自動的にLet's Encryptのサーバ証明書が取得されて設定される。

### 初期設定
1. `EXTERNAL_URL`にアクセス
2. rootパスワードの設定
3. Usernameをrootにして上で設定したパスワードでサインイン
4. Admin Area（ページ最上部スパナマーク） -> Settings から設定

- デフォルトだと誰でもSign-up（ユーザ登録）できてしまうので、インターネット上に公開するなら、
  "Sign-up restrictions"の"Send confirmation email on sign-up"にチェックを入れて
  "Whitelisted domains for sign-ups"に登録可能なドメインを指定するのが良い。


## GitLab Runnerのインストール
[GitLab Runner](https://docs.gitlab.com/runner/)は
repositoryへのcommitに対してjobを自動実行するためのCI/CDツール  
今回はGitLabと同じホスト上でshared Runnerが動作するようにインストール・設定する。

[公式ドキュメント](https://docs.gitlab.com/runner/install/linux-repository.html)に従って
以下の手順でインストール

### GitLabのpackage repositoryの追加
```shell
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
```

### gitlab-runnerパッケージのインストール
```shell
sudo yum install gitlab-runner
```


## GitLab RunnerのGitLabへの登録
[公式ドキュメント](https://docs.gitlab.com/runner/register/index.html)に従って
以下の手順で登録

### 登録情報の確認
1. `EXTERNAL_URL`にアクセスしてrootでサインイン
2. Admin Area -> Overview -> Runners を開く
3. "Set up a shared Runner manually"にあるURLとtokenを確認

### gitlab-runnerの登録設定
```shell
sudo gitlab-runner register
```
- URL, tokenは上で確認したものを指定
- descriptionとtagsはデフォルトのままで良い
- executorはとりあえずdocker 
  （[各Executorの説明](https://docs.gitlab.com/runner/executors/README.html))  
  Docker imageもとりあえずalpine:latest


## Pagesレポジトリの作成・設定
GitHub PagesのようにJekyllを用いて自動でサイトが更新されるレポジトリを作成して設定する。

ここでは`public`ディレクトリ以下を`pages.example.com`にFTPSでアップロードする場合について説明する。

1. 一般ユーザーでサインイン
2. レポジトリを作成
3. Settings -> CI/CD -> Variablesで以下の内容を"Add Variable"
    - Key: LFTP_PASSWORD
    - Value: FTPSで使用するパスワード
      （[Base64](https://ja.wikipedia.org/wiki/Base64)の文字しか使えないので注意）
    - Type: Var
    - Environment scope: All
    - Mask variableにチェック
3.  手元にgit clone
4. ファイル`.gitlab-ci.yml`を作成  
    ```yaml
    image: jekyll/builder:3.8

    deploy:
      stage: deploy
      script:
        - (cd public; jekyll build)
        - lftp -f lftp.scr
    ```
5. ファイル`lftp.scr`を作成  
    ```yaml
    set ssl:verify-certificate false
    open --user USERNAME --env-password ftp://pages.example.com mirror -e -n -R public/_site .
    ```
    - USERNAMEはFTPSで使用するユーザー名
6. ファイル``public/index.md``を作成（内容は適当に）
7. git commit & push

後は基本的にGitHub Pagesと同様