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

CloudShellを開いた直後はホームディレクトリ（`~`）にいるため、まずリポジトリをcloneして移動します。

```bash
git clone https://github.com/tmatusoka-apc/github-actions-run-tests-on-ec2.git
cd github-actions-run-tests-on-ec2
```

リポジトリのルートディレクトリでCloudFormationスタックを作成します。`--template-body`には、`file://`を付けてローカルファイルを指定してください。`./EC2.yaml`のように`file://`なしで指定すると、ファイルパスではなくテンプレート本文として解釈されます。

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

### IAMユーザーと権限を作成（AWS CLI）

> **注意:** ここでは本プロジェクトで使用している長期アクセスキー方式を再現します。新規構築では長期キーを発行せず、後述のGitHub OIDCによる認証を推奨します。

以下のコマンドは、CloudFormationで作成したEC2に対するSSMコマンド実行用のIAMユーザーとポリシーを作成します。実行者にはIAMユーザー・ポリシーの作成とアタッチ、およびアクセスキー発行の権限が必要です。

まずユーザー名、AWSアカウントID、対象EC2のインスタンスIDを設定し、コンソールログインを持たないIAMユーザーを作成します。

```bash
USER_NAME=github-actions-ssm
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name ec2-private-stack \
  --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" \
  --output text \
  --region ap-northeast-1)

aws iam create-user --user-name "$USER_NAME"
```

続いて、SSMコマンドの送信先を対象EC2と`AWS-RunShellScript`ドキュメントに限定したポリシーファイルを作成します。アカウントIDとインスタンスIDは、先ほど設定したシェル変数から展開されます。

```bash
cat > github-actions-ssm-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": [
        "arn:aws:ssm:ap-northeast-1::document/AWS-RunShellScript",
        "arn:aws:ec2:ap-northeast-1:${AWS_ACCOUNT_ID}:instance/${INSTANCE_ID}"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "ssm:GetCommandInvocation",
      "Resource": "*"
    }
  ]
}
EOF
```

ポリシーをAWSに登録してユーザーへアタッチします。`GetCommandInvocation`はコマンド実行結果の読み取り権限です。

```bash
POLICY_ARN=$(aws iam create-policy \
  --policy-name GitHubActionsSSMPolicy \
  --policy-document file://github-actions-ssm-policy.json \
  --query Policy.Arn \
  --output text)

aws iam attach-user-policy \
  --user-name "$USER_NAME" \
  --policy-arn "$POLICY_ARN"
```

最後にアクセスキーを発行します。結果にはシークレットアクセスキーが含まれるため、信頼できるローカル端末でのみ実行し、出力をチャットや共有ログへ貼り付けないでください。シークレットアクセスキーは発行時に一度しか表示されません。

```bash
aws iam create-access-key \
  --user-name "$USER_NAME" \
  --query 'AccessKey.[AccessKeyId,SecretAccessKey]' \
  --output text
```

### GitHub SecretsとVariablesを設定

GitHubリポジトリの **Settings > Secrets and variables > Actions** で、次のRepository VariablesとRepository Secretsを登録します。

* Repository Variable `EC2_INSTANCE_ID`: CloudFormationから取得したインスタンスID
* Repository Secret `AWS_ACCESS_KEY_ID`: 作成したIAMユーザーのアクセスキーID
* Repository Secret `AWS_SECRET_ACCESS_KEY`: 対応するシークレットアクセスキー

このワークフローは`main`ブランチへのpushで実行されます。SSM経由で`curl -sS https://example.com`を実行し、完了を待ってから標準出力をActionsのログに表示します。AWS認証情報などのシークレットをリポジトリへcommitしないでください。長期アクセスキーを使う場合は、専用IAMユーザーに必要最小限の権限だけを付与し、不要になったキーを無効化・削除してください。新規構築ではGitHub OIDCとIAMロールによる認証を推奨します。

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

専用IAMユーザーも不要になった場合は、GitHubのRepository SecretsとVariablesを削除した後、アクセスキー・ポリシー・ユーザーを削除します。以下は該当ユーザー専用に作成した場合だけ実行してください。

```bash
USER_NAME=github-actions-ssm
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
POLICY_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:policy/GitHubActionsSSMPolicy"
ACCESS_KEY_ID=$(aws iam list-access-keys \
  --user-name "$USER_NAME" \
  --query 'AccessKeyMetadata[0].AccessKeyId' \
  --output text)

aws iam delete-access-key --user-name "$USER_NAME" --access-key-id "$ACCESS_KEY_ID"
aws iam detach-user-policy --user-name "$USER_NAME" --policy-arn "$POLICY_ARN"
aws iam delete-user --user-name "$USER_NAME"
aws iam delete-policy --policy-arn "$POLICY_ARN"
```