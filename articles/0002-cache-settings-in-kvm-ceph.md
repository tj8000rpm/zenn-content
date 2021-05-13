---
title: OpenStackでCephを使う場合のディスクキャッシュ設定について
emoji:  🦊
type: "tech"
topics: ["仮想化", "ストレージ", "OpenStack", "KVM", "Ceph"]
published: true
---

# 概要

OpenStack Cinder や Nova のストレージバックエンドに Ceph RBDを利用するケースにおいて、OpenStackやCephの設定ファイルの内容によるディスクキャッシュ実際の振る舞いの違いを記します。

本ドキュメントでは以下を扱います。

- `/etc/nova/nova.conf`と`/etc/ceph/ceph.conf`の設定内容の違いによる実際のディスクキャッシュ動作の振る舞いのまとめ
- 仮想化におけるディスクキャッシュの一般論
- Ceph RBDを使用した場合の違い

本ドキュメントでは以下のことは**扱いません**。

- 仮想化とは
- OpenStackとは
- cpehとは
- QEMU/KVMとは

## 対象とするバージョン等

| ソフトウェア | バージョン |
| :----------- | :--------- |
| OpenStack    | Train      | 
| Ceph         | Nautilus   | 
| Linux Kernel | 4.18.0     | 
| libvirt      | 6.0.0      |
| QEMU         | 4.2.0      |

# はじめに

## Cephを利用した場合のキャッシュ設定の振舞いがわからぬ 

OpenStack において Ceph RBD を Cinder のブロックストレージのバックエンドに使用するのはよく知られた構成です。

しかしながらこのときに**ハイパーバイザ上のディスクキャッシュ設定**がどのように振る舞うのか、どの設定ファイルをどのように変更すれば期待する動きになるのかを明確に記載した文書は、英語文献も含めて十分に（かつ平易に）説明している記事があるわけではありません[^noarticle]。
[^noarticle]: 筆者調べ(2021-05)。

さらに、今回調べた結果、かなり特殊な動き方（設定の「言葉通り」の動作にならない）ということがわかりました。

諸般の事情からこれら設定がどのように動作し、そしてどのように実際に振る舞うのかを確認する必要があったため、各種公式ドキュメント、解説記事、ソースコードを確認した結果を記載します。

# 設定内容の違いによる実際のディスクキャッシュ動作の振舞い

:::message
**忙しい方のために**
どのように動くのかの結論だけ知りたい方は本章だけ読んでいただくで問題ありません。
本章には「なぜ」の部分はほとんど記載しませんので「なぜ」が知りたい方は次章以降も参照してください。
:::

## 扱う設定ファイル

以下に関係する設定ファイルと設定内容を記載します。これらの値がどのように有効化、無効化され、そして実際に振る舞うのかを以下の表にまとめています。

### OpenStack Novaの設定[^novaconf]

[^novaconf]: https://docs.openstack.org/nova/train/configuration/config.html#libvirt.disk_cachemodes

```ini:/etc/nova/nova.confの設定例（抜粋）
[libvirt]
disk_cachemodes = network=none
```

### Ceph の設定[^cephconf]

[^cephconf]: https://docs.ceph.com/en/nautilus/rbd/rbd-config-ref/#cache-settings

```ini:/etc/ceph/ceph.confの設定例（抜粋）
rbd_cache = true
rbd_cache_max_dirty = 24
rbd_cache_writethrough_until_flush = true
```

## 設定の振舞い

| # | Novaの設定             | ハイパーバイザでのキャッシュ                                     | ゲスト側へのキャッシュモード通知[^notecache] | 備考                       |
|--:| :--------------------- | :--------------------------------------------------------------- | :------------------------------------------- | :------------------------- |
| 1 | `network=none`         | `ceph.conf/rbd_cache = true`に関係なくキャッシュされない         | `write back`                                 |                            |
| 2 | `network=writeback`    | キャッシュするが実際の動作モード[^cache_mode]は`ceph.conf`による | `write back`                                 |                            |
| 3 | `network=writethrough` | キャッシュするが実際の動作モードは`ceph.conf`による              | `write through`                              |                            |
| 4 | `network=directsync`   | `ceph.conf/rbd_cache = true`に関係なくキャッシュされない         | `write through`                              |                            |
| 5 | `network=unsafe`       | キャッシュするが実際の動作モードは`ceph.conf`による              | `write back`                                 | ゲストからのsyncは無視する |
[^notecache]: `$cat /sys/block/vdb/cache_type`で表示される
[^cache_mode]: writeback or writethrough

