---
title: ETSI NFV SOL001の歩き方
emoji:  🦊
type: "tech"
topics: ["仮想化", "NFV", "ETSI", "標準化"]
published: true
---

本ドキュメントは通信業界アドベントカレンダー 2021 の1日目の記事として執筆したものです。

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


# ETSI GS NFV-SOL 001<br>NFV descriptors based on TOSCA specification

ETSI GS NFV-SOL 001はGS(Group Specification)と呼ばれるスペックを定めた標準化文書です。

SOL(ソルと発音します)は*Solution WG*の文書であり、 **Protocols and Data Models** を定めています。
実装する際に使用するファンクション間でやり取りをするファイルやAPIのデータモデルや手順などを定めています。

そのため、SOLの文書は実装や商用開発、運用において頻繁に参照することになります。

ETSI NFV-SOL 001 は VNFDやNSDと呼ばれる ディスクリプタ(Descriptor)をTOSCAで記述するための標準化文書です。

TOSCAは(Topology and Orchestration Specification for Cloud Applications)の略称で、OASISというこれまた別の標準化団体で策定されています。
簡単に言えば AWS Cloud Formation や OpenStack HeatのようなクラウドアプリケーションをYAML形式のIaCとして表現する手法です。

ETSI NFVはこのTOSCAによる記述を採用しています(そのほかにもYANGで記述するETSI NFV標準もあります)。

## 本記事のスコープ

今回はVNFDやNSDを記述する際などに、NFV標準でどのようなところをで読むべきか、という点にフォーカスして本記事を記載します。
なお、TOSCA形式でのVNFD等の作成方法やVNFDの構造については本記事ではかいせつしません。こちらについてはまた別の機会に紹介したいと思います[^caution_for_what_is_vnfd]。

[^caution_for_what_is_vnfd]: 余裕があれば・・・

## SOL001に記述されていること

SOL001ではこのTOSCA 形式のディスクリプタでどういうタイプ（リソースタイプやデータタイプ）が定義されているのかと、そのタイプが持つプロパティやケーパビリティ等に関する情報を知ることができます。

- プロパティ等は何を意味し、意図するのか
- プロパティ等は何が必須で何がオプションか
- プロパティ等の型・タイプは何か
- プロパティ等のとるべき値の範囲や制約は何か


## 対応するIFAドキュメント

ETSI NFV-SOL は ETSI NFV-IFA(アイファと発音します) と呼ばれる同じETSI NFV内の別WG(Interfaces and Architecture WG)で策定された標準化文書をさらに参照・準拠しています。IFAにおいてはNFVのファンクションが持つべき機能であったり、ファンクション内で保持すべき情報、あるいはファンクション間でやり取りすべき情報のモデルを定義しており、SOLではそれを実際のデータモデルにマッピングします。

SOLで定義されているデータモデルなどはIFA上にドキュメントに定義があり、それを実体化しています。

ETSI NFV-SOL001に対しては ETSI NFV-IFA011とIFA014が対応しています。

これらの情報は *2. Reference / 2.1 Normative Reference* に記載があります(Normative Referenceは準拠すべき他の標準) 。

