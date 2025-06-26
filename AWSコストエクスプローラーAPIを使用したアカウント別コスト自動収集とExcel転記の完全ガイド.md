<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# AWSコストエクスプローラーAPIを使用したアカウント別コスト自動収集とExcel転記の完全ガイド

AWSコストエクスプローラーAPIを使用してアカウント別のコストを自動で収集し、Excelファイルに転記するシステムの構築は、多くの企業でコスト管理の自動化を実現するために求められている機能です。既存の実装例とともに、具体的な実装方法について詳しく説明します。

## Cost Explorer APIの概要

AWS Cost Explorer APIは、プログラムによってコストと使用状況のデータを取得できるサービスです[^1]。このAPIを使用することで、以下の機能を実現できます：

- 日次・月次のコスト集計データの取得
- アカウント別、サービス別のコスト詳細の取得
- 過去のコストトレンド分析
- 予測データの取得

Cost Explorer APIには料金が発生し、ページ分割されたAPIリクエストごとに0.01 USDが課金されます[^2]。

## 基本的な実装アプローチ

### 1. 必要な権限設定

Cost Explorer APIを使用するためには、適切なIAM権限の設定が必要です[^3][^4]。

**必要なIAMポリシー**：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ce:Get*",
                "ce:Describe*",
                "ce:List*",
                "ce:GetCostAndUsage",
                "organizations:ListAccounts"
            ],
            "Resource": "*"
        }
    ]
}
```

**前提条件**：

- ルートユーザーまたは管理アカウントでCost Explorerを有効化[^3]
- IAMユーザー/ロールによる請求情報へのアクセスを有効化[^4][^5]


### 2. Python実装例（boto3使用）

アカウント別の日別コストを取得する基本的なコードです[^6]：

```python
import datetime
import boto3
import pandas as pd

def get_account_daily_costs():
    today = datetime.date.today()
    start = today.replace(day=1).strftime('%Y-%m-%d')
    end = today.strftime('%Y-%m-%d')
    
    ce = boto3.client('ce')
    
    response = ce.get_cost_and_usage(
        TimePeriod={
            'Start': start,
            'End': end,
        },
        Granularity='DAILY',
        Metrics=['NetUnblendedCost'],
        GroupBy=[
            {
                'Type': 'DIMENSION',
                'Key': 'LINKED_ACCOUNT'
            }
        ]
    )
    
    return response['ResultsByTime']
```


### 3. データ処理とExcel出力

取得したデータをExcel形式で出力するための処理です[^6]：

```python
def process_cost_data_to_excel(cost_data):
    # アカウント情報の取得
    org = boto3.client('organizations')
    accounts = list_accounts()  # アカウント一覧を取得
    account_df = pd.DataFrame(accounts, columns=['Account Id', 'Account Name'])
    
    # コストデータを日付ごとに処理
    merged_cost = pd.DataFrame(index=[], columns=['Account Id'])
    
    for item in cost_data:
        # JSONデータを正規化
        normalized_json = pd.json_normalize(item['Groups'])
        
        # アカウントIDを抽出
        split_keys = pd.DataFrame(
            normalized_json['Keys'].tolist(),
            columns=['Account Id']
        )
        
        # コストデータと結合
        cost = pd.concat([
            split_keys, 
            normalized_json['Metrics.NetUnblendedCost.Amount']
        ], axis=1)
        
        # 列名を日付に変更
        renamed_cost = cost.rename(
            columns={'Metrics.NetUnblendedCost.Amount': item['TimePeriod']['Start']}
        )
        
        # データをマージ
        merged_cost = pd.merge(merged_cost, renamed_cost, on='Account Id', how='outer')
    
    # アカウント名と結合
    daily_cost = pd.merge(account_df, merged_cost, on='Account Id', how='right')
    
    return daily_cost
