- SREは幅広い活動が必要
  - developing monitoring systems
  - planning capacity
  - responding to incidents
  - ensuring the root causes of outages are addressed
  - and so on.
- このPart3ではSREの日々の活動について実践、理論を話す
  - building and operating large distributed computing systems
http://psychclassics.yorku.ca/Maslow/motivation.htm
- permitting self-actualization and taking active control of the direction of the service rather than reactively fighting fires.

- googleではFigure 3-1のようなService Reliability Hierarchyを使ってる
- Monitoring
  - monitoringなしではサービスが稼働しているのかどうかを判断することはできない
  - しっかり設計しましょう
  - エラーに関してユーザーがそれらを気づく前に問題を認識したい
  - 10章で解説する
- incident response
  - how distributed computing systems actually work (and fail!).
  - we explain how we balance on-call duties with our other responsibilities.
  - 11章で解説する(Being On-Call)
  - Once you’re aware that there is a problem, how do you make it go away?
  - Responding effectively to incidents, however, is something applicable to all teams.
  - 12章で解説する(Effective Troubleshooting.)
  - During an incident, it's often tempting to give in to adrenalin and start responding ad hoc.
  - We advise against this temptation in Chapter 13
  - 13章で解説する(Emergency Response)
  - 14章で解説する
- Postmortem and Root-Cause Analysis(事後検討(反省会)と根本原因の解析)
  - We aim to be alerted on and manually solve only new and exciting problems presented by our service
  - 何度も同じ問題を「修正」するためにひどく退屈
  - 15章(Postmortem Culture: Learning from Failure.)
  - 16章(Tracking Outages,)
  - SRE teams to keep track of recent production incidents, their causes, and actions taken in response to them.
- Testing
  - 17章(Testing for Reliability)
- Capacity Planning
  - 18章(Software Engineering in SRE,)
    - a tool for automating capacity planning.
  - 19章(Load Balancing at the Frontend.)
  - 20章(Load Balancing in the Datacenter)
  - 21章(Handling Overload,)
    - how requests to our services get sent to datacenters
  - 22章(Addressing Cascading Failures)
- Development
  - 23章(Managing Critical State: Distributed Consensus for Reliability)
    - distributed consensus, which (in the guise of Paxos) is at the core of many of Google’s distributed systems, including our globally distributed Cron system.
  - 24章(Distributed Periodic Scheduling with Cron)
    - we outline a system that scales to whole datacenters and beyond, which is no easy task.
  - 25章(Data Processing Pipelines)
    - Different architectures can lead to surprising and counterintuitive challenges.
  - 26章(Data Integrity: What You Read Is What You Wrote)
    - how to keep data safe.
- Product
  - 27章(Reliable Product Launches at Scale,)
    - how Google does reliable product launches at scale to try to give users the best possible experience starting from Day Zero.


### 10章 Practical Alerting from Time-Series Data
- Hierarchy of Production Needsの一番下
- サービスの安定稼働のための基礎部分
- ビジネス目標とサービスの方向性を決定したりするのに必要なこと(第6章)
- Monitoring a very large system is challenging
  - The sheer number of components being analyzed(解析するコンポーネントが大量にある)
  - The need to maintain a reasonably low maintenance burden on the engineers responsible for the system(運用者のメンテナンスの負荷を考慮する必要がある)
- googleのモニタリングシステムはシンプルなメトリックではない
- 一つのサーバの故障でアラートを鳴らさない(being alerted for single-machine failures is unacceptable)
  - それらのデータは対応するにはtoo noisyである
- その代わりに冗長化して堅牢なシステムにする
- たくさんの個々のコンポーネントを管理するよりも、外れ値を取り除き、集約したシグナルになるように大きなシステムを設計すべきだ
  - Rather than requiring management of many individual components, a large system should be designed to aggregate signals and prune outliers.
- 粒度が大切
- googleは10年の歳月をかけてチェックスクリプト形式の古典的な方法から新たなパラダイムへと進化させた

- The Rise of Borgmon
  - borgmonはborgの監視システムとして作られた
  - この章では、内部の監視ツールのアーキテクチャとプログラミングインタフェースについて説明する
  - 近年ではカンブリア紀の大爆発のように監視ツールが登場している
    - Riemann, Heka, Bosun, and Prometheus
	- これらはborgmonとかなり近いtime-series-based alertingツールである
	- 特にPrometheusが近いのでborgmonと比較することができるだろう