![Normative reference](https://storage.googleapis.com/zenn-user-upload/315220271bd7-20211119.png)


## ドキュメントの置き場所

(ETSI内のリンクをたどって一体この場所にどのようにしてたどり着くのかは私は知りませんが、)パブリッシュ済みのドキュメントは以下から参照することができます。

https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/001/


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
4. Overview of TOSCA model
5. General concept of using TOSCA to model NFV descriptors
6. VNFD TOSCA model
7. NSD TOSCA model
8. PNFD TOSCA model
9. Annex A (informative): Examples
10. Annex B (normative): etsi_nfv_sol001_type definitions
11. Annex C (normative): Conformance
12. Annex D (informative): Mapping between properties of TOSCA types and API attributes
13. Annex E (informative): Authors & contributors
14. Annex F (informative): Change History
15. History
:::

文書の構成はバージョンによって多少差があります。

ただ1-3章までの項目は全NFV標準化ドキュメントで変わりません。
1章にはスコープを、2章には参照すべき他の標準化文章等を、3章には用語や略称、記号等を記載します。

SOL001を参照したいというモチベーションがある人はおそらくNSDやVNFDの作成や修正等において何か困ったことに遭遇している人だと思いますので、そのような人にとってはあまりこのあたりの章は重要ではないので、目的が明確な人は読まなくても大丈夫です。

4章以降、Annexより前までが基本的にはメインコンテンツになります。
この4章以降のメインコンテンツの構成は標準化文書により構成は大きく異なります。

SOL001では4章と5章においてTOSCAの説明、6章はVNFD、7章はNSD、8章はPNFDに関するデータモデル、Descriptorの記述内容を定義しています。

Annex以降にはExampleや参考となる情報が記載されています。
`normative`と書いているAnnexは準拠すべき内容ですが、`informative`と書いているAnnexは準拠すべき内容ではなく、参考となる情報、実例などが記載されています。


## ETSI NFV拡張 TOSCA タイプ

VNFDやNSDを記述する上で使用するTOSCA タイプをメインコンテンツでは定義しており、TOSCAのタイプ定義に従い、9つのタイプが定義されています。

|TOSCA Types       |URI表記                  |備考                                 |
|:-----------------|:------------------------|:------------------------------------|
|Data Types        |tosca.datatypes.nfv.*    |                                     |
|Artifact Types    |tosca.artifacts.nfv.*    |                                     |
|Capability Types  |tosca.capabilities.nfv.* |                                     |
|Requirements Types|-                        |2021年現在NFV標準上は使用されていない|
|Relationship Types|tosca.relationships.nfv.*|                                     |
|Interface Types   |tosca.interfaces.nfv.*   |                                     |
|Node Types        |tosca.nodes.nfv.*        |                                     |
|Group Types       |tosca.groups.nfv.*       |                                     |
|Policy Types      |tosca.policies.nfv.*     |                                     |

これらのタイプはTOSCAで元から定義されたタイプを拡張する形で定義されています。

これらのETSI NFVにて拡張されたTOSCAタイプを定義したYAMLファイルは`Annex B`にて言及があり、[ETSIの持つリポジトリ上](https://forge.etsi.org/rep/nfv/SOL001/)で配布されています。
また、標準文書と同じ場所でzip形式でもまとめて提供されています。これらを`import`して実際のVNFDやNSDなどでは使用することとなります。

|リリース|バージョン|URL                                                                                            |
|:------:|:---------|:----------------------------------------------------------------------------------------------|
|2       |2.5.1     |https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/001/02.05.01_60/gs_nfv-sol001v020501p0.zip|
|2       |2.6.1     |https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/001/02.06.01_60/gs_nfv-sol001v020601p0.zip|
|2       |2.7.1     |https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/001/02.07.01_60/gs_nfv-sol001v020701p0.zip|
|2       |2.8.1     |https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/001/02.08.01_60/gs_nfv-sol001v020801p0.zip|
|3       |3.3.1     |https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/001/03.03.01_60/gs_nfv-sol001v030301p0.zip|
|3       |3.5.1     |https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/001/03.05.01_60/gs_nfv-sol001v030501p0.zip|

## IFAの定義とTOSCAタイプの対応

VNFD TOSCA Model、NSD TOSCA Model、PNFD TOSCA Modelの各章のイントロダクションにはそれぞれ対応するIFA011/014で定義された情報モデルと、
それを表現するTOSCAタイプの紐付けを示しています。

例えば VNFDに関するイントロダクションでは以下のような表が示されています。

![](https://storage.googleapis.com/zenn-user-upload/174f2974dd07-20211122.png)

ここで示されている通りIFAで定義されている情報モデルがそのままデータモデルにマッピングされていません。
IFAの定義した情報モデルはTOSCAの仕様や制約、管理方法などのためにデータモデルの実装上では情報が分離されたり異なる表現方法が用いられています。

## TOSCA タイプの読み方

6章以降で記載されている ETSI NFVで定義された TOSCA タイプを記述しています。

各タイプの定義には以下のようなセクションが設けられています。

冒頭記載した通り、SOL001を読もうと志す人の大多数はVNFDやNSDを記述あるいは修正等しようとしている人だと思います。
そのような人にとっては、目的となるタイプについて、ここで紹介する内容を探すことになります。

ここでは`6.2.53 tosca.datatypes.nfv.ChecksumData`を例として引用しながら[^why_choose_this_type]読むべきポイントを紹介します。

[^why_choose_this_type]: 引用したい各図表がちょうどよい長さであったためこのタイプを選びました。

1.  **Description:**
    タイプの概要や、タイプのURI、派生元のTOSCAタイプが記載されています。
    VNFDやNSDを記述する上においては`Type URI`の値が必要になりますが、概要以外の情報で特段この情報を参照することはないかと思います。
    ![](https://storage.googleapis.com/zenn-user-upload/28c3502e0f69-20211123.png)

2.  **タイプが持つキー(`properties`や`requirements`等)毎の設定値:**
    このタイプに設定すべき、あるいは設定可能な値が記述されます。
    本内容がこのタイプに対するメインコンテンツであるといえます。
    VNFDやNSDを記述する、その内容を確認する、デバッグするなど、実装上でSOL001を参照する際に主にこの表とにらめっこすることになります。

    ![](https://storage.googleapis.com/zenn-user-upload/7c3965e7ab99-20211123.png)
    上記の例でからは、`algorithm`も`hash`も`required=true`であり省略不可であること、`algorithm`に許容される値は`sha-224`,`sha-256`,`sha-384`,`sha-512`であり、`md5`といったほかのハッシュアルゴリズムや`sha256`という表記の揺れは標準上認められないこと等がわかります。

3.  **Definition:**
    TOSCAのタイプ定義を表示しています。
    この内容は上述の AnnexBやリポジトリで更改されている定義ファイルと同じ内容です[^etsi_typedef]。
    ![](https://storage.googleapis.com/zenn-user-upload/2527eec97599-20211123.png)

[^etsi_typedef]: 本セクションと同じ内容 https://forge.etsi.org/rep/nfv/SOL001/blob/v2.6.1/etsi_nfv_sol001_vnfd_types.yaml#L759-772

4.  **Additional Requirements:**
    Definitionで表しきれない制約事項や実装上、利用上の制限、ルールを示します。
    特段考慮すべき事項がない場合は`None`が記載されます。

5.  **Example:**
    タイプの使用方法、実際の値の設定値などを例示しています。
    タイプによっては別のタイプ、あるいはAnnex Aに誘導しているものもあります。
    ![](https://storage.googleapis.com/zenn-user-upload/d258b0c04143-20211123.png)

## TOSCA タイプ同士の依存関係

SOL001で定義されているタイプのうち、特にData typesなどは他のタイプから参照されるために定義されています。

たとえば上記で引用した`tosca.datatypes.nfv.ChecksumData`というデータタイプは`tosca.datatypes.nfv.SwImageData`から参照されています。

![](https://storage.googleapis.com/zenn-user-upload/6072ff2481cb-20211123.png)

さらに、この`tosca.datatypes.nfv.SwImageData`は`tosca.nodes.nfv.Vdu.Compute`から参照されています。

![](https://storage.googleapis.com/zenn-user-upload/041b58baa5ce-20211123.png)

SOL001を読み進める上では、VNFD上のタイプからこのように相互依存の関係を解決しながら目的のタイプの設定値などを把握する必要があります。

## NSD/VNFDの構成方法について

各VNFD/NSD/PNFD毎の章の最後には`～ TOSCA service template design`と題された節が用意されています。
ここでは、TOSCAを使用する上でのTop-Levelサービステンプレート[^what_is_toplevel]で記載すべき内容やLower-Levelサービステンプレート[^what_is_lowerlevel]で記載すべき内容、複数のデプロイメントフレーバを用いる場合の構成などについて、TOSCAでの表現方法などを記載しています。

一からVNFDなどを作成する人によっては必読でかつ、完璧に理解すべきではありますが、既存の（ETSIに準拠した）VNFDが手元にある人にとっても本内容はその構成や意味の理解の参考となると思います。

[^what_is_toplevel]: TOSCAの用語でテンプレートの抽象化部分です。
[^what_is_lowerlevel]: TOSCAの用語でテンプレートの具体化部分です。

書いている内容についてはTOSCAの知識が無いとおそらく所見では理解し難い部分がかなりありますので、解説はまた別の機会に譲りたいと思います。

## 付録,Annexの活用

最後にAnnexの内容について簡単に紹介します。

特に *Annex A: Examples* については初学者にとっては非常に参考となる内容となっています。
実際のVNF構成、NSD構成等を図示しながらVNFD、NSDの定義を行っています。
実際[OpenStack Tackerのリポジトリに同梱されているVNFDのサンプル](https://opendev.org/openstack/tacker/src/branch/master/samples/etsi_getting_started/tosca/sample_vnf_package_csar/Definitions/sample_vnfd_top.yaml)などもこちらのサンプルを元にしているように見受けられます。

各タイプの定義を見ていても木を見ているようですが、これらのExamplesのVNFDを見ることで森としてVNFDを見ることができます。

# むすび

簡単ではありますが、ざっとSOL001で記載されていることがどんなことなのか、どんな点を気にしながら読む必要があるのかを記載しました。

文中にも記載しましたが、この内容を理解するにあたってはTOSCAの知識が求められる部分が多々あります。
極力解説を織り込んだつもりですが文書の長さもあるため割愛した部分や別の機会にと逃げた部分も多分にあります。

本内容で少しでもSOL001の理解につなげていただけるようなことがあれば幸いです。

非常にニッチな領域で、私自身もSOL001、VNFDのスペシャリストというわけではありません。
間違いがあればコメント欄にご指摘下さい。

Have a nice NFV life :)
