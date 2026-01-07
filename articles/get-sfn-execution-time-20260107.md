---
title: "Step Functionsの「経過時間」をDatadogで監視する(Lambda + EventBridge)"
emoji: "⏱️"
type: "tech"
topics: ["AWS", "StepFunctions", "Lambda", "Datadog"]
published: false
publication_name: "geekplus"
---

## はじめに

Geek+の鈴木です。

Step Functions(以下sfn)を運用していると、Datadogなどで「実行がいまどれくらい長引いているか」をリアルタイムに監視したくなる場面があります。

ただ、sfnが標準で提供する実行時間の情報は、基本的に**実行が完了してから確定する“結果の所要時間”**です。実行中(RUNNING)のあいだに「今この実行が何秒経過しているか」をそのまま追うのは簡単ではありません。

そのため、あるステップで想定外のリトライが続いて実行が長時間化していても、完了するまで気づけず、発見が遅れてユーザー影響が大きくなることがあります。

そこでこの記事では、**実行中のsfnの“実行経過時間”をDatadogへ送る**仕組みを紹介します。

## ターゲット
- Datadogにsfnの「実行経過時間」をメトリクスとして送り、可視化したい方

## Datadogカスタムメトリクス関連資料
カスタムメトリクス概要: 
https://docs.datadoghq.com/ja/metrics/custom_metrics/

ここでは、カスタムメトリクスの送信方法（DogStatsD、HTTP API、カスタムAgentチェックなど）が詳細に解説されています。メトリクス名とタグの組み合わせによる識別方法も記載。
​

メトリクス概要: 
https://docs.datadoghq.com/ja/metrics/overview/

カスタムメトリクスの管理やMetrics without Limits™の活用が紹介されています。
​

メトリクス全般: 
https://docs.datadoghq.com/ja/metrics/

カスタムメトリクスの基礎から高度な使い方までカバーされています。


カスタムメトリクスの料金:
https://qiita.com/ryukez/items/5bdfb654147731268e8e

こちらの記事が参考になります。

## 全体像

構成はシンプルです

![](/images/get-sfn-execution-time-20260107/Architect-get-sfn-execution-time.png)

- 実行時間の計算
  - RUNNING実行の`startDate`から`now - startDate`(秒)を算出
- LambdaからDatadogに転送している内容
  - メトリクス:`custom.sfn.elapsed.seconds`(gauge)
  - 値: RUNNINGが無い場合は`0`
  - タグ:`state_machine:<name>`、`env:<value>`.etc

## 工夫したこと

- **最大経過秒だけ送る**: RUNNING実行が複数ある場合「いちばん長く動いている1件」の経過秒だけを送ります。目的が“滞留の早期検知”なので最大値だけ見れば十分で、全実行分を送る方式に比べて送信量(＝Datadog側のコスト)や429(Too Many Requests)のリスクを増やしにくくなります。

## コードのメインの箇所紹介

Datadog転送で必要な変数

```py
# Datadog転送に必要な値
METRIC_NAME = "custom.sfn.elapsed.seconds"    # Datadog に送るメトリクス名
DD_SITE = "<DD_SITE>"                         # 例: datadoghq.com
DD_API_URL = f"https://api.{DD_SITE}/api/v1/series"
DD_API_KEY = "<DD_API_KEY>"
```

RUNNING実行の実行経過時間を算出する部分(サンプル)

```py
# RUNNING 実行の中で、いちばん経過しているもの(最大値)を集計する
max_elapsed = 0  # RUNNINGが無い場合は 0 のまま
token = None     # ページング用
while True:
    # 対象ステートマシンの RUNNING 実行を取得
    kwargs = {"stateMachineArn": sm_arn, "statusFilter": "RUNNING", "maxResults": MAX_RESULTS}
    if token:
        # 次ページがある場合
        kwargs["nextToken"] = token
    resp = sfn.list_executions(**kwargs)
    for ex in resp.get("executions", []):
        # 実行開始時刻からの経過秒を計算
        elapsed = int((now - ex["startDate"]).total_seconds())
        if elapsed > max_elapsed:
            # 最大値を更新
            max_elapsed = elapsed
    token = resp.get("nextToken")
    if not token:
        # 全ページ読み終わり
        break
```

Datadogへメトリクスを送る部分(サンプル)

```py
# 送信するメトリクス(gauge)を組み立てる
series = [{
    "metric": METRIC_NAME,         # メトリクス名
    "type": "gauge",               # 値を上書きできる時系列
    "points": [[ts, max_elapsed]], # [unix秒, 値]
    "tags": tags,                  # 例: ["state_machine:xxx", "env:prod"]
}]
payload = json.dumps({"series": series}).encode("utf-8")  # Datadog Metrics API の形式
headers = {"Content-Type": "application/json", "DD-API-KEY": DD_API_KEY}  # API Key はヘッダ

req = urllib.request.Request(DD_API_URL, data=payload, headers=headers, method="POST")
with urllib.request.urlopen(req, timeout=8):
    pass
```

## 効果

実行経過時間が一定のしきい値を超えたらアラートを飛ばすことで、ユーザーから問い合わせが来る前に異常に気づけるようになりました。
結果として、影響が広がる前に原因調査・復旧対応に着手できています。
また、Datadogで継続的に可視化・監視することで、「サービスの信頼性」を守ることに繋がっています。

![](/images/get-sfn-execution-time-20260107/sfn-monitoring.png)

## まとめ

sfnは「完了後の所要時間」は追いやすい一方で、「実行中の長時間化」は標準のデータだけだと検知が遅れがちです。
RUNNING実行の最大経過秒を定期送信するだけでも、滞留の早期検知とアラートの起点を作れます。
今後もプロダクトのオブザーバビリティを高め、サービスの信頼性を守っていきたいと思っています。

ここまで記事を読んでいただきありがとうございます。
最後に、少しだけ弊社の紹介をさせてください。
弊社は「サプライチェーンマネジメントの​新しい​あり方をソフトウェアで​思い描く​」ことができる
プロダクトを開発し世の中にリリース中のスタートアップです。
少しでも興味を持っていただけた方は、こちらのEntrance Bookをご覧ください。

https://geekplus.notion.site/Engineer-Entrance-Book-1b4e0d9003de80fbbd7cfb7596295c62