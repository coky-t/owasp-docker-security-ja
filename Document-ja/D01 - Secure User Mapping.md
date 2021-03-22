# D01 - 安全なユーザーマッピング


## 脅威シナリオ

ここでの脅威はコンテナ内に `root` の下で動作するマイクロサービスが提供されていることです。サービスに脆弱性が含まれている場合、攻撃者はコンテナ内で完全な権限を持ちます。デフォルトの保護 (Linux 機能、AppArmor または SELinux プロファイルのいずれか) はまだ残っていますが、保護の一つの層が削除されます。この余分な層は攻撃対象領域を広げます。また、これは最小権限の原則 [1] にも違反しており、OWASP の観点からは安全ではないデフォルトです。

特権コンテナ (`--privileged`) の場合、マイクロサービスからコンテナ内へのブレークアウトはコンテナなしで実行する場合とほぼ同じです。特権コンテナはホスト全体と他のすべてのコンテナを危険にさらします。


## どうすれば防ぐことができるのか？

可能な限り最小の権限でマイクロサービスを実行することが重要です。

ひとまず `--privileged` フラグを使用してはいけません。コンテナにいわゆるすべての機能 (D04 を参照) を提供し、ディスクを含むホストデバイス (`/dev`) にアクセスでき、`/sys` および `/proc` ファイルシステムにもアクセスできます。また少しの作業でコンテナはホストにカーネルモジュールをロードすることもできます [2] 。良い点はコンテナがデフォルトで特権を持たないということです。特権で実行するにはそれらを明示的に構成する必要があります。

ただしマイクロサービスを異なるユーザーの下で root として実行するには設定が必要です。コンテナのミニディストリビューションにユーザー (と場合によってはグループ) の両方を含めように設定する必要があり、サービスはこのユーザーとグループを利用する必要があります。

基本的には二つの選択肢があります。

シンプルなコンテナのシナリオでは、コンテナを構築する際に適切なパラメータを使用して `RUN useradd <username>` または `RUN adduser <username>` を追加する必要があります。それぞれ同じことがグループ ID にも当てはまります。次に、マイクロサービスを開始する前に、 `USER <username>` [3] でこのユーザーに切り替わります。標準のウェブサーバーは 80 や 443 などのポートを使用したがることに注意してください。ユーザーを設定しても 1024 未満のポートにサーバーをバインドすることはできません。どんなサービスでも低いポートにバインドする必要はまったくありません。より高いポートを設定し、それに応じて expose コマンド [4] でこのポートをマップする必要があります。

二つ目の選択肢は Linux *ユーザー名前空間* を使用することです。名前空間は Linux カーネルリソースの異なる (偽の) ビューをコンテナに提供するための一般的な手段です。ユーザー、ネットワーク、PID、IPC などのさまざまなリソースを利用できます。 `namespaces(7)` を参照してください。 *ユーザー名前空間* の場合、コンテナには標準の root ユーザーの相対的なパースペクティブを提供できますが、ホストカーネルはこれを別のユーザー ID にマップします。詳細は [5] `cgroup_namespaces(7)` および `user_namespaces(7)` を参照してください。

名前空間を使用する上での注意点は一度に一つの名前空間しか実行できないことです。ユーザー名前空間を実行する場合、たとえば同じホストでネットワーク名前空間を使用することはできません [6] 。また、コンテナごとに明示的に異なる設定をしない限り、ホスト上のすべてのコンテナはデフォルトの名前空間になります。

いずれにしてもまだ取得されていないユーザー ID を使用してください。たとえばコンテナでサービスを実行し、コンテナの外部で `systemd` ユーザーにマップする場合、これが必ずしも良いとは限りません。

オーケストレーションツールを使用している場合、有用性は異なるかもしれません。オーケストレートされた環境では適切なポッドセキュリティポリシーがあることを確認してください。

## どうすれば見つけ出せるのか？

#### 設定

コンテナをどのように起動するかによりますが、まず初めにコンテナの設定/ビルドファイルにユーザーが含まれているかどうかを調べます。

#### 実行時

ホストのプロセスリストを調べるか、 `docker top` または `docker inspect` を使用します。

1) `ps auxwf`

2) `docker top <containerID>` または `for d in $(docker ps -q); do docker top $d; done`

3) `docker inspect <containerID>` でキー `Config/User` の値を判断します。実行中のすべてのコンテナに対しては `docker inspect $(docker ps -q) --format='{{.Config.User}}'`

#### ユーザー名前空間

ファイル `/etc/subuid` と `/etc/subgid` はすべてのコンテナの UID マッピングを行います。それらが存在せず `/var/lib/docker/` に `root:root` が所有する他のエントリが含まれていない場合、UID 再マッピングを使用していません。一方、それらのファイルが存在し、そのディレクトリにファイルがある場合でも、docker デーモンが `--userns-remap` で起動されたか、設定ファイル `/etc/docker/daemon.json` が使用されたかを確認する必要があります。



## 参考情報
* [1] [OWASP: Security by Design Principles](https://www.owasp.org/index.php/Security_by_Design_Principles#Principle_of_Least_privilege)
* [3] [Docker Docs: USER command](https://docs.docker.com/engine/reference/builder/#user)
* [4] [Docker Docs: EXPOSE command](https://docs.docker.com/engine/reference/builder/#expose)
* [5] [Docker Docs: Isolate containers with a user namespace](https://docs.docker.com/engine/security/userns-remap/)
* [6] [Docker Docs: User namespace known limitations](https://docs.docker.com/engine/security/userns-remap/#user-namespace-known-restrictions)

### コマーシャル

* [2] [How I Hacked Play-with-Docker and Remotely Ran Code on the Host](https://www.cyberark.com/threat-research-blog/how-i-hacked-play-with-docker-and-remotely-ran-code-on-the-host/)
