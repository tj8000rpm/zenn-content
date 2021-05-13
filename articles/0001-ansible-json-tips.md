---
title: Ansibleでよく使うjson_queryのパターンまとめ。
emoji:  🦊
type: "tech"
topics: ["ansible", "json", "jmespath"]
published: false
---

# `json_query`フィルタ

Ansible-Playbookで、APIから取得した内容や、構造化された
varsファイルをくみあわせたり、特定条件で検索かけたいことは多々あるかと思います。

その際に頻繁に用いるのが[`json_query`](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#selecting-json-data-json-queries)ですが
コマンドラインで用いる同様のjsonパーサである[`jq`](https://stedolan.github.io/jq/)と一見似ているようにみえて
全然似ていないところがあるため中々迷うところがあります。

書いていると、頻繁に登場するパターンを備忘録的に記載しておきます。

なお、`json_query`は上記Ansibleの公式ドキュメントにも記載の通り、
[`jmespath`](https://jmespath.org/examples.html)というpythonライブラリをしようしてますのでそちらもご参考に。


なおパターンは自分でハマったケースを記載しているので、いくつか足していきます。

また表題の記述は可読性を考えて`yaml`で書いてます

## パターン１： 辞書のリストから特定のキーに特定の値を持つ要素を取得する

```yaml:test1.yml
- hosts: localhost
  connection: loacal
  gather_facts: no
  vars:
    test:
      - key: abc
        val: 123
      - key: def
        val: 456
      - key: ghi
        val: 789
  tasks:
  - debug:
      var: test | json_query("[?val==`456`]")
```

```shell-session
$ ansible-playbook test1.yml
PLAY [localhost] **************************************************************************************************

TASK [debug] ******************************************************************************************************
ok: [localhost] => {
    "test | json_query(\"[?val==`456`]\")": [
        {
            "key": "def",
            "val": 456
        }
    ]
}

PLAY RECAP ********************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### 応用例

keyの値のネストが深くても動作します。

```yaml:test2.yml
- hosts: localhost
  connection: loacal
  gather_facts: no
  vars:
    test:
      - key: abc
        val:
          name: Alpha Bravo Charlie
          size: 123
      - key: def
        val:
          name: Delta Echo Foxtrot
          size: 456
      - key: ghi
        val:
          name: Golf Hotel India
          size: 789
  tasks:
  - debug:
      var: test | json_query("[?val.size==`456`]")
```

```shell-session
$ ansible-playbook test2.yml
PLAY [localhost] ***************************************************************************************************

TASK [debug] *******************************************************************************************************
ok: [localhost] => {
    "test | json_query(\"[?val.size==`456`]\")": [
        {
            "key": "def",
            "val": {
                "name": "Delta Echo Foxtrot",
                "size": 456
            }
        }
    ]
}

PLAY RECAP *********************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## パターン２： ネストされた辞書のリストからネストされたキーに特定の値を持つ要素を取得する

やりたいことは先程と一緒なんですが、リストではなく、上記の`key: hoge`になっているっている箇所が辞書のキーとなった場合です。

```yaml:test3.yml
- hosts: localhost
  connection: loacal
  gather_facts: no
  vars:
    test:
      abc:
        val:
          name: Alpha Bravo Charlie
          size: 123
      def:
        val:
          name: Delta Echo Foxtrot
          size: 456
      ghi:
        val:
          name: Golf Hotel India
          size: 789
  tasks:
  - debug:
      var: test | dict2items | json_query("[?value.val.size==`456`]")
```

`dict2items` を用いてパターン１の形に変形してから同じことを実行すればOKです。

```shell-session
$ ansible-playbook test3.yml

PLAY [localhost] ***************************************************************************************************

TASK [debug] *******************************************************************************************************
ok: [localhost] => {
    "test | dict2items | json_query(\"[?value.val.size==`456`]\")": [
        {
            "key": "def",
            "value": {
                "val": {
                    "name": "Delta Echo Foxtrot",
                    "size": 456
                }
            }
        }
    ]
}

PLAY RECAP *********************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

上記例では`dict2items`でデータ構造を変形してしまっているので、元に戻したい時は以下

```yaml:test4.yml
- hosts: localhost
  connection: loacal
  gather_facts: no
  vars:
    test:
      abc:
        val:
          name: Alpha Bravo Charlie
          size: 123
      def:
        val:
          name: Delta Echo Foxtrot
          size: 456
      ghi:
        val:
          name: Golf Hotel India
          size: 789
  tasks:
  - debug:
      var: >-
        test | dict2items | json_query("[?value.val.size==`456`]") |
        items2dict
```

```shell-sesion
$ ansible-playbook test4.yml
PLAY [localhost] ***************************************************************************************************

TASK [debug] *******************************************************************************************************
ok: [localhost] => {
    "test | dict2items | json_query(\"[?value.val.size==`456`]\") | items2dict": {
        "def": {
            "val": {
                "name": "Delta Echo Foxtrot",
                "size": 456
            }
        }
    }
}

PLAY RECAP *********************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