```


## 既存の実装例

### 1. AWS公式サンプル - Cost Explorer Report Generator

GitHubにAWS公式のCost Explorer Report Generatorが公開されています[^7]。このプロジェクトは以下の機能を提供します：

- PythonとSAM Lambdaを使用したレポート生成
- Excelファイルとグラフの自動作成
- 月次コスト変化の追跡
- Amazon SESを使用したメール送信

**主要な特徴**：

- AWS Lambda上で動作
- スケジュール実行対応
- 複数のタグによるフィルタリング
- グラフ付きExcelレポートの生成


### 2. 日別アカウント別コスト取得の実装例

Qiitaで紹介されている実装例では、具体的なコード例が提供されています[^6]：

```python
import boto3
import pandas
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    today = datetime.date.today()
    start = today.replace(day=1).strftime('%Y-%m-%d')
    end = today.strftime('%Y-%m-%d')
    
    # アカウント一覧取得
    account_list = pandas.DataFrame(
        list_accounts(), 
        columns=['Account Id', 'Account Name']
    )
    
    # コストデータ取得
    daily_cost_list = get_cost_json(start, end)
    
    # データ処理とExcel出力
    merged_cost = process_daily_costs(daily_cost_list)
    daily_cost = pandas.merge(account_list, merged_cost, on='Account Id', how='right')
    
    # CSV/Excel出力
    output_file = '/tmp/output.csv'
    daily_cost.to_csv(output_file, index=False)
    
    # S3にアップロード
    upload_s3(output_file, key, bucket)
```


### 3. openpyxlを使用したExcel操作

PythonでExcelファイルを直接操作する場合、openpyxlライブラリが効果的です[^8][^9]：

```python
import openpyxl
from openpyxl.chart import BarChart, Reference

def create_excel_report(cost_data):
    # 新しいワークブックを作成
    wb = openpyxl.Workbook()
    sheet = wb.active
    sheet.title = "AWS Cost Report"
    
    # ヘッダーを設定
    headers = ["Account ID", "Account Name", "Total Cost (USD)"]
    for col, header in enumerate(headers, 1):
        sheet.cell(row=1, column=col).value = header
    
    # データを入力
    for row, (account_id, account_name, cost) in enumerate(cost_data, 2):
        sheet.cell(row=row, column=1).value = account_id
        sheet.cell(row=row, column=2).value = account_name
        sheet.cell(row=row, column=3).value = float(cost)
    
    # グラフを作成
    chart = BarChart()
    data = Reference(sheet, min_col=3, min_row=1, max_row=len(cost_data)+1)
    chart.add_data(data, titles_from_data=True)
    sheet.add_chart(chart, "E5")
    
    # ファイルを保存
    wb.save("aws_cost_report.xlsx")
```


## 自動化のためのデプロイメント

### 1. AWS Lambdaを使用したスケジュール実行

EventBridgeを使用して定期実行を設定できます[^6]：

```yaml
# SAMテンプレート例
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  CostReportFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: lambda_function.lambda_handler
      Runtime: python3.9
      Events:
        ScheduleEvent:
          Type: Schedule
          Properties:
            Schedule: cron(0 9 * * ? *)  # 毎日9時に実行
```


### 2. 多重アカウント対応

AWS Organizationsを使用している場合のアカウント一覧取得[^10]：

```python
def list_accounts():
    org = boto3.client('organizations')
    accounts = []
    
    try:
        paginator = org.get_paginator('list_accounts')
        for page in paginator.paginate():
            for account in page['Accounts']:
                if account['Status'] == 'ACTIVE':
                    accounts.append([
                        account['Id'],
                        account['Name']
                    ])
    except ClientError as err:
        logger.error(err.response['Error']['Message'])
        raise
    
    return accounts
