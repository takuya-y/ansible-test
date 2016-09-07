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
  # -*- mode: ruby -*-
  # vi: set ft=ruby :

  Vagrant.configure(2) do |config|
    config.vm.define "web" do |node|
          node.vm.box = "centos6.7"
          node.vm.hostname = "web"
          node.vm.network :private_network, ip: "192.168.100.20"
          node.vm.network :forwarded_port, id: "ssh", guest: 22, host: 2220
    end
    config.vm.define "database" do |node|
          node.vm.box = "centos6.7"
          node.vm.hostname = "database"
          node.vm.network :private_network, ip: "192.168.100.30"
          node.vm.network :forwarded_port, id: "ssh", guest: 22, host: 2230
    end
    config.vm.define "controller" do |node|
          node.vm.box = "centos6.7"
          node.vm.hostname = "controller"
          node.vm.network :private_network, ip: "192.168.100.10"
          node.vm.network :forwarded_port, id: "ssh", guest: 22, host: 2210
          node.vm.provision :shell, :inline => "yum install ansible -y"
          node.vm.provision :shell, :inline => "mkdir /root/ansible"
          node.vm.provision :shell, :inline => "ssh-keygen -t rsa -f /root/.ssh/id_rsa -N ''"
    end
  end
```
IP Address  
管理サーバ: 192.168.100.10  
webサーバ: 192.168.100.20  
DBサーバ: 192.168.100.30  

管理サーバでansibleのインストールと作業用ディレクトリ、鍵の作成まで行っている。  

1. Vagrantを実行しVMをデプロイ
```
  $ vagrant up
```

1. Ansible実行
  1. 管理ノードへログイン
  ```
    $ ssh root@192.168.100.10
    password: vagrant
  ```
  1. GitHubからPlaybook,アプリを取得
  ```
    $
  ```
  1. 鍵認証による自動ログイン設定
  ```
    $ ssh-copy-id root@192.168.100.20
    $ ssh-copy-id root@192.168.100.30
  ```
  1. inventoryファイル作成
  ```
    $ mkdir inventory
    $ vi inventory/hosts

    [dbserver]
    192.168.100.30

    [webserver]
    192.168.100.20
  ```

  1. Playbook作成
  ```
    - hosts: dbserver
    user: root
    tasks:
      - name: package install
        yum: name={{ item }} state=latest
        with_items:
          - mysql
          - mysql-server
          - mysql-libs
          - mysql-devel
          - MySQL-python

      - name: upload my.cnf
        copy: src=./files/my.cnf dest=/etc/my.cnf owner=root group=root mode=0644

      - name: restart mysql
        service: name=mysqld state=restarted enabled=yes

      - name: create db
        mysql_db: name=webapp state=present

      - name: create user for app
        mysql_user: name=webapp password=webapp priv=webapp.*:ALL,GRANT state=present host=192.168.100.20

      - name: create user for app
        mysql_user: name=webapp password=webapp priv=webapp.*:ALL,GRANT state=present host=192.168.100.40

      - name: delete default user
        mysql_user: name="" state=absent host=database

      - name: delete default user
        mysql_user: name="" state=absent host=localhost

    - hosts: webserver
      user: root
      tasks:
        - name: pakage install
          yum: name={{ item }} state=latest
          with_items:
            - httpd
            - libcurl-devel
            - mysql-devel
            - git
            - readline-devel
            - httpd-devel
            - unzip

        - name: Install rbenv
          git: repo=https://github.com/sstephenson/rbenv.git dest=~/.rbenv

        - name: Add rbenv to PATH
          lineinfile: >
            dest="~/.bash_profile"
            line="export PATH=$HOME/.rbenv/bin:$PATH"

        - name: Eval rbenv
          lineinfile: >
            dest="~/.bash_profile"
            line='eval "$(rbenv init -)"'

        - name: Install ruby-build
          git: repo=https://github.com/sstephenson/ruby-build.git dest=~/.rbenv/plugins/ruby-build

        - name: Check if version is installed ruby
          shell: "source ~/.bash_profile; rbenv versions | grep 2.3.1"
          register: rbenv_check_install
          changed_when: False
          ignore_errors: yes

        - name: Install ruby
          shell: "source ~/.bash_profile; rbenv install 2.3.1"
          when: rbenv_check_install|failed

        - name: Check if version is the default ruby version
          shell: "source ~/.bash_profile; rbenv version | grep 2.3.1"
          register: rbenv_check_default
          changed_when: False
          ignore_errors: yes

        - name: Set default ruby version
          shell: "source ~/.bash_profile; rbenv global 2.3.1"
          when: rbenv_check_default|failed

        - name: Install rails and bundler
          gem: name={{ item }} executable=.rbenv/shims/gem user_install=False
          with_items:
            - rails
            - bundler
            - passenger

        - name: upload httpd.conf
          copy: src=./files/httpd.conf dest=/etc/httpd/conf/httpd.conf owner=root group=root mode=0644

        - name: upload httpd.conf
          copy: src=./files/passenger.conf dest=/etc/httpd/conf.d/passenger.conf owner=root group=root mode=0644

        - name: find passenger module
          stat: path=/root/.rbenv/versions/2.3.1/lib/ruby/gems/2.3.0/gems/passenger-5.0.30/buildout/apache2/mod_passenger.so
          register: module

        - name: passenger-module should be installed
          shell: "source ~/.bash_profile; rbenv rehash;passenger-install-apache2-module --auto --languages=ruby"
          when: module.stat.exists != True

        - name: restart httpd
          service: name=httpd state=restarted enabled=yes

        - name: upload app
          copy: src=./files/rssreader.zip dest=/home/vagrant/rssreader.zip owner=root group=root mode=0644

        - name: find app
          stat: path=/home/vagrant/rssreader
          register: appdir

        - name: unzip app
          command: "unzip /home/vagrant/rssreader.zip -d /home/vagrant/"
          when: appdir.stat.exists != True

        - name: bundle install
          bundler: state=present chdir=/home/vagrant/rssreader executable=/root/.rbenv/versions/2.3.1/bin/bundle

        - name: db migrate
          shell: "source ~/.bash_profile;cd /home/vagrant/rssreader; rake db:migrate"

        - name: change mode vagrant dir
          file: path=/home/vagrant owner=vagrant group=vagrant mode=0701

        - name: change mode rssreader dir
          file: path=/home/vagrant/rssreader owner=vagrant group=vagrant recurse=yes

        - name: change mode root dir
          file: path=/root mode=o+x
    ```

  1. Ansible実行
  ```
    $ ansible-playbook inventory/hosts master.yml
    <略>

    PLAY RECAP *********************************************************************
    192.168.100.30             : ok=9    changed=1    unreachable=0    failed=0   
    192.168.100.40             : ok=20   changed=2    unreachable=0    failed=0   
  ```

### 作成してみての振り返り
* Playbookの作成前に一度手動でサーバを構築し、必要な手順を確認してから行った  
  それにより、Playbookの書き方の点に注意すれば良かったので作業がスムーズに行えた
* Ansibleの使い方、Playbookの書き方の基本は学べたので保守性の高い書き方、別システム構築の自動化を行いたい
* 実行速度の測定
* Vagrantの使い方も基礎を学べた

### 今後の課題
1. Playbookのリファクタリング  
   roll別での整理、わかりやすい処理の書き方
1. セキュリティを考慮した設定
