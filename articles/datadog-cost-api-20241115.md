---
title: "Datadog製品使用量APIを使ったデータ取得方法"
emoji: "🐾"
type: "tech"
topics: ["Datadog", "API", "Python"]
published: true
publication_name: "rehabforjapan"
---

## はじめに

皆さんは月のDatadogのコスト予算はどのように作成されていますか？

弊社では今後の予算管理と費用抑制のため、独自の予測計算を行っています。
そのため、月の途中で各製品の日次利用量（料金ではなく）を正確に把握する必要がありました。
しかし、Datadogのコンソール画面ではこれらの情報が確認しづらく確認する製品の数も多いこともあり、コンソール画面を変えながら何十回もクリックしなければいけない状況でした。
そんな中、Datadogの製品使用量APIがあることを知り、それを活用することでより詳細なデータを効率的に取得できるようになりました。現在はそのデータを活用して月の予算を作成しています。
本記事では、そのAPIの取り方をご紹介します。


## ターゲット

・Datadogのコスト管理を実施している方
・Datadogのコスト管理に製品使用量APIを使ってみたいと考えている方


## ゴール

取得したい製品のコストデータが取得できる


## 準備するもの

#### Datadogの「API Keys」 と 「Application Keys」の情報
Datadogのコンソール画面から下記の画面でそれぞれ取得します。
![](/images/datadog-cost-api/datadog-cost-api-manage.png)


#### Pythonの実行環境
Pythonの実行環境を準備します。
弊社の環境では下記のバージョンを利用しています。
- Python ver:3.10.13


## Datadogの製品使用量APIの概要

Datadogの製品使用量APIでは、製品ファミリーと使用量（usage）という概念が導入されています。
製品ファミリーは一つまたは複数の使用量をグループ化したものです。この概念に基づき、APIの構成も「product_family」の下に「usage_type」が配置されています。

下記はサンプルのAPIの返り値です。

```json:返り値sample
{
  "data": [
    {
      "attributes": {
        "product_family": "fargate",
        "measurements": [
          {
            "usage_type": "apm_fargate_count",
            "value": 5
          },
          {
            "usage_type": "tasks_count",
            "value": 10
          },
        ]
      }
    }
  ]
}
```


## サンプルコード