```


## 注意点と制限事項

### 1. データの確定タイミング

Cost Explorer APIで取得するコストデータには、確定までのタイムラグがあります[^6]。一般的に、当日のコストが確定するまでに1-2日程度かかることがあります。

### 2. ページネーション対応

大量のデータを取得する場合、ページネーション処理が必要です[^11]：

```python
def get_all_cost_data(start_date, end_date):
    ce = boto3.client('ce')
    all_results = []
    next_token = None
    
    while True:
        request_params = {
            'TimePeriod': {'Start': start_date, 'End': end_date},
            'Granularity': 'DAILY',
            'Metrics': ['NetUnblendedCost'],
            'GroupBy': [{'Type': 'DIMENSION', 'Key': 'LINKED_ACCOUNT'}]
        }
        
        if next_token:
            request_params['NextPageToken'] = next_token
        
        response = ce.get_cost_and_usage(**request_params)
        all_results.extend(response['ResultsByTime'])
        
        next_token = response.get('NextPageToken')
        if not next_token:
            break
    
    return all_results
```


### 3. AWS Organizationsでの制限事項

AWS Organizationsに参加する前のコストデータは、参加後にアクセスできなくなります[^12]。このため、組織参加のタイミングには注意が必要です。

## 実装のベストプラクティス

### 1. エラーハンドリング

Cost Explorer APIの呼び出しには適切なエラーハンドリングを実装します[^6]：

```python
from botocore.exceptions import ClientError
import logging

logger = logging.getLogger()

try:
    response = ce.get_cost_and_usage(**params)
except ClientError as err:
    error_code = err.response['Error']['Code']
    if error_code == 'LimitExceededException':
        logger.error("API rate limit exceeded")
        # リトライ処理
    else:
        logger.error(f"API call failed: {err}")
        raise