- 障害検知のためのスクリプトを実行するのではなく、Borgmon relies on a common data exposition format
  - 低オーバーヘッドでの大量データの収集を可能にし、サブプロセスの実行およびネットワーク接続セットアップのコストを回避することが可能
  - コレをwhite-box monitoringとよんでいる
- データはグラフ生成とアラートに使われる
- 大量のデータをとるにはメトリクスのフォーマットの標準化が必要
  - 例えばHTTP fetch
  ```
% curl http://webserver:80/varz
http_requests 37
errors_total 12
  ```
- Borgmanは、他のBorgmanから収集することができますので、我々は、集約や情報を要約し、各レベルで戦略的にいくつかを破棄、サービスのトポロジに従って階層を構築することができる。
  - a team runs a single Borgmon per cluster, and a pair at the global level.
  - クラスター内にBorgmonが1個、グローバルでペアを組んでいるのが典型パターン(nagios-distributedに近そう)
  - Some very large services shard below the cluster level into many scraper Borgmon, which in turn feed to the cluster-level Borgmon.

- Instrumentation of Applications
  - varzはspaceで区切られたplain textでこんな感じに表示される
  - HTTPのレスポンスコード
  ```
http_responses map:code 200:25 404:0 500:12
  ```
- In hindsight, it’s apparent that this schemaless textual interface makes the barrier to adding new instrumentation very low, which is a positive for both the software engineering and SRE teams. However, this has a trade-off against ongoing maintenance.

- Exporting Variables
  - OpenLDAP exports it through the cn=Monitor subtree; MySQL can report state with a SHOW VARIABLES query; Apache has its mod_status handler.
  - golang expvar expvarmon
    - https://golang.org/pkg/expvar/
    - http://golang.jp/pkg/expvar
    - http://klabgames.tech.blog.jp.klab.com/archives/1047514565.html
- Collection of Exported Data
  - ターゲットを探すには適切な名前解決手段が必要
  - ターゲットリストは、多くの場合、動的であるため、サービス・ディスカバリを使用すると、それを維持するためのコストを削減し、監視を拡張できる.
  - 事前に定義された間隔で、Borgmonは、各ターゲットに/ varz URIをフェッチし、結果をデコードし、メモリ内の値を保存
  - Borgmon also spreads the collection from each instance in the target list over the whole interval, so that collection from each target is not in lockstep with its peers.

- Storage in the Time-Series Arena
  - figure10-1のように管理
  - in-memory databaseを使用し、一定期間が経つとdiskに書き込む
  - In practice, the structure is a fixed-sized block of memory, known as the time-series arena, with a garbage collector that expires the oldest entries once the arena is full.
  - horizon
    - RAM内の一番古いエントリと一番新しいエントリの差
    - datacenter Borgmonとglobal Borgmonは12時間
    - 一つのデータに24byte必要だと12時間、1分単位では17GBのRAMを使う
  - RAMではなくディスクにするものはTSDBと呼ばれる
  - Borgmonでは古いデータをTSDBに投げる
    - TSDBは遅いが、安く巨大なデータを扱える

