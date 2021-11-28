---
title: ETSI NFV SOL003の歩き方
emoji:  🦊
type: "tech"
topics: ["仮想化", "NFV", "ETSI", "標準化"]
published: true
---

本ドキュメントは通信業界アドベントカレンダー 2021 の3日目の記事として執筆したものです。

その他の記事はQiitaのアドベントカレンダーページからどうぞ！

https://qiita.com/advent-calendar/2021/telecom


# ETSI GS NFV

ETSI GS NFVはETSI(欧州電気通信標準化機構/エッツィと発音します)で策定されているネットワーク仮想化(NFV)の標準化文章です。

通信関係の標準化団体としてはLTEやIMSや5Gなどで3GPPが有名ですが、3GPPが地域によらない世界的な標準化団体ですが、
ETSIに関してはヨーロッパ、ユーロ圏内向けの電気通信関係の標準化団体です（日本国内で言うところのTTC等）。
しかしなgら、ヨーロッパの標準化団体ではありますが、ETSI NFVの標準化策定には各国のオペレータ、ベンダが参加し議論が行われており、
ヨーロッパ、ユーロ内だけではなく、通信業界全体で利用されています。

https://portal.etsi.org/TB-SiteMap/NFV/NFV-List-members

近年ではゼロタッチオペレーションを目指しているZSMやエッジコンピューティングで有名なMECなどもETSIで策定されています。


# ETSI GS NFV-SOL 003<br>RESTful protocols specification for the Or-Vnfm Reference Point

ETSI GS NFV-SOL 003はGS(Group Specification)と呼ばれるスペックを定めた標準化文書です。

SOL(ソルと発音します)は*Solution WG*の文書であり、 **Protocols and Data Models** を定めています。
実装する際に使用するファンクション間でやり取りをするファイルやAPIのデータモデルや手順などを定めています。

そのため、SOLの文書は実装や商用開発、運用において頻繁に参照することになります。

ETSI NFV-SOL 003 は Or-Vnfm という、NFVO(NFV-Orchestrator)とVNFM(VNF Manager)の間の参照点で使用するRESTful Web APIの仕様を定めた標準化文書です。