```


### 2. コスト最適化

APIコストを最小化するため、以下の点に注意します：

- 必要最小限のデータのみ取得
- キャッシュ機能の実装
- バッチ処理での効率的なデータ取得


### 3. セキュリティ考慮事項

- IAM権限は最小権限の原則に従って設定
- AWS Secrets Managerを使用した認証情報の管理
- VPC内でのLambda実行によるネットワークセキュリティの確保

AWSコストエクスプローラーAPIを使用したアカウント別コスト収集とExcel転記の自動化は、適切な設計と実装により、効果的なコスト管理ツールとして機能します。既存の実装例を参考にしながら、組織の要件に合わせてカスタマイズすることで、運用効率の大幅な改善が期待できます。

<div style="text-align: center">⁂</div>

[^1]: https://docs.aws.amazon.com/ja_jp/cost-management/latest/userguide/ce-api.html

[^2]: https://pages.awscloud.com/rs/112-TZM-766/images/AWS-Black-Belt_2024_AWS-CostExplorer_0630_v1.pdf

[^3]: https://zenn.dev/jnxjez/articles/a465a143fa7dcc

[^4]: https://qiita.com/zumax/items/797afc498abeeec2a102

[^5]: https://zenn.dev/mn87/articles/74fd32847ab74d

[^6]: https://qiita.com/hayao_k/items/3034456883c08325e398

[^7]: https://github.com/aws-samples/aws-cost-explorer-report

[^8]: https://cloud5.jp/python_excel/

[^9]: https://saas.n-works.link/programming/python/pythonsopenpyxl

[^10]: https://dev.classmethod.jp/articles/tsnote-how-to-aws-organization-cost-explorer/

[^11]: https://qiita.com/hayao_k/items/f8d77517bdd4bd273148

[^12]: https://qiita.com/takumats/items/4cba40bcb59826b8d04d

[^13]: https://qiita.com/hayao_k/items/c116acf403eb29b53bcd

[^14]: https://docs.aws.amazon.com/cost-management/latest/userguide/ce-api.html

[^15]: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ce.html

[^16]: https://aws.amazon.com/jp/aws-cost-management/aws-cost-explorer/

[^17]: https://blog.future.ad.jp/aws-cost-explorer-のcliをたたいてみた

[^18]: https://www.ecomottblog.com/?p=10180

[^19]: https://pages.awscloud.com/rs/112-TZM-766/images/07-CloudOps-AWS-CostExplorer.pdf

[^20]: https://aws.amazon.com/jp/blogs/news/la-get-cost-estimates-faster-with-aws-pricing-calculator-bulk-import/

[^21]: https://www.ashisuto.co.jp/db_blog/article/aws-price-estimate.html

[^22]: https://thenote.app/note?id=CdXz5zHNQW_YMUcxKwbux

[^23]: https://dev.classmethod.jp/articles/2018-solo-boto3-advent-calendar-day10/

[^24]: https://www.yamamanx.com/aws-cost-explorer-38-history/

[^25]: https://docs.aws.amazon.com/ja_jp/cost-management/latest/userguide/cost-management-guide.pdf

[^26]: https://github.com/aws-samples/cost-explorer-multi-account-forecasting

[^27]: https://qiita.com/jianyi/items/ab855258f16e17c9a317

[^28]: https://qiita.com/akimai/items/d7f4c18e08347e46a18b

[^29]: https://dev.classmethod.jp/articles/cost-reporter/

[^30]: https://qiita.com/kooohei/items/f77ca7f0e15cce421541

[^31]: https://atmarkit.itmedia.co.jp/ait/subtop/features/di/all.html

[^32]: https://trends.codecamp.jp/blogs/media/column256

[^33]: https://tech-frontier.co.jp/media/python-automation-example/

[^34]: https://qiita.com/natsu_san/items/8a6f07a0cf770b923c89

[^35]: https://docs.aws.amazon.com/ja_jp/cost-management/latest/userguide/ce-access.html

[^36]: https://qiita.com/knaito27/items/285469b4a4e972b94ead

[^37]: https://techblog.nhn-techorus.com/archives/29218

[^38]: https://qiita.com/scat117/items/ecb502ce9ccc06a77dbf

[^39]: https://docs.aws.amazon.com/ja_jp/cost-management/latest/userguide/ce-api-best-practices.html

[^40]: https://aws.amazon.com/jp/aws-cost-management/aws-cost-explorer/pricing/

[^41]: https://blog.mmmcorp.co.jp/2018/07/09/scraping_aws_billing/

[^42]: https://aws.amazon.com/jp/textract/

[^43]: https://docs.aws.amazon.com/ja_jp/solutions/latest/cloud-migration-factory-on-aws/cloud-migration-factory-on-aws.pdf

[^44]: https://aws.amazon.com/jp/glue/faqs/

[^45]: https://www.servicenow.com/docs/ja-JP/bundle/yokohama-integrate-applications/page/administer/integrationhub-store-spokes/concept/aws-lambda-spoke.html

[^46]: https://www.cresco.co.jp/blog/entry/445.html

[^47]: https://primenumber.com/blog/aws-cost-explorer_to_redshift_to_looker

[^48]: https://dev.classmethod.jp/articles/openpyxl-large-excel-time-with-lambda/

[^49]: https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/best-practices.html

[^50]: https://zenn.dev/ikihsoy/articles/5cb037ffdaa697

[^51]: https://techblog.ap-com.co.jp/entry/notify-aws-cost-to-line

[^52]: https://docs.aws.amazon.com/ja_jp/prescriptive-guidance/latest/patterns/create-aws-cloudformation-templates-for-aws-dms-tasks-using-microsoft-excel-and-python.html

[^53]: https://www.xlsoft.com/jp/blog/blog/2024/10/09/anaconda-15-post-80162/

[^54]: https://note.com/pttrner_tech/n/n18d0fb472f73

[^55]: https://docs.aws.amazon.com/ja_jp/cost-management/latest/userguide/billing-permissions-ref.html

[^56]: https://www.sunnycloud.jp/column/20230118-01/

[^57]: https://zenn.dev/kr_aws/articles/bc4c5bd47f48ec