上記の通り、Novaで設定するに動作モードによりそれぞれ異なる振舞いをします。しかしながら実際にハイパーバイザのディスクキャッシュの振舞いがNovaの設定した意味通りになるわけではありません。すなわち、`writethrough`と書いたあっても実際に`writethrough`にはならないし、`writeback`と書いてあっても`writeback`にはなリません。

そして`ceph.conf`の`rbd_cache`値はいずれの場合でも無視され、Novaの設定に基づいてオーバライドされます。すなわち `network=none`または`network=directsync`の場合は`rbd_cache=false`として扱われ、`network=writeback`、`network=writethrough`、`network=unsafe`の場合は`rbd_cache=true`として扱われます。

`writethrough`として動作するかどうかは`ceph.conf`に設定された`rbd_cache_max_dirty`の値により決まります。具体的な設定値による振舞いの違いについては次のまとめの中で表にまとめます。


## 結論のまとめ

以上からまとめると次のようになります。自分自身の設定したい設定意図に従い、`nova.conf`と`ceph.conf`をカスタマイズします。


### Novaの設定

`nova.conf`の設定により以下の振舞いが決まります。

1. `rbd_cache` を有効化するかどうか
1. ゲストへキャッシュの有無を通知どうか
1. ゲストからのflushを無視するかどうか

このうち *ゲストからのflushを無視するかどうか* については特殊な用途なため以下表からは除外します[^unsafe2]。

|                                          | `rbd_cache`の有効化    |`rbd_cache`の無効化   |
| :--------------------------------------- | :--------------------- | :------------------- |
| ゲストへ`cache_mode=write back`と通知    | `network=writeback`    | `network=none`       |
| ゲストへ`cache_mode=write through`と通知 | `network=writethrough` | `network=directsync` |
[^unsafe2]: 実際には`network=unsafe`もこれに含まれますが、以降に記載の通り`unsafe`は通常用いられるべきではないためここでは除外して記載します。

### Cephの設定

Novaの設定において`network=writeback`または`network=writethrough`を選択した場合、`ceph.conf`の設定により以下の振舞いが決まります。

1. キャッシュモードとして`writethrough`にするか、`writeback`とするか
1. `writeback`モードの際にゲストVMからの最初のflashまでは`writethrough`として動作させるか

| `rbd_cache_max_dirty` | `rbd_cache_writethrough_until_flush` | 実際の振舞い                                                                 |
| :-------------------- | :----------------------------------- | :--------------------------------------------------------------------------- |
| `x > 0`               | `true`                               | `writeback`として動作、ただし最初のflushがあるまでは`writethrough`として動作 |
| `x == 0`              | `true`                               | `writethrough`として動作                                                     |
| `x > 0`               | `false`                              | `writeback`として動作                                                        |
| `x == 0`              | `false`                              | `writethrough`として動作                                                     |

### 参考:設定の考え方

どういった値を撮るのが良いのかがわからないという場合、一般論として以下に設定値の例を記載します。

総論としては

- `nova.conf`では`disk_cachemodes = network=none`を設定するのが安全とパフォーマンスをある程度のバランスで両立できる
- ディスクに確実に書き込んだかどうかは完全にVMの責任であり、性能を最大限に発揮させたいなら`nova.conf`には`disk_cachemodes = network=writeback`を設定し、`ceph.conf`では`rbd_cache_max_dirty > 0` で設定
- 確実にデータを書き込んだことをゲストVMではなくハイパーバイザが責任を負わなければならない場合は`disk_cachemodes = network=writethrough`を設定し、`ceph.conf`では`rbd_cache_max_dirty = 0`を設定

