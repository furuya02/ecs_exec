# ECS Exec

## 1 環境

* ECS Exec実行のため、下記の設定が環境していることが必要です

#### Session Manager プラグインのインストール

https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

```
% session-manager-plugin

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.
```

#### Execute Command の有効化

* TrueであればOK
```
% aws ecs describe-tasks --cluster <クラスター名> --tasks <タスクID> --query 'tasks[].enableExecuteCommand' --output text
True
```


* Falseの場合は、下記で有効化する
```
% aws ecs update-service --cluster <クラスター名> --service <サービス名> --enable-execute-command
```

#### ESCタスクロールに権限が必要

```json
{  
 "Version": "2012-10-17",  
 "Statement": [  
     {  
     "Effect": "Allow",  
     "Action": [  
          "ssmmessages:CreateControlChannel",  
          "ssmmessages:CreateDataChannel",  
          "ssmmessages:OpenControlChannel",  
          "ssmmessages:OpenDataChannel"  
     ],  
    "Resource": "*"  
    }  
 ]  
}  
```


#### ログインする側に権限が必要

```json
{  
  "Version": "2012-10-17",  
  "Statement": [  
      {  
          "Effect": "Allow",  
          "Action": [  
              "ecs:ExecuteCommand",  
              "ssm:StartSession",  
              "ecs:DescribeTasks"  
          ],  
          "Resource": "*"  
      }  
  ]  
} 
```

#### SSMへのエンドポイントが必要

```js
myVpc.addInterfaceEndpoint(`${id}-SSMMessagesEndpoinForPrivatet`, {
  service: ec2.InterfaceVpcEndpointAwsService.SSM_MESSAGES,
  subnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
});
myVpc.addInterfaceEndpoint(`${id}-SSMEndpointForPrivatet`, {
  service: ec2.InterfaceVpcEndpointAwsService.SSM,
  subnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
});
```

## 2 処理内容

#### プログラムの処理内容は以下の通り

1. ECSクラスタの一覧取得
2. クラスタの選択
3. サービスの有効化確認
4. タスク名一覧取得
5. タスクIDの選択
6. コンテナ名取得
7. ログイン

```
【実行権限付与し、パスの通った所に配置しておく】
% which ecs_exec
/usr/local/bin/ecs_exec

% ls -la /usr/local/bin/ecs_exec
-rwxr-xr-x@ 1 root  wheel  4195 12 27 03:12 /usr/local/bin/ecs_exec

% ecs_exec 

Please choose an cluster:  【カーソル移動でクラスタを選択する】
* クラスタ名1 running:2
  クラスタ名2 running:0
  クラスタ名3 running:1

Please choose an task:　【カーソル移動でタスクを選択する】
  ip:172.18.15.148      RUNNING start:2024-12-01 01:00:23 id:xxxxx
* ip:172.18.11.29       RUNNING start:2024-12-27 02:40:09 id:xxxxx

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session
Starting session with SessionId: ecs-execute-command-7invh2oeqckfx3ytusxdh88rci
/ # hostname 【ログイン】
ip-172-18-11-29.ap-northeast-1.compute.internal

#　exit　【ログアウト】

%
```

####  CLIで実行するならば以下のようになります

* タスク名の取得

```
%aws ecs list-tasks --cluster クラスター名 --query "taskArns[]" --output text
arn:aws:ecs:ap-northeast-1:123456789012:task/xxxxx-cluster/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

* コンテナ名の取得

```
% aws ecs describe-tasks --cluster クラスター名 --tasks  上記コマンドで出力されたタスク名  --query "tasks[].containers[].name" --output text
```

* Fargateへのグイン

```
% aws ecs execute-command --cluster クラスター名 \
    --task タスク名 \
    --container コンテナ名 \
    --interactive \
    --command "/bin/bash" \
```