:::message
![](https://storage.googleapis.com/zenn-user-upload/7b334d13fefc-20211125.png)
**IFA009より抜粋 赤色部分筆者補足で追記**
:::

VNFのライフサイクル管理をつかさどるVNFMに対して、NFVOからVNFMへ発行されるAPIと、VNFMからNetwork Serviceの管理やNFVI/VIMのリソースを横断的に管理するNFVOに対して発行されるAPIの二つの方向のAPIが定義されています。

SOL003を理解することでNFVにおいて実際にVNFMがデプロイされる手順やパラメータを知ることができよりNFVを身近に感じられることになると思います。

## 本記事のスコープ

今回はVNFのライフサイクルを制御することを中心に、SOL003のどのようなところをで読むべきか、という点にフォーカスして本記事を記載します。

SOL003には後述しますが、いくつかのファンクションが定義されており障害管理や性能管理などのインターフェースも定義されています。
しかしながら、私が携わったことのあるVNFMや実装においてもこれらのファンクションを活用しているような実装を見たことが無いということもあり、
今回この点はあまり触れず、主にライフサイクルマネージメントを中心に内容を紹介したいと思います。


## SOL003に記述されていること

SOL003では、RESTful Web APIを用いて、VNFMの持つファンクションをどのように呼び出すのかということを記載されいます。
ETSI NFV標準ではVNFMの機能の呼び出し方はSOL002/SOL003/SOL005などで示されています。これらの中でもRESTful Web API以外での呼び出し方は定義されておらず、基本的には外部からVNFM等の機能を呼び出すにはこれらの標準で定義されている手順を踏む必要があります。

そのため、VNFMの機能などをETSI NFVで定義されているインターフェース(たとえばOr-Vnfm)を外部や他ベンダのNFVO等に公開する際にはこれらの標準に準拠したやり方で無ければETSI NFVの標準に適合しているとは言えないということになります。

## 対応するIFAドキュメント

ETSI NFV-SOL は ETSI NFV-IFA(アイファと発音します) と呼ばれる同じETSI NFV内の別WG(Interfaces and Architecture WG)で策定された標準化文書をさらに参照・準拠しています。IFAにおいてはNFVのファンクションが持つべき機能であったり、ファンクション内で保持すべき情報、あるいはファンクション間でやり取りすべき情報のモデルを定義しており、SOLではそれを実際のデータモデルにマッピングします。

SOLで定義されているデータモデルなどはIFA上にドキュメントに定義があり、それを実体化しています。

ETSI NFV-SOL003に対しては ETSI NFV-IFA007が主として対応しており、パフォーマンスマネージメント機能の点としてはIFA027が対応しています。

これらの情報は *2. Reference / 2.1 Normative Reference* に記載があります(Normative Referenceは準拠すべき他の標準) 。



## ドキュメントの置き場所

(ETSI内のリンクをたどって一体この場所にどのようにしてたどり着くのかは私は知りませんが、)パブリッシュ済みのドキュメントは以下から参照することができます。

https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/003/


## バージョン

ETSI NFVにおいても 3GPP 等と同じくバージョンやリリースの概念が取り入れられています。

バージョンの差異によりNFVソリューションが持つAPIやデータ構造が異なることがあります。
そのため、自分自身が利用しているVNFMやNFVOといったNFVソリューションがどのバージョンをサポートしているのかを理解しておく必要があります。

2021年12月現在の最新版は2021年7月にリリースされた `3.5.1` です。

以後、実際の例等を提示する際、少し古いですが、今回は筆者自身が良く使用する、OpenStack Tacker(Xena現在)が基本的に参照している v2.6.1 をベースに参照しつつ、バージョンで差異がある部分についてその差異を補足します。

## 文書の構成

:::message
### 2.6.1の章構成
1. Scope
2. Reference
3. Definition of terms, symbols and abbreviations 
4. General aspects
5. VNF Lifecycle Management interface
6. VNF Performance Management interface
7. VNF Fault Management interface
8. VNF Indicator interface
9. VNF Lifecycle Operation Granting interface
10. VNF Package Management interface
11. Virtualised Resources Quota Available Notification interface
12. Annex A (informative): Mapping operations to protocol elements
13. Annex B (informative): Explanations
14. Annex C (normative): VimConnectionInfo registry
15. Annex D (informative): Complementary material for API utilization
16. Annex E (informative): Authors & contributors
17. Annex F (informative): Change History
18. History
:::

文書の構成はバージョンによって多少差があります。

ただ1-3章までの項目は全NFV標準化ドキュメントで変わりません。
1章にはスコープを、2章には参照すべき他の標準化文章等を、3章には用語や略称、記号等を記載します。

SOL003を参照したいというモチベーションがある人はおそらくAPIの引数に何を入れたらいいのか、などといった明確な理由があるか、もしくは興味本位で見る人がいたとしてもどんなAPIが実際に定義されているのかといったことが知りたい方だと思いますので、そのような人にとってはあまりこのあたりの章は重要ではないので、目的が明確な人は読まなくても大丈夫です。

4章以降、Annexより前までが基本的にはメインコンテンツになります。
この4章以降のメインコンテンツの構成は標準化文書により構成は大きく異なります。

SOL003では4章には以下の各章共通の事項を記載しています。5章からインターフェースに関する仕様がきじゅつされ、VNFのライフサイクル管理機能のインターフェースを5章に、6章にはVNFのパフォーマンス管理(性能管理)の機能のインターフェースを、7章には障害管理機能のインターフェース、8章にはインジケータインターフェース、9章から11章まではNFVOと連携するための機能に関するインターフェースが定義されており、9章はグラント機能のインターフェース、10章はVNFパッケージの管理インターフェース、11章はリソースのクォータ管理に関するインターフェースがそれぞれ定義されています。

Annex以降にはExampleや参考となる情報が記載されています。
`normative`と書いているAnnexは準拠すべき内容ですが、`informative`と書いているAnnexは準拠すべき内容ではなく、参考となる情報、実例などが記載されています。

## APIの全体像の把握

SOL003に限りませんが、仕様書を理解する上においてはまず全体像を把握することは大事です。SOL003ではAnnex Aの*Mapping operations to protocol elements*を読むことでそれぞれ定義されているインターフェースのREST APIのリソースと、どちらのファンクションがクライアントで、どちらがサーバなのか(VNFMからNFVOなのか、NFVOからVNFMなのか)を把握することができます。ただし、本Annexには *informative* と記載がありますので仕様書上はあくまでも参考情報であるという点は留意が必要です[^annex_a_info]。

[^annex_a_info]: 基本的には*normative*な本編の内容の整理ではあるので大きく外れているわけではないはずです。

また、APIを見ることにより各NFVO/VNFMの関係性も理解することができるため、NFVの詳細を知らない人がそのファンクションの関連性を知ることもできます。

### VNF パッケージの管理機能に見るVNFMとNFVOの関係性

![](https://storage.googleapis.com/zenn-user-upload/4cbb17a72aa9-20211128.png)

例えば、Annex A.2 には 上記のようなVNFパッケージ管理のインターフェースに関わる仕様が記載されています。
*Direction*に記載がある通り、このSOL003で定義されているパッケージ管理のインターフェースでは、VNFMからNFVOに対して問合せる方向の定義がされていることがわかります。また、VNFMからNFVOに対して、VNFパッケージに関係する操作はGETのメソッドしか用意されておらず、POSTやPUT、DELETEといった操作が定義されていません、これはCRUD操作で言うところのRの読み取りしか基本的には`Or-Vnfm`では許されないということになります。
また、VNFパッケージの操作だけではなく、`Subscribe`、`Query Subscription Information`、`Terminate subscription`、`Notify`といった操作が定義されていることがわかります。 これらの操作は`Notify`だけが、NFVOからVNFMへのリクエストです。このSubscription/NotifyというAPIはETSI NFVでは至る所で定義されています。すでに読者の方はお気づきだと思いますが、これらはVNFMとNFVOが連携するためのAPIであり、NFVO側で起こった変更をVNFMに通知するためのAPIです。例えば、VNFMはNFVOに新たなVNFパッケージが登録されたら自分自身に`Notify`を通知するようにNFVOに設定しておき、新たなVNFのオンボードに備えるといった具合で使用することを想定しています。
このようにETSI-NFVではそれぞれのファンクションが疎結合でかつ、それぞれが必要な情報のみを受け取れるようにということで、いわゆるPub/Subモデルの情報の連携をする仕組みを設けています。

### VNFライフサイクル管理機能の概要

![](https://storage.googleapis.com/zenn-user-upload/2757f5b650b8-20211129.png)

同様に、VNFMの主要な機能であるVNFライフサイクル管理のインターフェースについても簡単に概要を見ておきましょう。
　
このインターフェースは Annex A.6 にて情報が記載されています。他のAnnex Aと見比べてわかる通り、他のインターフェースに比べてもAPIのエンドポイントの数が多いのが見てとれます。また、上記の引用では 3.5.1 と 2.6.1 では定義されたインターフェースも違うのも見てとれます。ここにはバージョンの差異もあるため後程説明します。
本ライフサイクルのインターフェースは全てNFVOからのリクエストであることと、パッケージの管理でも記載したようなPub/Subの仕組みがあることもわかります。

このように基本的な各ファンクションの関係性、どちらのファンクションが何を持つのかといったことを本Annexでは見てとることができるため、初めに読むことを筆者はおススメいたします。

### APIバージョン

上記で軽く触れましたが、SOL003ではリリースによってバージョンに差異があります。

これらのバージョンはAPIエンドポイントのバージョン表記により2021年12月現在、以下のような違いがあります。

| リリース |ドキュメントバージョン|APIバージョン(`{apiMajorVersion}`)| APIベースURLの例 具体例                                               |
|:--------:|:---------------------|:---------------------------------|:----------------------------------------------------------------------|
|2         |`v2`[^v2]             |`v1`                              |`{apiRoot}/vnflcm/v1/vnf_instances`                                    |
|3         |`v3`[^v3]             |`v1`[^v3inv1],`v2` 混在           |`{apiRoot}/vnflcm/v2/vnf_instances`<br>`{apiRoot}/vnfind/v1/indicators`|

[^v2]: 2021年12月現在: `2.3.1` `2.4.1` `2.5.1` `2.6.1` `2.7.1` `2.8.1`
[^v3]: 2021年12月現在: `3.3.1` `3.5.1`
[^v3inv1]: 基本的にはあまり使用されておらず、v2ドキュメントから改版が無かったAPIはそのままAPIバージョンをv1として残しているようです

多少ややこしいのですが、リリース2 の ドキュメントバージョン v2系 の APIバージョンが v1で、リリース3のドキュメントバージョン v3 系 のAPIバージョンがv2と数字が一つズレています。

このようにバージョンを分けることで同一VNFM/NFVOにおいても APIバージョン v1 と APIバージョン v2 を共存させることができるようになっています。現在OpenStack Tackerにおいても APIバージョン v1 と APIバージョン v2 を共存させる形で APIバージョン v2 のインプリメンテーションが進められています。

## インターフェースの読み方

各インターフェースの説明には以下の順番で仕様が記述されています。

1. Description
    - API version
2. Resource structure and methods
3. Sequence diagrams (informative)
4. Resources
5. Data model
6. Handling of Error (一部インターフェースのみ)
7. Handling of security-sensitive attributes (v3ドキュメントの一部インターフェースのみ)

### Description

DescriptionではVNFMがもつ関連インターフェースの一覧が記載されています。これらのリソース毎に基本的には後段の *Resources*が記載されています。

#### API version

APIバージョンは上記に補足した通りです、ここでドキュメントバージョンv3の仕様書ではここで、v2にリバイスしたのかv1を維持したのかが明示されています。

### Resource structure and methods

![](https://storage.googleapis.com/zenn-user-upload/e717ea4fe2b6-20211129.png)

APIのURL表記をツリー構造で図示してくれています。

![](https://storage.googleapis.com/zenn-user-upload/277b18bc7e3e-20211129.png)

またAPIリソースの一覧を表形式で表示しています。Annex Aで表示されている表と近いですが、こちらではその仕様上APIが必須か、オプションかを表示しているカラムである*CAT*カラムと説明用の*Meaning*のカラムが存在します。

### Sequence diagrams

APIのシーケンス図を図示し、API間の相互関係などを読み取ることができます。

![](https://storage.googleapis.com/zenn-user-upload/ba6f4dea475a-20211129.png)

例えば上記はVNFライフサイクル管理における一連のNFVOとVNFMのやり取りを図示しています。

1.においてNFVOから発せられたVNFライフサイクルインターフェースの処理を進むにあたり、どのような通知(`Notify`)がVNFMからNFVOに発せられ、途中の確認処理のためにNFVOがどのようなリクエストを投げてその進捗や状態を確認するのかということが図示されています。

![](https://storage.googleapis.com/zenn-user-upload/e293c33d97d2-20211129.png)

また、そのほかにもPre-condition/Post-conditionという形で事前事後の状態遷移、変更などについても規定されています。

ただ、ここで図示されている内容はあくまでもインターフェース個別のシーケンスだけにはなり、そのほかのインターフェース（たとえばVNFパッケージ管理のインターフェースとVNFライフサイクル管理のインターフェース）がどのように相互作用するのかという点はこのシーケンスだけでは読み取れません。そのようなより広い全体像の定義のために[SOL016](https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/016/)が設けられています。

:::message alert
本シーケンスはSOL003のみならずETSI NFのV標準化仕様の理解を助け、非常に有意義な内容ですが、タイトルにある通り**informative**な内容であることは理解が必要です。必ずしもAPIの呼出しにおいて、本シーケンスに乗っ取った形でなければならない**わけではない**ということです。
:::


### Resources

SOL003ではAPIはRESTful形式を採用しています。そのため仕様書ではリソースを定義し、そのリソースに対して認められている操作やその操作に伴い必要なHTP APIのボディ部のデータ構造（詳細は[Data moelで記載](#data-model)）を定義しています。またAPIの戻り値やレスポンスコードなどが定められています。

まとめると以下の項目が記載されています。

1. **Description**
   APIの概要の記載
2. **Resource definition**
   APIのエンドポイント、URLとそのURL中の変数値が記載されています。
   ![](https://storage.googleapis.com/zenn-user-upload/ce966388ac07-20211129.png)

3. **Resource methods**
   このセクションは具体的なAPIの振る舞いが記載されています。
   このセクションでは大きく二つの表があり、一つがURIパラメータとして認められているパラメータの定義の表であり、
   ![](https://storage.googleapis.com/zenn-user-upload/8adf20a552ac-20211129.png)
   もう一つがBodyおよびレスポンスコードの定義した表です。
   たとえばVNFMが不正なリクエストを受信した場合の動作などもここで記されています。
   リクエスト/レスポンス双方のBodyのデータ構造については後段のData modelで定義がなされています。
   ![](https://storage.googleapis.com/zenn-user-upload/8e2622f2775a-20211129.png)

   これらの情報について、以下のHTTPメソッド毎に記載しており、対応しないメソッドがある場合は非対応であるため405を返せと記載があります。
    1. POST
    2. GET
    3. PUT
    4. PATCH
    5. DELETE

### Data model

SOL001にあったデータモデルの定義と似ています。

SOL003においてはボディはJSONで送るということが[SOL013](https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/013/)で定められており、2章のReferenceで参照すべき標準として定められています。

そのため、SOL003本記載のデータ構造をJSONとして実装し、APIのボディとして組立てます。

![](https://storage.googleapis.com/zenn-user-upload/89b000ec3d8b-20211129.png)

上記引用した `CreateVnfRequest` は非常にシンプルなボディの例です。

この場合以下のようなJSONになります。

```json:CreateVnfRequest
{
    "vnfdId": "UUID for vnfdId",
    "vnfInstanceName": "name of your vnf instance",
    "vnfInstanceDescriptoin": "description of your vnf instance",
    "metadata": {
        "somekey": "somevalue",
        "any_number": 123
    }
}
```

`Cardinarity` がそれぞれ設定されており、`0..1`となっているものは0個以上1個以下ですので、存在するかしないかを選択でき存在する場合は単一のデータです。
対して`0..N`となっているものについては存在しない、もしくは任意の個数の構造を設定できることができるため、これはJSONであればリスト構造として表現します。
また、より複雑なデータ構造の場合は入れ子構造となっており、定義をさらに参照する必要があります。

![](https://storage.googleapis.com/zenn-user-upload/94e2ecae2495-20211129.png)

たとえば `InstantiateVnfRequest` の例においては `ExtVirtualLinkData`、`ExtManagedVirtualLinkData`、`VimConnectionInfo`といったデータは他の場所で定義されたデータ構造を参照する必要があります。`InstantiateVnfRequest`を書いてみると以下のような例となり、`localizationLanguage`などは省略できます（外部参照している構造は省略）。

```json:InstantiateVnfRequest
{
    "flavourId": "sample_flavor_id",
    "instantiationLevelId": "level_min",
    "extVirtualLinks": [
        ...
    ]
    "vimConnectionInfo": [
        ...
    ]
    "additionalParams": {
        "additional": "param",
        ...
    }
}
```

このようにして、SOL003を読み解くことで必要なAPIのリソースを表すURLと、リクエストに与えるボディ等のパラメータを構築することができるようになります。

### Handling of Error<br>(一部インターフェースのみ)

一部のAPIにおいてはAPI発行に対して事前の条件や状態が決まっていることがあります。それらを図示するなどの目的で、一部インターフェースに対してはエラーハンドル等についての言及があります。

![](https://storage.googleapis.com/zenn-user-upload/f1940b6b9f8a-20211129.png)

例えばVNFライフサイクル管理のインターフェースにおいては状態の遷移が発生し、それぞれの状態で呼び出すことができるAPIが決まっています。それを図示するために上記のような状態遷移図などが定義されており、これもまたAPIだけではなくVNFMの機能を理解する上においても非常に役立つ情報となっています。

### Handling of security-sensitive attributes<br>(v3ドキュメントの一部インターフェースのみ)

ドキュメントバージョンv3から追加されています。

具体的にはこれもVNFライフサイクル管理のインターフェースにおいて記載がされており、APIのボディ上にセンシティブな情報(VNFへのログインパスワード情報等)が乗ることがあるという点をフォローする内容となっています。

## 付録,Annexの活用

最後にAnnexの内容について簡単に紹介します。

*Annex A* については冒頭紹介した通り全体像の把握のために非常に役に立つ内容です。*Annex B* についてもAPIの細かな呼出し方について補足が必要となる箇所としてVNFのスケールとネットワークの接続変更を例に例示があります。これらは実際にその動作について知りたい場合については有益な情報です。

*Annex C* はこのAnnexの中では唯一 `normative`な内容です。筆者の認識が正しければ2021年12月現在において ETSI NFVにおける VIM(OpenStackなど)を扱い方が記載された唯一の標準です。

元来SOL003においてもSOL005においても、お互いが認識する VIM というものについて明確に管理する手法が用意され、十分にケアされているように見えません。
これは筆者の推察ですが、ETSIをはじめとした標準化全般でのスタンスとして、オープンソース実装がいくらデファクトスタンダードであるからといって特定のソフトウェアの実装を標準で再定義するということを極力しないようにしていることと[^3gpp_contaner_is_too]、VIMの実装としてはOpenStack以外にもVMwareもありるし、AWSであってもあり得るため、標準化が難しいという側面があるためと考えられます。
とはいえ、NFVの実装上はやはりOpenStackの採用例がおおく、もっぱらデファクトスタンダードである一報、各NEPベンダのNFV製品でそれぞれが好き勝手な値を入れていてはNFVOとVNFM間などでの相互接続が難しいためAnnexの中で、OpenStackという実装ではこのようにしましょうという決まり事として、VIMのケアが唯一なされているようです。

[^3gpp_contaner_is_too]: これはETSI NFVのコンテナ関連の標準化でも、実装としてはもはやK8sしかありえないが、K8sの名前が登場せず、概念だけを再実装しているのにも現れてます。また5Gの標準化なども行っている3GPPでもこの傾向がみられます。

*Annex D*では swagger での実装を ETSIのソースコードリポジトリで公開していることが触れられています。

# むすび

簡単ではありますが、ざっとSOL003で記載されていることがどんなことなのか、どんな点を気にしながら読む必要があるのかを記載しました。

ETSI NFVにおいてはSOLで定義されたAPI以外から、VNFMの機能を呼び出すことを想定していません。そのためSOL003に定義されているAPIを理解することで、合わせてVNFMの持つ動きを理解することができるようになります。また今回は時間の関係からSOL005やSOL002について記述しませんでしたが、基本的にはSOL003と同じ文章構成のため、ここで書いたことをそのまま持って行きSOL002やSOL005も読むことができるはずです。

本内容で少しでもSOL003をはじめ、ETSI-NFVで定義されたAPIの理解につなげていただけるようなことがあれば幸いです。

非常にニッチな領域で、私自身もSOL001、VNFDのスペシャリストというわけではありません。
間違いがあればコメント欄にご指摘下さい。

Have a nice NFV life :)
