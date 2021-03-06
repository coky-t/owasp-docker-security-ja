Docker Security
===============

これは OWASP Docker Top 10 です。仕掛中のものです。

## 本ドキュメントについて

このドキュメントではセキュアにコンテナ化された環境を構築するための最も重要な 10 のセキュリティの箇条書きについて説明します。
ゼロから始める場合の仕様書としても使えますし、請負業者に渡すこともできます。



また既存のインストレーションを監査または保護するためにも使用できますが、特にここでは非常に早い段階からセキュリティについて検討を始めるべきです。
設計フェーズがベストです。
後からでは、決定を変更することが難しくなったり、お金や時間の面でコストがかかります。


### 名称

このドキュメントの名称は OWASP Top 10 に似ていますが、まったく異なります。
まず、OWASP Top 10 のように収集されたデータに基づくリスクについては書かれていません。
次にここでの 10 の箇条書きは (プロアクティブ) コントロールに似ています。

### これは誰のため？

このガイドは開発者、監査人、アーキテクト、システムおよびネットワークエンジニアを対象としています。
上述のように外部の請負業者に対してこのガイドを使用して、契約にフォーマルな技術要件を追加することもできます。
情報セキュリティ責任者はベースラインのセキュリティ要件以上を満たすためにも、ある程度の関心を持つべきです。



これらの 10 の箇条書きは主に (この段落以下を参照) システムとネットワークのセキュリティ、およびシステムとネットワークアーキテクチャに関するものです。
開発者がこれらの専門家である必要はありません。そのためにこのガイドがあります。
しかし上述のようにこれらの点について早い段階で検討し、対処し始めることがベストです。
くれぐれもいきなり作り始めないでください。


箇条書きの中で誤解してはいけないことがあります。
パッチ管理は技術的なポイントではありません。
それは管理プロセスです。
最後になりましたが、コンテナ化についてあまり気にしていなかった技術者や情報セキュリティ管理者のために、このドキュメントは関連するリスクについての洞察も提供しています。

### 本ドキュメントの構成

Docker 環境のセキュリティはしばしば誤解されているようでした。
脅威がどのようなものであるかについて大いに議論されています。
そのため Docker Top 10 の箇条書きに飛び込む前に、脅威をモデル化する必要があり、本ドキュメントで前もって行われています。
これはあらゆるセキュリティ上の影響を理解するのに役立つだけでなく、タスクに優先順位をつけることができます。


### 寄稿

CONTRIBUTING.md をご覧ください。オープンポイントへの寄稿を容易にするために、
対応する開発ブランチ (D06_dev, D07_dev, ...) に対して PR を提出してください。


### PDF 版のビルド方法

Docker と docker-compose がインストールされていれば PDF 版を自分でビルドできます。


```
docker-compose run --rm build
```

リポジトリを汚さないように、このリポジトリでは頻繁には更新されません。

## FAQ

### なぜ "コンテナセキュリティ" ではないのか

このプロジェクトの名前には "Docker" という単語がついていますが、他のコンテナソリューションにもほとんど抽象化せずに使用できます。 
Docker は最も一般的なものであるため、詳細については今のところ Docker に焦点を当てています。
これは後に変更される可能性があります。


### 単一のコンテナ？

一つのサーバーで 3 つ以上のコンテナを実行している場合、おそらくそれらを管理するためにオーケストレーションソリューションを使用しているでしょう。
このようなツールの _具体的な_ セキュリティの落とし穴は現在のところ本ドキュメントの範囲外です。
だからといって、このガイドが手動で管理されている一つまたはいくつかのコンテナだけを対象としているわけではありません。
これは、このようなオーケストレーション環境でのネットワークやホストシステムを含むコンテナに着目していることを意味しており、 _Kubernetes_, _Swarm_, _Rancher_, _OKD/OpenShift_ などの特定の落とし穴には着目していません。




### なぜ十？

正直なところ、私たち人間にとって 10 という数字はキャッチーに聞こえますし、まとめてみるとこの 10 が最も重要なものだと考えられました。
