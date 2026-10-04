# GitHub ActionsからSSM経由でプライベートEC2上のコマンドを実行する

GitHub ActionsからAWS Systems Manager（SSM）のRun Commandを使って、プライベートサブネット上のAmazon EC2インスタンスでシェルコマンドを実行するサンプルです。EC2にインバウンドルールを設定せず、SSH接続も使用しません。EC2からインターネットへの通信にはNAT Gatewayを利用します。

## リポジトリ構成

```text
.
|-- .github/
|   `-- workflows/
|       `-- deploy.yml
|-- cloudformation/
|   `-- EC2.yaml
`-- README.md
```

## 前提条件

* AWSアカウントと、対象アカウントに接続できるAWS CLI
* `cloudformation/EC2.yaml`に定義したリソースを作成する権限
* GitHub Actionsを有効にしたGitHubリポジトリ
* コマンド例はBash形式です。WindowsではGit BashまたはWSLなどで実行してください。

リージョンは`ap-northeast-1`（東京）を使用します。CloudFormationテンプレートはVPC、パブリック／プライベートサブネット、NAT Gateway、Elastic IP、および`t3.micro`のEC2インスタンスを作成します。スタックが存在する間はNAT Gatewayなどの料金が発生するため、検証後は必ず削除してください。

## AWSリソースの作成

リポジトリのルートディレクトリでCloudFormationスタックを作成します。

```bash
aws cloudformation create-stack \
  --stack-name ec2-private-stack \
  --template-body file://cloudformation/EC2.yaml \
  --capabilities CAPABILITY_IAM \
  --region ap-northeast-1

aws cloudformation wait stack-create-complete \
  --stack-name ec2-private-stack \
  --region ap-northeast-1
```

スタックの作成が完了したら、EC2のインスタンスIDを取得します。

```bash
aws cloudformation describe-stacks \
  --stack-name ec2-private-stack \
  --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" \
  --output text \
  --region ap-northeast-1
```

## GitHub Actionsの設定

GitHubリポジトリの **Settings > Secrets and variables > Actions** で、次のRepository VariablesとRepository Secretsを設定します。

* Repository Variable `EC2_INSTANCE_ID`: CloudFormationから取得したインスタンスID
* Repository Secret `AWS_ACCESS_KEY_ID`: 自動実行専用のIAMユーザーなどに発行したアクセスキーID
* Repository Secret `AWS_SECRET_ACCESS_KEY`: 対応するシークレットアクセスキー

AWS認証情報には、対象インスタンスへの`AWS-RunShellScript`送信と、コマンド実行結果の取得に必要な権限を付与します。可能な範囲で対象リソースを限定してください。実運用では、長期アクセスキーの代わりにGitHub OIDCとIAMロールを利用する方法を推奨します。

このワークフローは`main`ブランチへのpushで実行されます。SSM経由で`curl -sS https://example.com`を実行し、完了を待ってから標準出力をActionsのログに表示します。AWS認証情報などのシークレットをリポジトリへcommitしないでください。

## リソースの削除

検証後はスタックを削除し、NAT Gatewayなどの継続課金を防ぎます。

```bash
aws cloudformation delete-stack \
  --stack-name ec2-private-stack \
  --region ap-northeast-1

aws cloudformation wait stack-delete-complete \
  --stack-name ec2-private-stack \
  --region ap-northeast-1
```