![フローチャート](https://storage.googleapis.com/zenn-user-upload/fnewnbxk0lkercmq0a8wcmtb63bd)
*made by: https://app.diagrams.net/*

- `rbd_cache_writethrough_until_flush`については、`writeback`を意図した場合のみ気にすれば良い。特殊なゲストOSを使用しない限りは特別気にする必要はないが、`true`としておくこと方が安全。ただし、書きはじめだけ性能が劣化するため性能のムラが起こることを避けたいのであれば`false`にする

# 仮想化におけるディスクキャッシュの一般論

## 仮想化におけるディスクキャッシュの全体像

仮想環境におけるディスクキャッシュは少々複雑ですが、基本的にはLinuxのページキャッシュ、バッファーキャシュの仕組みおよび、ストレージディスクやRAIDコントローラの持つのキャッシュシステムの仕組みをそのまま利用します。

下図左側、`(a)`のに示すとおり、一般的にはキャッシュ構造をネストで形成します。これを上から順に見ていくと、

1. ゲストVM上でのページキャッシュ(含バッファキャッシュ)
1. ハイパーバイザ上でのページキャッシュ（含バッファキャッシュ）
1. 物理ストレージデバイスでのハードウェアキャッシュ

以上の3つの箇所でキャッシュされる可能性を示しています。もちろん、ゲストVM、ハイパーバイザでの設定や使用するハードウェアやストレージシステムにより、これらのキャッシュは存在したりしなかったりします。

しかし、ゲストVMから見た場合には、`(b)`のようにハイパーバイザ以下がすべて仮想ディスクデバイスのように見え、ハイパーバイザの中でどのようなキャッシュ保持をしているのかをうかがい知ることは基本的にできません。同様に`(c)`のハイパーバイザから見た場合にも、ゲストVMがどのようにファイルシステムを管理し、どのようにキャッシュを持っている（もしくは持っていないのか）を知ることはできません。

![仮想化におけるディスクキャッシュ](https://storage.googleapis.com/zenn-user-upload/cwparqbc7r4xaolno4s6ha24x5c7)
*仮想化におけるディスクキャッシュ*

## ネストされたキャッシュの持つ特徴

このようにネストされた構造はいくつかの特徴や欠点が有りますが、もっとも考慮すべき点はハイパーバイザおよびゲストVM双方で同じデータをキャッシュするということです。
通常VMの仮想ブロックデバイスはイメージファイルと呼ばれるqcow2などファイルとしてハイパーバイザ上では扱われます。
そして、そのイメージファイルはハイパーバイザ上のファイルシステム上に配置され、VMのプロセス（例えばQEMU）によってファイルとして開かれることになります。

Linuxでは通常ファイルをオープンする際にはファイル領域を一旦ホストページキャッシュに読込み、アプリケーションはページキャッシュに読み込まれたメモリ領域を参照することで効率的なファイルアクセスを実現します。

ゲストVMでも当然同様の処理が行われます。ゲストVM上でファイルをオープンした結果、仮想ディスク上のある領域を（ファイルシステム経由で）読込み、その結果をゲストページキャッシュに読み込みます。このとき、当然ながらホストVM上のVMイメージファイルの該当する箇所をハイパーバイザ側では読込み、ホストページキャッシュに読み込んだ上でゲストVMの仮想ストレージデバイスの読込み結果としてゲストVMに返却しています。

この際にキャッシュされる、ゲストページキャッシュとホストページキャッシュは、同じハイパーバイザの物理メモリ上に確保された領域では有りますが、それぞれ別のメモリアドレスを持つことになります。

![メモリに確保された重複するデータ](https://storage.googleapis.com/zenn-user-upload/e5c5r77jr7nmnfibovvb9ftaxtw9)
*メモリに確保された重複するデータ*

これは以下の特徴が有ります。

+ メリット
    + ゲストVMでキャッシュが消えてもハイパーバイザ側にキャッシュが残っていればアクセスが高速化できる
    + ゲストからの要求をハイパーバイザが一旦吸収できるためゲストのIOとは関係のない、ハイパーバイザの効率の良いタイミングで実際の物理ディスクにアクセスができる。
- デメリット
    - 同一データに対してメモリが少なくとも2倍必要になる(容量の問題)
    - キャッシュに書く、コストが発生する（レイテンシの問題）
* その他の特徴（メリットともデメリットもつかないもの）
    * キャッシュを実際にフラッシュ（ストレージデバイスへの書出し）を行うタイミングがゲストとホストでは異なるためIOのタイミングが複雑化する

## 仮想化でのキャッシュの選択

これらのネストされた構造でのキャッシュ機構に対して、いくつかのアプローチが有ります。

その一つの軸が積極的にキャッシュを使ってリードもしくはライトの性能を向上しようという軸で、もう一つが積極的にデータの信頼性を向上しようという軸です。

そこでlibvirtにはこれらのディスクキャッシュの振舞いを設定するためのパラメータがいくつか用意されています。代表的なものとして4つ有り、それぞれ`writethrough`、`writeback`、`directsync`、`none`が用意されています。

その他の仕組みとして`unsafe`という仕組みも用意されていますが、こちらはVMからのディスクの同期命令を無視するという、極端に書込性能を向上させるための仕組みでありlibvirtのドキュメント上でも明確に特別な理由が無い限りは使用しないよに明記されていますのでここでは割愛します。


|                                | データの信頼性をケアする(`O_SYNC`) | ケアしない |
|:-------------------------------|:-----------------------------------|:-----------|
| **性能をケアする**             | writethrough                       | writeback  |
| **ケアしない(`O_DIRECT`)**     | directsync                         | none       |

![libvirtで選択できるキャッシュ選択](https://storage.googleapis.com/zenn-user-upload/kqq51i0w1h8tfi373hq5mwq84rqi)

ここで使用されるのが、 `O_DIRECT` と `O_SYNC` の考え方です。
通常Linuxでは`O_DIRECT`フラグを付与してファイルをオープンした場合、ページキャッシュを使用せずにファイルを読み込みます。`O_SYNC`フラグを付与した場合はライト命令に対して常にディスクへの同期を実施します。
この仕組みをハイパーバイザの中で仮想イメージファイルを開く際にとりいれることにより、図、表に示した通り、ホストキャッシュを利用したり、ハイパーバイザが書込命令を受け取った際にストレージディスクデバイスに対して同期を都度実施するかなどの振舞いを決めることができます。

これにより、ハイパーバイザでのディスクアクセススピードの向上を計ったり、ストレージディスクデバイスへのアクセス回数を減らしたり、あるいは信頼性を向上させることができるようなります。

## 仮想化でのディスクキャッシュ設定の伝播

さて、上記に記載したとおり、libvirtでは`writethrough`、`writeback`、`directsync`、`none`といったディスクキャッシュモードをハイパーバイザとしてVMに設定することができます。
libvirtはVMの設定をXMLの設定ファイルで記載するなど、人間が管理しやすい形で、QEMU/KVM用のゲストVMプロセスを生成するためのライブラリ、ソフトウェアです。

そのため、上記ディスクキャッシュ設定、`writethrough`、`writeback`、`directsync`、`none`は最終的にはQEMU/KVMのためのコマンドライン引数の形で展開されます。

ここではその展開の仕方を説明します。

まずlibvirtではqemu-kvmに渡す引数に変換する前に`writethrough`、`writeback`、`directsync`、`none`のそれぞれ(+`unsafe`)を、`writeback`、`direct`、`no-flush`の3つの値の組み合わせに変換します。

|              |`writeback`|`direct`|`no-flush`|
|:-------------|:----------|:-------|:---------|
|`writethrough`| no        | no     | no       |
|`writeback`   | yes       | no     | no       |
|`directsync`  | no        | yes    | no       |
|`none`        | yes       | yes    | no       |
|(`unsafe`)    | yes       | no     | yes      |


そしてこれらのパラメータは最終的に qemu-kvm においては以下のようなコマンドライ引数に展開されます。これは`ps`コマンドなどで動いているVMのプロセスに与えられた引数を確認することで見ることができます。

```
/usr/bin/qemu-system-x86_64
...
-blockdev {... "cache":{"direct":true,"no-flush":false}, ...}
-device ...,write-cache=on
...
```

QEMUプロセスはこれらのオプションパラメータに与えられた値を見て、指定されたブロックデバイスドライバー毎にファイルのオープンフラグ等を制御します。しかしながらここで注意が必要なのが、あくまでもQEMUの用いる仮想ストレージのタイプ、ドライバによりその動きは変わってくるということです。
すなわち、これらの`write-back=true`、`cache.direct`、`cache.no-flush`といった値が、今回の本題である、Ceph RBDを用いた仮想ブロックデバイスのドライバでどのように動作するのかというのは必ずしも自明ではありません。そして、結論からいえばCeph RBDの場合通常のイメージファイルをオープンするときとは違う動きをするということです。

## Ceph RBDのキャッシュ機構とQEMU設定

まずはじめにQEMUにてCeph RBDを用いて仮想ストレージデバイスを用いる場合には先程来記載している、「ホストキャッシュ」は使用されません。
これはCeph RBDをQEMUから使用する際に`librbd`と呼ばれるユーザランドで動作するライブラリを用いるためです。

そのため、カーネルレベルで保持するページキャッシュ機構を用いる事ができず、その代替としてRBDキャッシングという独自のキャッシュ機構をlibrbd内で提供しています[^librbd_cache]。
[^librbd_cache]: https://docs.ceph.com/en/nautilus/rbd/rbd-config-ref/#cache-settings

これは完全にハイパーバイザにおけるカーネルレベルのページキャッシュに代わるものであり、QCOW2イメージをローカルディスクから読み込みホストページキャッシュを用いる場合と比較して次のように図示することができます。

![Ceph RBDのキャッシュスタック](https://storage.googleapis.com/zenn-user-upload/izyu9nuhsgnnmy4sgd9ln4z87edk)
*Ceph RBDのキャッシュスタック*

そして、キャッシュより先にはlibrbd/libradosと呼ばれるCephのライブラリ実装があり、ライブラリ実装を通じて、Ceph OSDのクラスタにネットワーク越しにアクセスします。

また、今回はこれ以上触れませんが、Ceph OSD上で動作するCeph独自のファイルシステムであるBluestoreにもBluestore キャッシュと呼ばれる機構があり、更に場合によってはCeph OSDサーバ上のRAIDコントローラなどにもキャッシュメモリがついている場合も考えられるなど、複数の箇所でキャッシュが働く仕組みとなっています。

また、RBD cacheではキャッシュの有効化か無効化しか設定しません(Nautilusの場合)。そのため基本的にはライトバックとして動作します。
ライトスルーとして動作させたい場合はキャッシュ中にダーティなキャッシュをどれだけ許容するのかを設定する`rbd_cache_max_dirty`という設定値を`0`に設定することでライトスルーとして動作させることになります。

さて、ホストページキャッシュの代わりとしてlibrbdのRBDキャッシングが用いられるということを説明しましたが、QEMUに設定されたキャッシュのパラメータはVMに対してどのように働くのでしょうか。
この点をもう少し深掘って行きたいと思います。なお前提としてはVMが使用するドライバーとしては`virtio_block`デバイスを用いる前提で記載（調べている）のでその他の何かしらのエミュレータを用いることができる場合はこの限りではないかもしれない点についてのみ先に記載しておきます。

QEMUに設定されたパラメータがRBD上どう設定されていくかは [QEMUのRBDドライバーのソースコード](https://github.com/qemu/qemu/blob/v6.0.0/block/rbd.c)に記載が有ります。

該当する関数は[`static int qemu_rbd_connect()`](https://github.com/qemu/qemu/blob/v6.0.0/block/rbd.c#L544-L547)に記載されています。 

```c:rbd.c#544-547
static int qemu_rbd_connect(rados_t *cluster, rados_ioctx_t *io_ctx,
                            BlockdevOptionsRbd *opts, bool cache,
                            const char *keypairs, const char *secretid,
                            Error **errp)
```

まず関数を確認すると以下の通り、`/etc/ceph/ceph.conf`の読み取りを行っています。

```c:rbd.c#L576-L577
    /* try default location when conf=NULL, but ignore failure */
    r = rados_conf_read_file(*cluster, opts->conf);
```

通常、設定ファイルに記載していないパラメータについては設定項目のデフォルト値が利用されます。
ここで、ハイパーバイザに設定されている`/etc/ceph/ceph.conf`に明示的に記載した内容でその設定値がオーバライドされていることがわかります。

次に、`qemu_rbd_connect()`関数の引数にある`cache`のブール値を評価し、真の場合は`rbd_cache`を`true`とし、偽の場合は`false`としてオーバライドします。

```c:rbd.c#L600-L611
    /*
     * Fallback to more conservative semantics if setting cache
     * options fails. Ignore errors from setting rbd_cache because the
     * only possible error is that the option does not exist, and
     * librbd defaults to no caching. If write through caching cannot
     * be set up, fall back to no caching.
     */
    if (cache) {
        rados_conf_set(*cluster, "rbd_cache", "true");
    } else {
        rados_conf_set(*cluster, "rbd_cache", "false");
    }
```

この`cache`引数については以下の箇所で設定のうえ、`qemu_rbd_connect()`関数が呼ばれています。

```c:rbd.c#L742-743
    r = qemu_rbd_connect(&s->cluster, &s->io_ctx, opts,
                         !(flags & BDRV_O_NOCACHE), keypairs, secretid, errp);

```

これは`qemu_rbd_open()`関数内で定義されており、RBDブロックデバイスを利用をはじめる際に呼ばれる関数です。
ディスクドライバへの設定事項を定義した`int flag` パラメータのうち、`BDRV_O_NOCACHE` フラグが立っている場合`true`となります。

さらに、この`flag`はこのRBDドライバの呼び出し元である[`qemu/block.c`](https://github.com/qemu/qemu/blob/v6.0.0/block.c#L1425-L1444)の、
`int update_flags_from_options()`内で`opts`の引数に従い、設定されていることがわかります。

```c:block.c#L1425-L1444
static void update_flags_from_options(int *flags, QemuOpts *opts)
{
    *flags &= ~(BDRV_O_CACHE_MASK | BDRV_O_RDWR | BDRV_O_AUTO_RDONLY);

    if (qemu_opt_get_bool_del(opts, BDRV_OPT_CACHE_NO_FLUSH, false)) {
        *flags |= BDRV_O_NO_FLUSH;
    }

    if (qemu_opt_get_bool_del(opts, BDRV_OPT_CACHE_DIRECT, false)) {
        *flags |= BDRV_O_NOCACHE;
    }

    if (!qemu_opt_get_bool_del(opts, BDRV_OPT_READ_ONLY, false)) {
        *flags |= BDRV_O_RDWR;
    }

    if (qemu_opt_get_bool_del(opts, BDRV_OPT_AUTO_READ_ONLY, false)) {
        *flags |= BDRV_O_AUTO_RDONLY;
    }
}
```

`BDRV_OPT_CACHE_DIRECT`の値は[include/block/block.h](https://github.com/qemu/qemu/blob/v6.0.0/include/block/block.h#L130)内で定義を見ることができ、
引数に与えられたjsonをパースしている事がわかります。

```c:include/block/block.h#L130
#define BDRV_OPT_CACHE_DIRECT   "cache.direct"
```

紙面の関係もあるためこれ以上の深堀は行いませんが、このように、`cache.direct`が指定された場合、`flag`に`BDRV_O_NOCACHE`が設定され、
`BDRV_O_NOCACHE`が設定されている場合`rbd_cache=false`となり、設定されていない場合`rbd_cache=true`となることがわかりました。

`block/rbd.c`に戻って見ると RADOSの接続である`rados_connect()`を行い、`qemu_rbd_open()`にて`rbd_open()`に進んで接続していることがわかります。

```c:block/rbd.c#L613
    r = rados_connect(*cluster);
```

```c:block/rbd.c#L752
    r = rbd_open(s->io_ctx, s->image_name, &s->image, s->snap);
```

また、この他に、リポジトリ全体にて`rbd_cache`等の文字列でgrepをかけてみたところ、`rbd_cache`のRBDパラメータが他の箇所で設定されていることが無いこともわかりました。

以上の結果から、QEMUからRBDを使用する場合は以下の通りコンフィグを読み込んでいることがわかります。

1. librbdのデフォルト設定を読み込む
1. `/etc/ceph/ceph.conf`に明示的に設定されている値で上書きする
1. `cache.direct == false` の場合は `rbd_cache=true`で、`cache.direct == true`の場合は`rbd_cache=false`でそれぞれ上書きする

このことから、QEMUにて設定された `write-cache=on`という`writeback`なのか、`writethrough`なのかを示すパラメータは用いられていないことがわかります。
そしてRBDキャッシングの中で説明した`rbd_cache_max_dirty`の設定値も変更されていないことがわかります。

librbdのRBDキャッシングを用いる上で`writethrough`として動作する、`writeback`として動作するの振舞いを決めるはずの`rbd_cache_max_dirty`に変化が生じていないということから、
QEMU、あるいはLibvirt、あるいはOpenStack Novaで設定された`writeback`と`writethrough`はCeph RBDを持ちる場合では本質的には意味をなさないということがわかります。

ここに、冒頭記載した結論を導き出すことができます。

# 最後に

さて以上が、Nova、Libvir、QEMU、Librbd(Ceph)を通したキャッシュ設定の伝播の流れでした。

正直このような設定になっているのは想定していなかったためはじめは一生懸命`rbd_cache_max_dirty`を書き換えている人がいるはずだという前提でありとあらゆるところを探していました。
しかしそのような人を見つけることができず、立ち止まって考えたときに、もしかしてこれは誰も更新していないということではないのか？とふと気づいたというのが実際のところです。
ですのではじめからこのきれいな流れを理解できたわけでは無かったためかなり設定内容を理解するのに時間がかかりました。

こういった点（おそらく）あまり理解されて、あるいは気にして使っていらっしゃる方も少ないのかなという思いも有り、
また技術的にはトリッキーな動きをしているという思いもあったため、今回一連の経緯やサマリも含めて、調べて理解した内容を記載したいと思います。

私も相当のど素人ですので読み違えている、あるいは見落としているファクタがあるかもしれません。
なにか重大な誤りや間違いがありそうでしたらコメント、あるいはTwitterなどでご意見いただければ幸いです。
