## Ansibleとは

### 概要
エージェントレス
羃等性

### 環境情報

Controller node
  * CentOS 6.7
  * ansible 2.1.1.0
  * Python 2.6.6

Target node
  * CentOS 6.7
  * Python 2.6.6

### インストール
```
$ yum install ansible -y
```

### ハンズオン
```
$ mkdir /ansible
$ cd /ansible
$ mkdir inventory
$ vi inventory/hosts
[targets]
192.168.100.20

$ ansible all -i inventory/hosts -m ping
192.168.100.20 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Ansibleでできること
* パッケージ操作
httpとrubyのパッケージをyumでinstall,バージョンは最新
```
$ vi test.yml
- name: install packages from yum
  yum: name={{ item }} state=latest
  with_items:
    - httpd
    - ruby
$ ansible-playbook -i inventory/hosts test.yml
  TASK [install packages from yum] ***********************************************
  changed: [192.168.100.20] => (item=[u'httpd', u'ruby'])

  PLAY RECAP *********************************************************************
  192.168.100.20             : ok=4    changed=1    unreachable=0    failed=0   
```

* cron設定
cron設定
```
$ vi test.yml
- name: register cron job
  cron: name="check ping" day="*/2" hour="12" minute="0" job="ping -c 3 192.168.100.10"
$ ansible-playbook -i inventory/hosts test.yml
  TASK [register cron job] *******************************************************
  changed: [192.168.100.20]

  PLAY RECAP *********************************************************************
  192.168.100.20             : ok=5    changed=1    unreachable=0    failed=0   
```
target node
```
$ crontab -l
#Ansible: check ping
0 12 */2 * * ping -c 3 192.168.100.10
```

* ディレクトリ作成
```
$ vi test.yml
- name: create directories
  file: path={{ item.path }} owner={{ item.owner }} group={{ item.group }} mode=0{{ item.mode }} state=directory
  with_items:
    - { "path":"/opt/ansible", "owner":"root", "group":"root", "mode":"755" }
    - { "path":"/opt/vagrant", "owner":"vagrant", "group":"vagrant", "mode":"755" }
$ ansible-playbook -i inventory/hosts test.yml
  TASK [create directories] ******************************************************
  changed: [192.168.100.20] => (item={u'owner': u'root', u'path': u'/opt/ansible', u'group': u'root', u'mode': u'755'})
  changed: [192.168.100.20] => (item={u'owner': u'vagrant', u'path': u'/opt/vagrant', u'group': u'vagrant', u'mode': u'755'})

  PLAY RECAP *********************************************************************
  192.168.100.20             : ok=6    changed=1    unreachable=0    failed=0  
```

* 静的ファイル配置
```
$ vi test.yml
- name: copy files
  copy: src=./files/hoge dest=/opt/ansible/hoge owner=root group=root mode=0755
$ ansible-playbook -i inventory/hosts test.yml
TASK [copy files] **************************************************************
changed: [192.168.100.20]

PLAY RECAP *********************************************************************
192.168.100.20             : ok=7    changed=1    unreachable=0    failed=0   
```

* 動的ファイル配置
```
$ vi test.yml
- name: copy template files
  template: src=./templates/fuga.j2 dest=/opt/ansible/fuga owner=root group=root mode=0755
$ vi templates/fuga.j2
This is a jinja template file.

{{ message }}

jinja template can extract variables. like, ...

{% for key,value in fruits.iteritems() %}
We want {{ value.amount }} {{ key }} !
{% endfor %}
$ ansible-playbook -i inventory/hosts test.yml
TASK [copy template files] *****************************************************
changed: [192.168.100.20]

PLAY RECAP *********************************************************************
192.168.100.20             : ok=8    changed=1    unreachable=0    failed=0   
```
target node
```
$ cat fuga
This is a jinja template file.

Hello Ansible !

jinja template can extract variables. like, ...

We want 10 apples !
We want 30 oranges !
We want 20 bananas !
```