- Labels and Vectors
  - figure10-2のように一次元にvalueを突っ込んでいく
  - timestampは不要
    - because the values are inserted in the vector at regular intervals in time
    - インターバルがわかれば計算できる
  - 時系列データの名前はlabelsetである
  - labelsetはkey=valueの1以上のペア(labes)からなる(varz pageにkeyはある)
  - TSDBで一意にするために最低でも以下のlabelは付いている
    - var  the name of the variable
    - job  the name given to the type of server being monitored
    - service  A loosely defined collection of jobs that provide a service to users, either internal or external
    - zone  location (typically the datacenter)
  - labelsetの例
    - {var=http_requests,job=webserver,instance=host0:80,service=web,zone=us-west}
  - 問い合わせクエリはすべてのlabelをセットする必要はない
    - その場合はmatchしたものすべてが変える
    - 問い合わせ {var=http_requests,job=webserver,service=web,zone=us-west}
      - {var=http_requests,job=webserver,instance=host0:80,service=web,zone=us-west} 10
      - {var=http_requests,job=webserver,instance=host1:80,service=web,zone=us-west} 9
      - {var=http_requests,job=webserver,instance=host2:80,service=web,zone=us-west} 11
      - {var=http_requests,job=webserver,instance=host3:80,service=web,zone=us-west} 0
      - {var=http_requests,job=webserver,instance=host4:80,service=web,zone=us-west} 10
    - 時間指定もできる  {var=http_requests,job=webserver,service=web,zone=us-west}[10m]
      -  {var=http_requests,job=webserver,instance=host0:80, ...} 0 1 2 3 4 5 6 7 8 9 10
      -  {var=http_requests,job=webserver,instance=host1:80, ...} 0 1 2 3 4 4 5 6 7 8 9
      -  {var=http_requests,job=webserver,instance=host2:80, ...} 0 0 0 0 0 0 0 0 0 0 0
      -  {var=http_requests,job=webserver,instance=host3:80, ...} 0 1 2 3 4 5 6 7 8 9 10

- Rule Evaluation
  - webserverの例
    - 1クラスタのすべてのホスト(task)のステータスコード200以外のパーセンテージでアラート設定(エラーレート)
    - 達成するには以下が必要
      - すべてのtaskでレスポンスコードのレートの集計して、 時間内のエラーレートのベクターを出力
      - ↑の合計エラーレート(sum of that vector) 時間内のクラスタのエラーレートを出力(single value) 200コードは除いて計算する
      - クラスターをまたがったエラーレートをリクエストする トータルエラーレートを到着したエラーレートで割る クラスター内のエラーレートを出力する
    - 115ページ下の方
      - 116ページの例
      - エラーレートをsumで足しておりtask:http_requestのエラーレートの合計がdc:http_requestとなる
      - 今回の例では10分間のエラーレート
        - “task HTTP requests 10-minute rate” and “datacenter HTTP requests 10-minute rate.”
    - 117ページ
      - 最終的に出てくる {var=dc:http_errors:ratio_rate10m,job=webserver}
        - これでdcでのエラーレートはこのBorgmonに問い合わせるだけで良い

- Alerting
  - さっきの例ではエラーレートが0.15 > 0.01 だが、カウントが1なのでペンディングされる
  - もう一度来たら2 > 1になる、2分間ペンディングになり、その後アラートが発行される
  - 中央集中のAlertmanagerに通知される
    - Alert PPCを受け取るとAlertmanagerは正しいところに通知をする

- Sharding the Monitoring Topology
  - Borgmonは他のBorgmonの時系列データをインポートできる
  - 中央サーバが全部集めようとするとボトルネック、SPOFがある設計になる
  - ストリーミングプロトコルでBorgmonが通信しあうことでCPU、通信量をtext-baseのvarzよりも抑えうことができる
  - DC aggregation layer
    - performs mostly rule evaluation for aggregation
  - global layer
    - split between rule evaluation and dashboarding.
- Black-Box Monitoring
  - いわゆる外形監視
    - white-box monitoring means that you aren’t aware o what the users see.DNSエラーとかが見えないとか
  - Proberでやっている
    - runs a protocol check against a target and reports success or failure.
    - The prober can send alerts directly to Alertmanager, or its own varz can be collected by a Borgmon
    - アラートマネージャに直接送るか自身のvarzでBorgmonに取得させる
  - validate the response payload of the protocol (HTML contents of an HTTP response)
    - response times by operation type and payload size
      - they can slice and dice the user-visible performance.
      - データサイズを取るのはいいのかも知れない
  - can be pointed at either the frontend domain or behind the load balancer.
    - detect localized failures and suppress alerts.
- Maintaining the Configuration
  - Borgmon also supports language templates
  - macro-like system enables engineers to construct libraries of rules that can be reused
  - unit and regression tests
  - Productio Monitoring teamっていうのがいて、CIを管理
    - testが正常に回っているか、設定が変ではないか確認している
  - ライブラリが大きく分けて2種類
  - 共通系ライブラリ
    - HTTP server library
    - memory allocation
    - the storage client library
    - generic RPC services,
  - generic aggregation rules for exported variables that engineers can use to model the topology of their service. 
    - provide a single global API, but be homed in many data‐centers.
