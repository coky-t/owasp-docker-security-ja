# D04 - 安全なデフォルトと堅牢化

*D03 - ネットワークセグメンテーションとファイアウォール* はホスト、コンテナ、オーケストレーションツール上のネットワークベースのサービスに保護レイヤを提供することを目的としていますが、根本的な原因には対処していません。多くの場合、症状を緩和するだけです。まったく必要のないネットワークサービスが開始されているのであれば、そもそも開始すべきではありません。また、開始したサービスが必要なものであれば、適切にロックダウンすべきです。

## 脅威シナリオ

基本的に、サービスが攻撃される可能性のある三つの "ドメイン" があります。

* オーケストレーションツールからのインタフェース。一般的には、ダッシュボード, etcd, API
* ホストからのインタフェース。一般的には、RPC サービス, OpenSSHD, avahi, ネットワークベースの systemd サービス
* コンテナ内のインタフェース。マイクロサービス (spring-boot など) から、またはディストリビューションから。


## どうすれば防ぐことができるのか？

### オーケストレーション / ホスト

オーケストレーションツールでは、どのようなサービスが実行されているか、またそのサービスが適切に保護されているか、あるいはデフォルトの設定が脆弱ではないかを知ることが重要です。

ホストでは、最初のステップは適切なディストリビューションを選択することです。標準の Ubuntu システムは標準の Ubuntu システムです。コンテナのホスティングに特化したディストリビューションがありますので、むしろそちらを検討すべきです。いずれにせよ、最小限のディストリビューションをインストールし、最小限のベアメタルシステムを考えます。また、標準的なディストリビューションを選んだ場合、デスクトップアプリケーション、コンパイラ環境、サーバーアプリケーションなどはここでは出る幕がありません。これらはすべて環境内の重要なシステムに攻撃対象領域を追加してしまいます。

サポートの有効期間はもう一つのトピックです。ホストの OS を選択する際には、EOL の日付を確認します。

一般的には LAN 内の各コンポーネントからどのようなサービスが提供されているかを確認する必要があります。次にそれぞれをどうするか決める必要があります。

* 動作に影響を与えることなく停止/無効化できるか？
* localhost インタフェースのみ、または他のネットワークインタフェースで起動できるか？
* このサービスには認証が設定されているか？
* tcpwrapper (host) またはその他の構成オプションでこのサービスへのアクセスを絞り込むことはできるか？
* 既知の設計上の欠陥はあるか？セキュリティの観点からドキュメントをレビューしたか？

オフ、再構成、または堅牢化できないサービスの場合、ここではネットワークベースの保護 (D03) が少なくとも一つの防御層を提供する必要があります。

また、AppArmor または SELinux ルールが原因でホスト OS に問題が発生した場合、これらの追加の保護をオフにしてはいけません。提供されているツールでシステムログファイルの根本原因を見つけ、それらのルールのみを緩和します。

### コンテナ

コンテナでも、ベストプラクティスは次のとおりです。不要なパッケージをインストールしない [1] 。Alpine Linux はフットプリントが小さく、デフォルトで搭載されているバイナリが少なくなっています。ただし `wget` や `netcat` などの (busybox が提供する) バイナリセットがまだ付属しています。アプリケーションがコンテナ内に侵入した場合、これらのバイナリは攻撃者が "家に電話をかけ" 、ツールを取得するのに役立つ可能性があります。ハードルを高くしたいのであれば、"distroless" [2] イメージを検討すべきです。

検討すべき選択肢がさらにいくつかあります。ホストカーネルに影響を与える可能性があるのはシステムコールの欠陥です。最悪の場合、これによりユーザーとしてコンテナからホスト上の root への特権昇格となる可能性があります。いわゆるケイパビリティはシステムコールのスーパーセットです。

ここではいくつか防御策を紹介します。

* SUID/SGID ビットを無効にする (`--security-opt no-new-privileges`): ユーザーとして実行している場合でも、SUID バイナリが権限を昇格する可能性があります。または、以下を適用する場合には `--cap-drop=setuid --cap-drop=setgid` を使用します。
* 多くのケイパビリティを削除する (`--cap-drop`): Docker ではコンテナのいわゆるケイパビリティを 38 (`/usr/include/linux/capability.h` 参照) から 14 (``man 7 capabilities`` および [3] 参照) に制限しています。 `net_bind_service`, `net_raw` などいくつかの機能を削除してもよいかもしれません ([4] 参照) 。 `pscap` はホスト上でケイパビリティをリストするツールです。 `--cap-add=all` を使用してはいけません。
* もしケイパビリティでは実現できないような細かい制御が必要であれば JSON 形式のプロファイル (`--security-opt seccomp=mysecure.json`) で 300 を超えるシステムコールのそれぞれを seccomp で制御できます ([5] 参照) 。 44 ほどのシステムコールはデフォルトですでに無効になっています。ここでは `unconfined` や `apparmor=unconfined` を使用しないでください。

ベストプラクティスは上記のいずれを選択するか決めることです。ケイパビリティ設定と seccomp プロファイルを混在させないほうがよいでしょう。


## どうすれば見つけ出せるのか？

* オーケストレーションツールには特別な注意が必要です。これまでにも (悪い) 設計により保護されないインタフェースがありました [6], [7]-[9] 。
* 常に同じネットワークからシステムをスキャンして、この LAN で何が公開されているかを確認できます。 D03 ではその方法を説明しています。
* より良い方法はシステムを調べることです。
    * ホスト: 管理者権限でログインし `netstat -tulpn | grep -v ESTABLISHED` または `lsof -i -Pn| grep -v ESTABLISHED` で何が動いているかを確認します。この方法ではコンテナからのネットワークソケットを返しません。
    * コンテナではイメージで `netstat` や `lsof` が提供されていれば、同様にこれらのコマンドを使用できます。
* すべてのサービスはホストベースのファイアウォールにより保護されていることもあります。デフォルトで適用されるルールはホスト OS ごとに異なります。 `iptables -t nat -L -nv` や `iptables -L -nv` からの出力を読み取るだけでは、より大きなコンテナ環境ではすぐに面倒な作業になります。そのため、ここでは LAN もスキャンすることをお勧めします。

## 参考情報

* [1] Docker's [Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
* [2] Google's FLOSS project [distroless](https://github.com/GoogleContainerTools/distroless)
* [3] Docker Documentation: [Runtime privilege and Linux capabilities](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)
* [5] [Docker Documentation, Seccomp security profiles for Docker](https://docs.docker.com/engine/security/seccomp/)
* [6] Weak default of etcd in CoreOS 2.1: [The security footgun in etcd](https://gcollazo.com/the-security-footgun-in-etcd)
* [7] Kubernetes documentation: [Controlling access to the Kubelet](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/#controlling-access-to-the-kubelet): _Kubelets expose HTTPS endpoints which grant powerful control over the node and containers. By default Kubelets allow unauthenticated access to this API. Production clusters should enable Kubelet authentication and authorization._
* [8] Github: ["Exploit"](https://github.com/kayrus/kubelet-exploit) for the API in [7].
* [9] Medium: [Analysis of a Kubernetes hack — Backdooring through kubelet](https://medium.com/handy-tech/analysis-of-a-kubernetes-hack-backdooring-through-kubelet-823be5c3d67c). Incident because of an open API, see [7].

### コマーシャル

* [4] RedHat Blog: Secure your Container: [One weird trick](https://www.redhat.com/en/blog/secure-your-containers-one-weird-trick)



