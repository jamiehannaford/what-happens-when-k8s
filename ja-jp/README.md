# 〜のとき何が起きるのか ... Kubernetes版！

nginx を Kubernetes クラスターにデプロイしたいとしましょう。私はおそらくターミナルでつぎのようなコマンドをタイプするでしょう。

```bash
kubectl run --image=nginx --replicas=3
```

そしてエンターキーを押します。数秒後、3つの nginx ポッドがすべてのワーカーノードに展開されているのがわかるでしょう。魔法のように動作し、素晴らしいです！しかし、実際のところ何が起こっているのでしょうか。

Kubernetes の素晴らしいところの1つは、ユーザーフレンドリーな API を介してインフラストラクチャ全体でワークロードの展開を処理することです。複雑さはシンプルな抽象化によって隠されています。しかし、それが提供する価値を十分に理解するためには、その内部を理解することも有用です。このガイドではクライアントから kubelet へのリクエストのライフサイクル全体を通してあなたを理解へと導きます。そして何が起こっているのかを説明するのに必要なところでソースコードを参照します。

これ文書は鋭意作成中です。改善や書き換えが可能な部分を見つけたら、ぜひコントリビューションしてください！

## contents

1. [kubectl](#kubectl)
   - [バリデーションとジェネレーター](#バリデーションとジェネレーター)
   - [APIグループとバージョンネゴシエーション](#APIグループとバージョンネゴシエーション)
   - [クライアント認証](#クライアント認証)
1. [kube-apiserver](#kube-apiserver)
   - [認証](#認証)
   - [認可](#認可)
   - [アドミッションコントロール](#アドミッションコントロール)
1. [etcd](#etcd)
1. [イニシャライザー](#イニシャライザー)
1. [コントロールループ](#コントロールループ)
   - [Deploymentコントローラー](#Deploymentコントローラー)
   - [Replicasets controller](#Replicasetコントローラー)
   - [インフォーマー](#インフォーマー)
   - [スケジューラー](#スケジューラー)
1. [kubelet](#kubelet)
   - [Pod同期](#Pod同期)
   - [CRIと一時停止コンテナ](#CRIと一時停止コンテナ)
   - [CNIとpodネットワーキング](#CNIとpodネットワーキング)
   - [ホスト間ネットワーキング](#ホスト間ネットワーキング)
   - [コンテナの起動](#コンテナの起動)
1. [まとめ](#まとめ)

## kubectl

### バリデーションとジェネレーター

さあ始めましょう。ターミナルでエンターキーを押しました。何が起こりますか？

kubectl が最初に行うのはクライアントサイドのバリデーションです。これにより、**必ず**失敗するリクエスト（サポートされていないリソースの作成や[不正な形式のイメージ名](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L264)を使用することなど）は早く失敗し、kube-apiserver に送信されません。これにより、不要な負荷を減少させ、システムパフォーマンスが向上します。

バリデーション後、kubectl は kube-apiserver に送信する HTTP リクエストの組み立てを開始します。Kubernetes システム内の状態にアクセスしたり状態を変更しようとする試みはすべて API サーバーを介して行われ、API サーバーは etcd と通信します。kubectl クライアントも同じです。HTTP リクエストを構築するために、kubectl は[ジェネレーター](https://kubernetes.io/docs/user-guide/kubectl-conventions/#generators)と呼ばれるものを使用します。これはシリアル化を処理する抽象です。

`kubectl run` の対象には Deployment リソースだけでなく複数のリソースタイプを指定できるのはよくわからないかもしれません。これを機能させるために、ジェネレーター名が `--generator` フラグを使って明示的に指定されていなければ、kubectl はリソースタイプを[推測](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L319-L339)します。

たとえば、 `--restart-policy = Always` をリソースは Deployment リソースとみなされ、 `--restart-policy = Never` を持つリソースは Pod リソースとみなされます。kubectl はコマンドを記録する（ロールアウトや監査用）など他のアクションを起動する必要があるかどうか、このコマンドが単なるドライランであるかどうか（ `--dry-run` フラグが指定される）も判断します。

Deployment リソースを作成したいことが認識された後、提供されたパラメータから[ランタイムオブジェクトを生成](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/generate/versioned/run.go#L237)するために `DeploymentAppsV1` ジェネレーターを使います。「ランタイムオブジェクト」はリソースの総称です。

### APIグループとバージョンネゴシエーション

先に進む前に指摘する価値があるのは、Kubernetes は「APIグループ」に分類される _versioned_API を使用しているということです。APIグループは、似たリソースを分類して、簡単に推測できるようにすることを目的としています。それはまた、単一のモノリシックAPIに対するより良い代替手段を提供します。Deployment リソースのAPIグループは `apps` という名前で、その最新バージョンは `v1` です。Deployment リソースのマニフェストの上部に `apiVersion: apps/v1` と書く必要があるのはこのためです。

とにかく... kubectl はランタイムオブジェクトを生成した後、[適切なAPIグループとそれに対するバージョンを見つけ始め](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L674-L686)、リソースに対する様々なRESTセマンティクスを知っている[バージョン管理されたクライアント](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L705-L708)を組み立てます。この探索ステージはバージョンネゴシエーションと呼ばれ、すべての利用可能なAPIグループを取得するためにリモートAPI上の `/apis` パスを kubectl がスキャンすることを含みます。kube-apiserver はこのパスでスキーマ文書（ OpenAPI フォーマット）を公開しているので、クライアントがディスカバリーを実行するのは簡単です。

パフォーマンスを向上させるため、kubectl は [OpenAPI スキーマを `〜/.kube/cache/discovery` ディレクトリにもキャッシュします](https://github.com/kubernetes/kubernetes/blob/v1.14.0/staging/src/k8s.io/cli-runtime/pkg/genericclioptions/config_flags.go#L234)。この API のディスカバリーを実際に見たい場合、そのディレクトリを削除し、 `-v` フラグを最大にしてコマンドを実行してみてください。それらの API バージョンを見つけようとしているすべての HTTP リクエストが表示されます。たくさんあります！

最後のステップは、実際に HTTP リクエストを[送信する](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L709)ことです。リクエストが行われ成功レスポンスが返ってきたら、kubectl は[希望された出力フォーマットに基づいて](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L459)成功メッセージを表示します。

### クライアント認証

前のステップで言及しなかったことの1つはクライアント認証です（これは HTTP リクエストが送信される前に処理されます）ので、それを見てみましょう。

リクエストを正常に送信するために、kubectl は認証できる必要があります。ユーザ認証情報はほとんどの場合ディスク上の `kubeconfig`ファイルに保存されていますが、そのファイルは別の場所に保存することもできます。それを見つけるために、kubectl は以下を行います。

- `--kubeconfig` フラグが指定されている場合はそれを使います。
- `$ KUBECONFIG` 環境変数が定義されている場合はそれを使います。
- その他は `~/.kube` のような[予測可能なホームディレクトリ](https://github.com/kubernetes/client-go/blob/master/tools/clientcmd/loader.go#L52)を探し、見つかった最初のファイルを使います。

ファイルを解析した後、使用する現在のコンテキスト、指す現在のクラスタ、現在のユーザーに紐付けられている認証情報を決定します。ユーザーがフラグ固有の値（ `--username` など）を指定した場合、それらが優先され、kubeconfig で指定された値を上書きします。この情報が得られるとkubectl はクライアントの設定を追加し、HTTP リクエストを適切に装飾できるようになります。

-  x509証明書は [`tls.TLSConfig`](https://github.com/kubernetes/client-go/blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89) を使って送信されます。これにはルート CA も含まれます）
- ベアラトークンは「Authorization」HTTP ヘッダーで[送信](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L314)されます
- ユーザー名とパスワードは HTTP ベーシック認証を介して[送信](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223)されます
- OpenID 認証プロセスは事前にユーザーによって手動で処理され、ベアラトークンのように送信されるトークンを生成します

## kube-apiserver

### 認証

リクエストは送信されました、万歳！次は何でしょうか？ここで kube-apiserver が登場します。すでに述べたように、kube-apiserver は、クライアントとシステムコンポーネントがクラスタの状態を永続化して取得するために使用する主要なインタフェースです。その機能を実行するには要求者の本人情報が確認できる必要があります。このプロセスは認証と呼ばれます。

apiserver はどのようにリクエストを認証するのでしょうか？サーバーが最初に起動するとき、ユーザーが提供したすべての [CLI フラグ](https://kubernetes.io/docs/admin/kube-apiserver/)を調べ、適切なオーセンティケーターのリストを組み立てます。例を見てみましょう。 `--client-ca-file` が渡された場合、それはx509オーセンティケーターを追加します。 `--token-auth-file` が渡された場合、トークンオーセンティケーターをリストに追加します。リクエストを受け取るたびに、[成功するまでオーセンティケーターチェーンを通過します](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/union/union.go#L54)。

- [x509ハンドラー](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/x509/x509.go#L60)は、HTTP リクエストが CA ルート証明書によって署名された TLS キーでエンコードされていることを確認します
- [ベアラートークンハンドラー](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/bearertoken/bearertoken.go#L38))は、（HTTP Authorization ヘッダで指定された）トークンが `--token-auth-file` で指定されたディスク上のファイルに存在することを確認します
- [basicauth ハンドラー](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/plugin/pkg/authenticator/request/basicauth/basicauth.go#L37)は、HTTP リクエストの基本認証資格情報が自身のローカル状態と一致することを同様に保証します

すべてのオーセンティケーターが失敗すると、[リクエストは失敗](https://github.com/kubernetes/apiserver/blob/20bfbdf738a0643fe77ffd527b88034dcde1b8e3/pkg/authentication/request/union/union.go#L71)し、集約エラーが返されます。認証が成功すると、 `Authorization` ヘッダがリクエストから削除され、ユーザ情報がそのコンテキストに追加されます。これにより、後続のステップ（認可や認可コントローラーなど）で以前に確立されたユーザーの ID にアクセスできるようになります。

### 認可

さて、リクエストは送信されました。そして kube-apiserver に認証されました。一安心です。しかしまだ終わっていません。私たちは認証されましたが、あるアクションを実行するための権限はあるのでしょうか？結局、アイデンティティと許可は同じものではありません。処理を続けるためには、kube-apiserver に認可される必要があります。

kube-apiserver が認可を処理する方法は認証と非常に似ています。フラグ入力に基づき、すべてのリクエストに対して実行される一連のオーソライザーを集めます。すべてのオーソライザーがリクエストを拒否した場合、リクエストは `Forbidden` レスポンスとなり、それ以上[先に進むことはありません](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authorization.go#L60)。単一のオーソライザーが認可するとリクエストが続行されます。

v1.8に含まれているオーソライザーの例は次のとおりです。

- [webhook](https://github.com/kubernetes/apiserver/blob/d299c880c4e33854f8c45bdd7ab599fb54cbe575/plugin/pkg/authorizer/webhook/webhook.go#L143) はクラスタ外の HTTP（S） サービスとやり取りします
- [ABAC](https://github.com/kubernetes/kubernetes/blob/77b83e446b4e655a71c315ad3f3890dc2a220ccf/pkg/auth/authorizer/abac/abac.go#L223) は静的ファイルで定義されたポリシーを強制します
- [RBAC](https://github.com/kubernetes/kubernetes/blob/8db5ca1fbb280035b126faf0cd7f0420cec5b2b6/plugin/pkg/auth/authorizer/rbac/rbac.go#L43) は管理者によって k8s リソースとして追加された RBAC ロールを強制します
- [NODE](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/plugin/pkg/auth/authorizer/node/node_authorizer.go#L67) ノードクライアント、つまり kubelet は自分自身がホストしているリソースにのみアクセスできるようになります

各 `Authorize` メソッドをチェックしてそれらがどのように機能するのかを確かめてください！

### アドミッションコントロール

さて、それでこの時点で認証され、 kube-apiserver によって認可されました。それでは何が残っているでしょうか？kube-apiserver から見れば、私たちが何者であるかということを信頼し、継続することを許可しますが、Kubernetesとしては、システムの他の部分は何が起こるべきか、そして許されるべきでないかについて強い意見を持ちます。ここで[アドミッションコントローラー](https://kubernetes.io/docs/admin/admission-controllers/#what-are-they)が登場します。

認可はユーザーが許可を得ているかどうかを答えることに焦点を当てていますが、アドミッションコントローラーはリクエストがクラスタのより広い期待とルールに一致することを保証するためにリクエストをインターセプトします。これらは、オブジェクトが etcd に永続化される前の制御の最後の砦です。アクションが予期しない結果や悪影響を与えないように、残りのシステムチェックをカプセル化します。

アドミッションコントローラーの動作方法はオーセンティケーターとオーソライザーの動作方法と似ていますが、1つ違いがあります。オーセンティケーターおよびオーソライザーチェーンとは異なり、単一のアドミッションコントローラーが失敗すると、チェーン全体が壊れ、リクエストは失敗します。

アドミッションコントローラの設計に関して本当にクールなのは、拡張性の促進に焦点を当てていることです。各コントローラーはプラグインとして [`plugin/pkg/admission/directory`](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission) に保存され、小さなインターフェースを満たすように作られています。それぞれが kubernetes の main バイナリにコンパイルされます。

アドミッションコントローラーは通常、リソース管理、セキュリティ、デフォルト設定、参照整合性に分類されます。リソース管理を行うアドミッションコントローラーの例をいくつか示します。

- `InitialResources` は過去の使用状況に基づいてコンテナのリソースにデフォルトのリソース制限を設定します
- `LimitRanger` はコンテナのリクエストと制限のデフォルトを設定するか、特定のリソースに上限を強制します（メモリは2GB以下でデフォルトは512MB）
- `ResourceQuota` は名前空間内のいくつかのオブジェクト（pod、rc、service ロードバランサー）か総消費リソース（cpu、メモリ、ディスク）を計算して拒否します。

## etcd

ここまでで、Kubernetes はリクエストを完全に吟味し、先に進むことを許可しました。次のステップで、kube-apiserver は HTTP リクエストをデシリアライズし、それらからランタイムオブジェクトを構築し（kubectl のジェネレーターの逆プロセスのようなものです）、それらをデータストアに永続化します。少し分解してみましょう。

kube-apiserver はリクエストを受けたとき、どのようにして何をすべきかを知るのでしょうか？リクエストが処理される前にはかなり複雑な一連のステップがあります。バイナリを最初に実行したときから始めましょう。

1. `kube-apiserver` バイナリが実行されると、サーバーチェーンを作成します。これにより　apiserver 集約が可能になります。これは基本的に複数の apiserver をサポートする方法です（これについて心配する必要はありません）。
1. これが起こると、デフォルトの実装として機能する[汎用的な apiserver が作成](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L149)されます。
1. 生成された OpenAPI スキーマが [apiserver の設定](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149)を取り込みます。
1. kube-apiserver は、スキーマで指定されているすべての API グループを反復処理し、それぞれに対して汎用的な抽象ストレージとして機能する[ストレージプロバイダー](https://github.com/kubernetes/kubernetes/blob/c7a1a061c3dc5acabcc0c35b3b96a6935dccf546/pkg/master/master.go#L410)を設定します。これがkube-apiserver がリソースの状態にアクセスしたり変更したりする対象です。
1. すべてのAPIグループに対して各グループバージョンについても繰り返し、HTTP ルートごとに[REST マッピングをインストール](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92)します。これにより kube-apiserver はリクエストをマッピングし、一致するものが見つかったら正しいロジックに委任することができるようになります。
1. 特定のユースケースでは、[POST ハンドラー](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/installer.go#L710)が登録され、それが順番に [create resource ハンドラー](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)に委譲されます。

この時点で、kube-apiserver はどのルートが存在するかを完全に認識しており、リクエストが一致した場合にどのハンドラーとストレージプロバイダーを呼び出すかの内部マッピングを持っています。なんと賢いクッキーなのでしょうか。それでは、HTTP リクエストが流れ込んだとしましょう。

1. ハンドラーチェーンがリクエストを設定パターン（つまり、登録したルート）に一致させることができる場合、そのルートに登録されていた[専用ハンドラにディスパッチ](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/handler.go#L143)されます。それ以外は[パスベースのハンドラー](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L248)にフォールバックします（これは `/apis` を呼び出したときに起こることです）。そのパスにハンドラが登録されていない場合は、[not found ハンドラー](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254)が呼び出され、404が返されます。
1. 幸いなことに、私たちには [`createHandler`](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37) という登録されたルートがあります。それは何をするためのものなのでしょうか？それは最初にHTTPリクエストをデコードし、提供された JSON がバージョン管理された API リソースの期待に沿うことを保証するような基本的なバリデーションを実行するでしょう。
1. 監査と最終アドミッションが[実行され](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104)ます。
1. [ストレージプロバイダに委譲する](https://github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327)ことでリソースが [etcd に保存され](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111)ます。通常、etcd キーは `<名前空間>/<名前>`の形式になりますが、設定可能です。
1. あらゆる作成エラーがキャッチされ、最後に、ストレージプロバイダーはオブジェクトが実際に作成されたことを確認するために `get` 呼び出しを実行します。追加のファイナライズが必要な場合は、post-create ハンドラーとデコレータを呼び出します。
1. HTTP レスポンスが[作成され](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L131-L142)て返送されます。

たくさんのステップがあります！私たちが実際にどれだけの仕事をしているのかを理解しているので、ウサギの穴のパンの耳をたどるのはとても素晴らしいことです。要約すると、Deployment リソースは etcd にあります。しかし、わざと壊すだけでは、まだ見ることはできません…

## イニシャライザー

オブジェクトがデータストアに永続化された後、一連の[イニシャライザー](https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers)が実行されるまでそのオブジェクトは apiserver に完全に可視状態になるわけではなく、スケジュールされることもありません。イニシャライザーは、リソースタイプに関連付けられ、リソースが外部に公開される前にそのリソースに対してロジックを実行するコントローラです。リソースタイプにイニシャライザーが登録されていない場合、この初期化手順はスキップされ、リソースはすぐに可視状態になります。

[多くの素晴らしいブログ投稿](https://ahmet.im/blog/initializers/)で指摘されているように、これを使うと一般的なブートストラップ操作を実行できるので強力な機能です。例えば、

- ポート80が公開された、または特定のアノテーションを備えたプロキシサイドカーコンテナを Pod にインジェクションする
- テスト証明書付きのボリュームを特定のネームスペース内のすべての Pod にインジェクションする
- Secret が20文字未満の場合（パスワードなど）、作成させない

`initializerConfiguration` オブジェクトを使うと、特定のリソースタイプに対してどのイニシャライザーを実行するかを宣言できます。Pod が作成されるたびにカスタムイニシャライザーを実行する必要があるとときは次のように記述します。

```
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```

この設定が作成されると、すべてのPodの `metadata.initializers.pending` フィールドに `custom-pod-initializer` が追加されます。イニシャライザーコントローラーはすでにデプロイされており、新しい Pod を定期的にスキャンします。イニシャライザが Pod の pending フィールドに名前を持つものを検出すると、そのロジックを実行します。処理が完了すると、ペンディングリストから名前が削除されます。名前がリストの先頭にあるイニシャライザーだけがリソースを操作できます。すべてのイニシャライザが終了して `pending` フィールドが空になると、オブジェクトは初期化されたとみなされます。

鋭いあなたは潜在的な問題に気づいたかもしれません。これらのリソースが kube-apiserver によって可視化されていない場合、どのようにしてユーザーランドコントローラーがリソースを処理することができるのでしょうか？この問題を回避するために、kube-apiserver は、初期化されていないものも含め、すべてのオブジェクトを返す `?includeUninitialized` クエリパラメータを公開しています。

## コントロールループ

### Deploymentコントローラー

この段階で、私たちの Deployment レコードは etcd に存在し、初期化ロジックはすべて完了しています。次のステップでは、Kubernetes が依存するリソーストポロジーを設定します。
考えてみると、Deployment は実際には単に ReplicaSet の集まりであり、ReplicaSet は Pod の集まりです。では、Kubernetes は1つの HTTP リクエストからこの階層をどのように作成するのでしょうか？これが、Kubernetes のビルドインコントローラーが引き継ぐところです。

Kubernetes はシステム全体で「コントローラー」を強力に利用しています。コントローラーは、Kubernetes システムの現在の状態を目的の状態に調整するための非同期スクリプトです。
それぞれのコントローラーは小さな責務を持ち、 `kube-controller-manager` コンポーネントによって並行して実行されます。最初に引き継ぐものである Deployment コントローラーを紹介します。

Deployment レコードが etcd に保存され初期化された後、それは kube-apiserver を通して可視化されます。この新しいリソースが利用可能になると、Deployment レコードが変更されたかどうかを監視する役割を持つ Deployment コントローラーによって検出されます。私たちの場合、コントローラーはインフォーマーを介して create イベント用の[特定のコールバックを登録](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L122)します（これが何であるかについての詳細は下記を参照してください）。

このハンドラーは、Deployment が最初に使用可能になったときに実行され、[オブジェクトを内部ワーカーキューに追加する](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L170)ことによって開始されます。オブジェクトの処理に取り掛かるまでに、コントローラは [Deployment を調べ](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L572)てそれに関連付けられた ReplicaSet または Pod レコードがないことを[認識し](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L633)ます。ラベルセレクタを使って kube-apiserver に問い合わせることによって実現されます。興味深いのは、この同期プロセスは状態に依存しないということです。既存のレコードと同じ方法で新しいレコードを調整します。

何も存在しないことを認識した後、状態の解決を開始するために[スケーリングプロセス](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L385)を開始します。ReplicaSet リソースをロールアウト（作成）し、それにラベルセレクタを割り当て、リビジョン番号1を与えることで実現します。ReplicaSet の PodSpec は、Deployment のマニフェストとその他の関連メタデータからコピーされます。場合によっては、Deployment レコードもこの後に更新する必要があります（たとえば、デッドラインが設定されている場合）。

次に[ステータスが更新され](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L70)、Deployment が期待された完了状態に一致するのを待つのと同じ調整ループに戻ります。Deployment コントローラーは ReplicaSet の作成についてのみ関心をもつので、この調整ステージは次のコントローラーである ReplicaSet コントローラによって継続する必要があります。

# ReplicaSetコントローラー

前のステップでは、Deployment コントローラーは、Deployment の最初の ReplicaSet を作成しましたが、まだ Pod はありません。ここで ReplicaSet コントローラーの出番です！仕事は # ReplicaSet とその依存リソース（ Pod ）のライフサイクルを監視することです。他のほとんどのコントローラーと同様に、特定のイベントでハンドラを起動することによって実現されます。

関心のあるイベントは作成です。ReplicaSet が作成されると（ Deployment コントローラーの動作の結果）、RS コントローラーは新しい ReplicaSet の[状態を調べ](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L583)、既存のものと必要なものとの間に差分があることを認識します。その後、ReplicaSet に属する [Pod の数を増やし](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L460)てこの状態を調整しようとします。ReplicaSet のバーストカウント（親の Deployment から継承したもの）が常に一致するように、慎重に作成されます。

Kubernetes はオーナーリファレンス（子リソース内の、親の ID を参照するフィールド）を通じてオブジェクトの階層構造を強制します。これは、コントローラーによって管理されているリソースが削除されると子リソースがガベージコレクションされること（カスケード削除）、を保証するだけでなく、親リソースが子と戦わないための効果的な方法も提供します（2人の潜在的な親が同じ子を共有するのを想像してください！）。

Owner Reference 設計のもう1つの小さな利点はステートフルであることです。コントローラーが再起動されたとしても、リソーストポロジーはコントローラーから独立しているため、ダウンタイムはシステム全体に影響を与えません。分離に集中すると、コントローラー自体の設計にも影響を与えます。コントローラーは、明示的に所有していないリソースを操作するべきではありません。代わりに、コントローラーはその所有権アサーション、非干渉、非共有において選択的であるべきです。

とにかく、オーナーリファレンスに戻りましょう！システムに「孤立した」リソースがある場合があります。つぎのような場合です。

1. 親が削除されたが、子が削除されていないとき
1. ガベージコレクションポリシーがこの削除を禁じているとき

この状況において、コントローラーは新しい親に孤立した子を選ぶよう保証します。複数の親が子を選ぶことを争えますが、成功するのは1人だけです（他の親はバリデーションエラーを受け取ります）。

### インフォーマー

お気づきかもしれませんが、RBAC オーソライザーやDeployment コントローラーはクラスターの状態を関数に退避する必要があります。RBAC オーソライザーの例に戻ると、リクエストがきたときに、オーセンティケーターは後で使用するためにユーザー状態の初期表現を保存します。RBAC オーソライザーは、これを使用して etcd 内のユーザーに関連付けられているすべてのロールとロールバインディングを取得します。コントローラーはどのようにしてそのようなリソースにアクセスして変更するのでしょうか？これは一般的な使用例であり、Kubernetes ではインフォーマーを使って解決されています。

インフォーマーとは、コントローラーがストレージイベントをサブスクライブして、関心のあるリソースを簡単にリストできるようにするパターンです。扱いやすい抽象化を提供することとは別に、それはキャッシングのような多くの仕組みの面倒を見ます（キャッシングは、不要な kube-apiserver 接続を減らし、サーバーとコントローラーの重複するシリアル化コストを減らすので重要です）。この設計を使用することで、周囲に迷惑をかけずに、コントローラーがスレッドセーフな方法でやりとりできるようになります。

インフォーマーがコントローラーに関してどのように機能するかについての詳細は、[このブログ投稿](http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores)をチェックしてください。

# スケジューラー

すべてのコントローラが実行されると、Deployment、ReplicaSet、および3つの Pod が etcd に格納され、kube-apiserver を通じて使用可能になります。しかしながら、私たちの Pod はまだ Node にスケジュールされていないので、 `Pending` 状態のままです。これを解決する最後のコントローラはスケジューラーです。

スケジューラーはコントロールプレーンのスタンドアロンコンポーネントとして実行され、他のコントローラーと同じように動作します。つまりイベントを待機して状態の調整を試みます。この場合、PodSpec に空の `NodeName` フィールドを持つ [Pod をフィルタリングし](https://github.com/kubernetes/kubernetes/blob/master/plugin/pkg/scheduler/factory/factory.go#L190)、その Pod が存在できる適切な Node を見つけようとします。

適切なポッドを見つけるために、特定のスケジューリングアルゴリズムが使用されます。デフォルトのスケジューリングアルゴリズムのしくみは次のとおりです。

1. スケジューラが起動すると、[一連のデフォルト述語が登録され](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go#L65-L81)ます。これらの述語は、評価時に Pod をホストする適性に基づいて [Node をフィルタリングする](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L117)効果的な機能です。たとえば、PodSpec が明示的に CPU または RAM リソースを要求し、Node がキャパシティ不足のためにこれらの要求を満たすことができない場合、Pod の選択は解除されます（リソースキャパシティは現在実行中のコンテナ 「キャパシティ合計」から「リソース要求の合計」を引いたものです）。

1. 適切なノードが選択されると、それらの適合性をランク付けするために、残りの Node に対して一連の[優先順位関数が実行され](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L354-L360)ます。たとえば、ワークロードをシステム全体に分散させるには、他よりもリソース要求が少ないノードを優先します（これは、実行中のワークロードが少ないことを示すためです）。これらの機能を実行すると、各ノードに数値ランクが割り当てられます。そして最高ランクのノードがスケジューリングのために選択されます。

アルゴリズムがノードを見つけると、スケジューラーは Name と UID が Pod と一致し、ObjectReference フィールドが選択された Node の名前を含む [Binding オブジェクトを作成](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/scheduler.go#L336-L342)します。その後、これは POST リクエストを介して [apiserver に送信され](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/factory/factory.go#L1095)ます。

kube-apiserverが この Binding オブジェクトを受け取ると、レジストリはオブジェクトをデシリアライズし、Pod オブジェクトの以下のフィールドを更新します。ObjectReference で [NodeName を設定](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L170)し、[関連するアノテーションを追加](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L174-L176)し、`PodScheduled` ステータス条件を `True` に[設定](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L177-L180)します。

スケジューラーが Podを Node にスケジューリングすると、そのノード上の kubelet が引き継ぎを実行してデプロイメントを開始できます。面白いですね！

**追記: スケジューラーのカスタマイズ:** 面白いのは述語と優先順位関数の両方が拡張可能で、 `--policy-config-file` フラグを使って定義できることです。これはある程度の柔軟性をもたらします。管理者は、スタンドアロン Deployment でカスタムスケジューラー（カスタム処理ロジックを持つコントローラー）を実行することもできます。PodSpec に `schedulerName` が含まれている場合、Kubernetes はその pod のスケジューリングをその名前で登録されているスケジューラーに引き継ぎます。

## kubelet

### Pod同期

さて、メインコントローラのループは終了しました。まとめて見ましょう。HTTP リクエストが認証、認可、アドミッションコントロールの各ステップを通過しました。Deployment、ReplicaSet、3つのPodリソースは etcd に永続化されました。一連のイニシャライザーが実行されましたそして最後に、各 Pod は適切なノードにスケジュールされました。しかしこれまでのところ、私たちが推理してきた状態は純粋に etcd に存在します。次のステップには、ワーカーノード間での状態を分配することが含まれます。これは、Kubernetesの ような分散システムの本質なのです！これは kubelet と呼ばれるコンポーネントを通して行われます。さぁ、始めましょう！

kubelet は、Kubernetes クラスタ内のノード毎に実行されるエージェントで、特に Pod のライフサイクルの管理を担当します。これは、「Pod」（これは実際には単なるKubernetes の概念です）の抽象化とその構成要素であるコンテナとの間のすべての翻訳ロジックを処理することを意味します。また、ボリュームのマウント、コンテナのログ記録、ガベージコレクション、その他多くの重要なことに関連するすべての関連ロジックも処理します。

kubelet について考えるのに便利な方法は、やはりコントローラーのようなものです。20秒ごと（これは設定可能です）に kube-apiserver から Pod をクエリし、 `NodeName` が kubelet が実行されているノードの名前と一致するものをフィルタリングします。リストを持っていると、自身の内部キャッシュと比較することによって新たな追加を検出し、何らかの矛盾が存在すれば状態を同期させ始めます。その同期プロセスがどのようなものかを見てみましょう。

1. pod が作成されている場合（私たちのものです！）、pod のレイテンシーを追跡するために Prometheus で使用されるいくつかの[スタートアップメトリックスを登録](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L1519)します。
1. 次に、Pod の現在の Phase の状態を表す [PodStatus オブジェクトを生成](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1287)します。Pod の Phase は、pod がそのライフサイクルのどこにあるのかの概要です。例としては、`Pending`、`Running`、`Succeeded`、`Failed`、`Unknown` などがあります。この状態を生成するのは非常に複雑なので、正確に何が起こるのかを見てみましょう。
    - 最初に、一連の `PodSyncHandlers` が順番に実行されます。各ハンドラは、Pod がまだノードに存在すべきかどうかを確認します。Pod がもうそこに属していないと判断した場合、Pod のフェーズは `PodFailed` に[変わり](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1293-L1297)、最終的に Node から削除されます。この例としては、 `activeDeadlineSeconds` を超えた後に Pod を削除することがあります（ Jobs 中に使用されます）。
    - 次に、Pod の Phase は init と実際のコンテナのステータスによって決まります。コンテナはまだ起動されていないので、コンテナは[待機中](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1244)として分類されます。待機中のコンテナを持つ Pod には、[`Pending`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1258-L1261) の Phase になります。
    - 最後に、Pod Condition はそのコンテナの状態によって決定されます。コンテナはコンテナランタイムによってまだ作成されていないので、 [`PodReady` 条件をFalse に設定](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/status/generate.go#L70-L81)します。
1. PodStatus が生成された後、Pod のステータスマネージャーに送信されます。これは apiserver を介して etcd レコードを非同期的に更新することを担います。
1. 次に、pod に正しいセキュリティ権限があることを保証するために一連のアドミッションハンドラーが実行されます。これらは [AppArmor プロファイルと `NO_NEW_PRIVS`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L883-L884)を強制することを含みます。この段階で拒否された Pod は無期限に `Pending` の状態のままになります。
1. `cgroups-per-qos`ランタイムフラグが指定されている場合、kubelet はpod 用の cgroup を作成し、リソースパラメータを適用します。これは、pod のサービス品質（ QoS ）処理を向上させるためです。
1. [Pod 用のデータディレクトリが作成され](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L772)ます。これらには pod ディレクトリ（通常は `/var/run/kubelet/pods/<podID>`）、そのボリュームディレクトリ（ `<podDir>/volumes` ）およびそのプラグインディレクトリ（ `<podDir>/plugins` ）が含まれます。
1. ボリュームマネージャは `Spec.Volumes`で定義された関連ボリュームがあればそれを[アタッチして待ち](https://github.com/kubernetes/kubernetes/blob/2723e06a251a4ec3ef241397217e73fa782b0b98/pkg/kubelet/volumemanager/volume_manager.go#L330)ます。マウントされているボリュームの種類によっては、いくつかのポッドではより長い時間待つ必要があります（クラウドや NFS ボリュームなど）。
1. `Spec.ImagePullSecrets` で定義されているすべてのシークレットは、後でコンテナにインジェクションできるように、[apiserver から取得され](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L788)ます。
1. その後、コンテナランタイムはコンテナを実行します（詳細は後述）。

### CRIと一時停止コンテナ

これでほとんどのセットアップが完了し、コンテナを起動する準備が整いました。この起動を行うソフトウェアはコンテナランタイムと呼ばれます（ `docker` や `rkt` がその例です）。

より拡張性を高めるために、v1.5.0以降の kubelet では、具体的なコンテナランタイムとやりとりするために CRI（ Container Runtime Interface ）と呼ばれる概念を使用してきました。一言で言えば、CRI は kubelet と特定のランタイム実装の間の抽象化を提供します。通信は[プロトコルバッファ](https://github.com/google/protobuf)（より速いJSONのようなもの）と[gRPC API](https://grpc.io/)（ Kubernetes オペレーションを実行するのに最適なタイプの API ）を介して行われます。kubelet とランタイムの間で定義済みの契約を使用することによって、コンテナの編成方法に関する実際の実装の詳細はほとんど無関係になるため、これは非常に素晴らしいアイデアです。重要なのは契約だけです。これにより、コア Kubernetes コードを変更する必要がないため、最小限のオーバーヘッドで新しいランタイムを追加できます！

だいぶ横道にそれてしまったのでコンテナのデプロイに戻りましょう…。Pod が最初に起動されると、kubelet は `RunPodSandbox`リモートプロシージャコマンド（ RPC ）を呼び出します。「サンドボックス」とは、CRI用語では一連のコンテナを表し、ご想像の通りKubernetesで言う Pod です。この用語は意図的に曖昧になっているため、実際にコンテナを使用しない他のランタイムの意味を失うことはありません（サンドボックスが VM の場合があるハイパーバイザベースのランタイムを想像してください）。

今回は Docker を使用しています。このランタイムでは、サンドボックスの作成には「一時停止」コンテナの作成が含まれます。一時停止コンテナは、ワークロードコンテナが使用することになる多くの pod レベルのリソースをホストするため、pod 内の他のすべてのコンテナの親のように機能します。これらの「リソース」とは Linux ネームスペース（IPC、network、PID）です。Linux でコンテナがどのように機能するのかに慣れていない場合は、簡単に説明しましょう。Linux カーネルにはネームスペースの概念があり、ホスト OS は専用のリソースセット（ CPU やメモリなど）を切り出し、それを使用している世界で唯一のものであるかのようにプロセスに提供できます。Cgroup は、Linux がリソース割り当てを管理する方法であるため、ここでも重要です（リソース使用量を監視する警官のようなものです）。Docker は、これらのカーネル機能の両方を使用して、リソースが保証され分離が強化されたプロセスをホストします。詳細については、b0rkの素晴らしい投稿「[コンテナとは何か？](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/)」をチェックしてください。

「一時停止」コンテナは、これらの名前空間をすべてホストし、子コンテナがそれらを共有できるようにする方法を提供します。同じネットワーク名前空間の一部であるため、同じ pod 内のコンテナが `localhost` を使用して互いに参照できるのが1つの利点です。一時停止コンテナの2つめの役割は、PID 名前空間がどのように機能するかに関連しています。この種の名前空間では、プロセスが階層ツリーを形成し、一番上の「init」プロセスがデッドプロセスの「刈り取り」を担当します。これがどのように機能するかについての詳細は、[この素晴らしいブログ記事](https://www.ianlewis.org/en/almighty-pause-container)をチェックしてください。

### CNIとpodネットワーキング

私たちの Pod は今や基本的な構造、つまり Pod 間通信を可能にするためにすべての名前空間をホストする一時停止コンテナ持っています。しかし、ネットワーキングはどのように機能し、どのように設定されるのでしょうか？

kubelet が pod 用のネットワークを設定すると、タスクを「CNI」プラグインに委譲します。CNI は Container Network Interface の略で、Container Runtime Interfaceと 同じように動作します。一言で言えば、CNI はさまざまなネットワークプロバイダがさまざまなネットワーク実装をコンテナに使用できるようにするための抽象化です。プラグインが登録され、kubelet は JSON データ（設定ファイルは `/etc/cni/net.d` にあります）を stdin を介して関連する CNI バイナリ（`/opt/cni/bin` にある）にストリーミングすることによってそれらとやりとりします。これは JSON 設定の例です。

```
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```

また、 `CNI_ARGS` 環境変数を通してその名前や名前空間のような pod のための追加のメタデータを指定します。

次に起こることは CNI プラグインに依存していますが、ここでは `bridge` CNI プラグインを見てみましょう。

1. プラグインは最初にルートネットワーク名前空間にローカル Linux ブリッジを設定してそのホスト上のすべてのコンテナにサービスを提供します。
1. 次に、一時停止コンテナのネットワークネームスペースにインターフェイス（ veth ペアの一端）を挿入し、もう一方の端をブリッジに接続します。Vethペア大きなチューブのようなものと考えるのがよいでしょう。一方がコンテナに接続され、もう一方がルートネットワークのネームスペースにあり、パケットがその間を通過できるようにするのです。
1. 次に、一時停止コンテナのインターフェイスに IP を割り当て、ルートを設定します。これにより、Pod に独自の IP アドレスが割り当てられます。IP 割り当ては、JSON 設定に指定されている IPAM プロバイダーに委譲されます。
    - IPAM プラグインは、メインネットワークプラグインと似ています。バイナリを介して呼び出され、標準化されたインターフェースを持ちます。それぞれがコンテナのインターフェースの IP /サブネットをゲートウェイとルートと共に決定し、この情報をメインプラグインに返す必要があります。最も一般的な IPAM プラグインは `host-local`と呼ばれ、事前に定義されたアドレス範囲のセットから IP アドレスを割り当てます。状態をホストファイルシステムのローカルに保存するため、単一ホスト上の IP アドレスの一意性が保証されます。
1. DNS の場合、kubelet は CNI プラグインに内部 DNS サーバーの IP アドレスを指定します。これにより、コンテナの `resolv.conf` ファイルが適切に設定されます。

プロセスが完了すると、プラグインは JSON データを kubelet に返して操作の結果を示します。

### ホスト間ネットワーキング

これまで、コンテナがホストに接続する方法について説明しましたが、ホストはどのように通信するのでしょうか？異なるマシン上の2つのPodが通信したい場合必ず起こります。

これは通常、オーバーレイネットワーキングと呼ばれる概念を使用して実現されます。これは、複数のホスト間で動的にルートを同期させる方法です。人気のオーバーレイネットワークプロバイダの1つが Flannel です。インストール時の中心的な役割は、クラスタ内の複数のノード間にレイヤ3 IPv4ネットワークを提供することです。Flannel はコンテナがどのようにホストにネットワーク接続されるか（これはCNIが覚えている仕事です）を制御するのではなく、トラフィックがホスト「間」でどのように転送されるかを制御します。これを行うには、ホストのサブネットを選択して etcd に登録します。次に、クラスタルートのローカル表現を維持し、発信パケットを UDP データグラムにカプセル化して、正しいホストに到達できるようにします。詳細については、[CoreOS のドキュメント](https://github.com/coreos/flannel)をチェックしてください。

### コンテナの起動

ネットワーキングのすべての面倒なことは完了しました。何が残っているでしょうか？ワークロードコンテナを実際に起動する必要があります。

サンドボックスの初期化が完了してアクティブになると、kubelet はそれに対するコンテナの作成を開始できます。最初に PodSpec で定義されている [init コンテナを起動し](https://github.com/kubernetes/kubernetes/blob/5adfb24f8f25a0d57eb9a7b158db46f9f46f0d80/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L690)、次にメインコンテナ自体を起動します。次のような手順です。

1. [コンテナイメージをプルし](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L90)ます。PodSpec で定義されている Secret はすべてプライベートレジストリに使用されます。
1. CRI を介して[コンテナを作成し](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L115)ます。これは、親の PodSpec から `ContainerConfig` 構造体（コマンド、画像、ラベル、マウント、デバイス、環境変数などが定義されている）を生成し、それをプロトコルバッファ経由で CRI プラグインに送信することによって行われます。Docker の場合は、ペイロードをデシリアライズして、Daemon API に送信するための独自の設定構造体を生成します。その過程で、コンテナにいくつかのメタデータラベル（コンテナタイプ、ログパス、サンドボックス ID など）が適用されます。
1. コンテナを CPU マネージャに登録します。これは1.8の新しいアルファ機能であり、 `UpdateContainerResources` CRI メソッドを使ってローカルノード上の CPU のセットにコンテナを割り当てます。
1. コンテナか[開始され](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L135)ます。
1. 起動後のコンテナライフサイクルフックが登録されている場合は[実行](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L156-L170)されます。フックは `Exec`（コンテナ内の特定のコマンドを実行する）か` HTTP`（コンテナエンドポイントに対してHTTPリクエストを実行する）のどちらかです。PostStartフックの実行に時間がかかりすぎたり、ハングアップしたり、失敗したりした場合、コンテナは決して `running` 状態にはなりません。

### まとめ

完了です。

結局のところ、1つ以上のワーカーノードでは3つのコンテナを実行する必要があります。すべてのネットワーキング、ボリューム、シークレットは、kubelet によって集められ、CRI プラグインを介してコンテナになりました。
