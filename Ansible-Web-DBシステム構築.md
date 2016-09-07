## Ansibleを使用したWeb+DBサーバ構築とアプリデプロイ

### 目的（想定読者が資料を読んだ後にとる行動）
読者がAnsibleを使用する

### 想定読者
Ansibleを使用したことがないエンジニア  
VMの知識はある  
実行コマンドが細かく書いてあると行動しやすい  

### 概要
Webサーバ+DBサーバの環境構築をAnsibleにより自動で作成する  
アプリをデプロイし、動作確認を行う  
細かいセキュリティ等については考慮しない  

### 構成図
Webサーバ(Apache)1台とDBサーバ(MySQL)1台を連携させる  
管理サーバ(Ansible)1台からAnsibleを実行し、自動構築を行う　　

### 環境情報
  - マシン
    - Vagrant 1.8.1
    - Virtual Box 5.0
  - OS
    - CentOS 6.7
  - ソフトウェアバージョン
    - Ansible 2.1.1.0
    - Apache http server 2.2.15
    - MySQL 5.1
    - Ruby 2.3.1
    - Rails 4.2.6

### 作成手順

#### VMを3台デプロイする　　
1. 以下のVagrantfileを用意する
```
  ※ Vagrantfileの中身を記載する
```
IP Address  
管理サーバ: 192.168.100.10  
webサーバ: 192.168.100.20  
DBサーバ: 192.168.100.30  

管理サーバでansibleとgitのインストール、鍵の作成まで行っている。  

1. Vagrantを実行しVMをデプロイ
```
  $ vagrant up
```

### Web+DBサーバ構築  
1. Ansible実行
  1. 管理ノードへログイン
  ```
    $ ssh root@192.168.100.10
    password: vagrant
  ```
  1. GitHubからPlaybook,アプリを取得
  ```
    # git clone https://github.com/takuya-y/ansible-test.git
    # cd ansible-test/ansible
  ```
  1. 鍵認証による自動ログイン設定
  ```
    # ssh-copy-id root@192.168.100.20
    # ssh-copy-id root@192.168.100.30
  ```
  1. Ansible実行
  ```
    # ansible-playbook -i inventory/hosts master.yml
    <略>

    PLAY RECAP *********************************************************************
    192.168.100.20             : ok=24   changed=19   unreachable=0    failed=0   
    192.168.100.30             : ok=8    changed=7    unreachable=0    failed=0   
  ```

### アプリの動作確認
1. ブラウザからアプリへアクセス
```
  http://192.168.100.20:8080/topic
  username:test1
  password:12345
```
動作確認用のアプリはITmediaのRSSを取得し、情報をDBに保存、  
タイトルをリンクとして画面に表示する。というもの

### 作成してみての振り返り
* Playbookの作成前に一度手動でサーバを構築し、必要な手順を確認してから行った  
  それにより、Playbookの書き方の点に注意すれば良かったので作業がスムーズに行えた
* Ansibleの使い方、Playbookの書き方の基本は学べたので保守性の高い書き方、別システム構築の自動化を行いたい
* Vagrantの使い方も基礎を学べた

### 今後の課題
1. Playbookのリファクタリング  
   roll別での整理、わかりやすい処理の書き方
1. セキュリティを考慮した設定(Ansible実行ユーザ)
1. 環境までの時間短縮