過去10日間のfargate(product_family)のデータを取得するコード(Python)です。
APIのV1は非推奨となっているためV2から取得しています。
注意：[反映されるまでに最大72時間かかる場合があります](https://docs.datadoghq.com/ja/account_management/plan_and_usage/#:~:text=%E6%B3%A8%3A%20%E3%81%93%E3%81%AE%E3%82%BB%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AF%E3%80%81%E6%9B%B4%E6%96%B0%E3%81%BE%E3%81%A7%E3%81%AB%E6%9C%80%E5%A4%A7%2072%20%E6%99%82%E9%96%93%E3%81%8B%E3%81%8B%E3%82%8A%E3%81%BE%E3%81%99%E3%80%82)

```py:sample.py
import requests
from datetime import datetime, timedelta

# APIキーとアプリケーションキーを設定
DATADOG_API_KEY = '{APIキー}'  # Datadogの管理画面から確認
DATADOG_APP_KEY = '{APPキー}'   # Datadogの管理画面から確認

# Datadogの使用量APIエンドポイント
URL = 'https://api.datadoghq.com/api/v2/usage/hourly_usage'

# 認証ヘッダー
headers = {
    'DD-API-KEY': DATADOG_API_KEY,
    'DD-APPLICATION-KEY': DATADOG_APP_KEY,
}

# パラメータの設定（例: 過去15日間のデータを取得）
start_time = (datetime.utcnow() - timedelta(days=10)).strftime('%Y-%m-%dT%H:%M:%SZ')
end_time = datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%SZ')

params = {
    'filter[timestamp][start]': start_time,
    'filter[timestamp][end]': end_time,
    'filter[product_families]': 'fargate',  # ここに任意のproduct_familyを入れる
}

# APIリクエストを実行
response = requests.get(URL, headers=headers, params=params)

# 応答のステータスコードとデータを確認
if response.status_code == 200:
    # 応答からデータを取得
    data = response.json()
    print(data)
else:
    print(f'Error fetching data: {response.status_code}')
```

上記スクリプトの返り値のサンプルです。
```json:返り値(サンプル)
{
  "data": [
    {
      "id": "",
      "type": "usage_timeseries",
      "attributes": {
        "product_family": "fargate",
        "org_name": "",
        "public_id": "",
        "account_name": "",
        "account_public_id": "",
        "region": "us",
        "timestamp": "2024-09-30T13:00:00+00:00",
        "measurements": [
          {
            "usage_type": "apm_fargate_count",
            "value": 2
          },
          {
            "usage_type": "appsec_fargate_avg_task_count",
            "value": 4
          },
          {
            "usage_type": "avg_profiled_fargate_tasks",
            "value": 5
          },
          {
            "usage_type": "tasks_count",
            "value": 15
          }
        ]
      }
    }
  ]
}
```

各product_familyで取得可能な項目は以下の通りです（一部抜粋）
参考
・[使用量のメータリング](https://docs.datadoghq.com/ja/api/latest/usage-metering/)
・[Hourly Usage APIのV1からV2への移行](https://docs.datadoghq.com/ja/account_management/guide/hourly-usage-migration/)
・[Indexed LogsとRUMのHourly UsageおよびSummary Usage APIでの移行](https://docs.datadoghq.com/ja/account_management/guide/relevant-usage-migration/)


#### **infra_hosts** (EC2などのサーバー用)

|`usage_type`|用途|
|---|---|
|`agent_host_count`|Datadogエージェントで監視されているホストの数|
|`apm_azure_app_service_host_count`|APMで監視されているAzure App Serviceのホスト数|
|`apm_host_count`|APMで監視されているホストの数|
|`aws_host_count`|AWSで監視されているホストの数|
|`azure_host_count`|Azureで監視されているホストの数|
|`container_count`|監視されているコンテナの数|
|`gcp_host_count`|Google Cloud Platformで監視されているホストの数|
|`heroku_host_count`|Herokuで監視されているホストの数|
|`host_count`|監視されているホストの総数|
|`infra_azure_app_service`|インフラ監視でのAzure App Serviceの数|
|`opentelemetry_host_count`|OpenTelemetryで監視されているホストの数|

#### **fargate**

|`usage_type`|用途|
|---|---|
|`apm_fargate_count`|APMがtrueになっているFargateタスクの数|
|`appsec_fargate_avg_task_count`|アプリケーションセキュリティモニタリング（AppSec）機能で監視されている Fargate タスクの平均数|
|`avg_profiled_fargate_tasks`|プロファイリングされたFargateタスクの平均数|
|`tasks_count`|Fargateタスクの数|

#### **analyzed_logs**

|`usage_type`|用途|
|---|---|
|`analyzed_logs`|セキュリティ解析のために分析されたログの数|

#### **indexed_logs**

|`usage_type`|用途|
|---|---|
|`logs_indexed_events_3_day_count`|3日間の保持期間でインデックスされたログイベントの数|
|`logs_live_indexed_events_3_day_count`|3日間のライブインデックスされたログイベントの数|
|`logs_rehydrated_indexed_events_3_day_count`|3日間の再ハイドレートされたログイベントの数|
|_他、保持期間別に同様の`usage_type`が存在します_||

#### **indexed_spans**

|`usage_type`|用途|
|---|---|
|`indexed_events_count`|インデックスされたスパンのイベント数|

#### **serverless**

|`usage_type`|用途|
|---|---|
|`func_count`|監視されているサーバーレス関数の数|
|`invocations_sum`|サーバーレス関数の総呼び出し回数|

#### **synthetics_api**

|`usage_type`|用途|
|---|---|
|`check_calls_count`|Synthetics APIテストの実行回数|

#### **ingested_spans**

|`usage_type`|用途|
|---|---|
|`ingested_events_bytes`|インジェストされたスパンデータの総バイト数|

#### **rum** 

|`usage_type`|用途|
|---|---|
|`rum_replay_session_count`|RUMリプレイセッションの数|
|`rum_lite_session_count`|RUM Liteセッションの数|
|`rum_total_session_count`|RUMセッションの総数|
|`rum_mobile_legacy_session_count_android`|モバイル（Android）のレガシーRUMセッションの数|
|`rum_mobile_lite_session_count_ios`|モバイル（iOS）のレガシーRUMセッションの数|


#### **timeseries** (カスタムしたメトリクスのコスト)

| `usage_type`                   | 用途                            |
| ------------------------------ | ----------------------------- |
| `num_custom_timeseries`        | カスタムタイムシリーズの総数                |


## 応用例

弊社では`usage_type`のデータを利用し、当月のコスト予測を独自計算で算出し、月の予算を作成しています。
本ブログでは簡単な紹介のみで詳細は割愛しますが、別の記事で紹介できればと思います。

#### 簡単な紹介
毎月スケジュール起動でLambdaが実行され、取得したデータをRDSに保存します。そのデータをMetabase（BIツール）から参照・コスト予測(計算)し、結果をMetabaseのダッシュボードに表示します。

##### [構成]
![](/images/datadog-cost-api/datadog-cost-api-architecture.png)

##### [コスト算出ダッシュボード]
![](/images/datadog-cost-api/datadog-cost-api-metabase.png)


## まとめ
今回Datadogの製品使用量APIを使ったデータ取得方法の記事を書いたのは、便利なAPIですが活用事例の記事が少なく、少しでも利用を検討している方の助けになればと思ったからです。
弊社でも実際使ってみて、コスト予測の計算をより効率的にすることができました。

ここまで記事を読んでいただきありがとうございます。
最後に、一言だけ弊社の紹介をさせてください。
弊社は「歳を重ねることが 楽しみになる 社会の創造」を目指しているスタートアップです。
それを実現するために様々なサービスをリリース中です。
https://rehabforjapan.com/
