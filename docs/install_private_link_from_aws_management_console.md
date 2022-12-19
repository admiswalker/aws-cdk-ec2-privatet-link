# AWS PrivateLink を AWS Management Console からインストールする手順

このドキュメントでは，PrivateLink を AWS Management Console で構成する手順を説明する

## 初期構成

下記のコマンドを実行すると，表記の環境が deploy される．

```console
cd ./infra
npx npm i
npx cdk deloy
```

- VPC1
  - EC2: web サーバへの接続検証用
- VPC2
  - EC2: 検証用の web サーバ (nginx が可動)

## PrivateLink の構成手順

### VPC2 の EC2 にエンドポイントサービス向けの NLB を設置する

1. NLB のパット転送先となるターゲットグループの作成
   1. 操作画面への遷移
      [AWS Management Console]
      → [EC2] で検索
      → [(サイドバーの) ロードバランシング]
      → [(サイドバーの) ターゲットグループ]
      → [ターゲットグループの作成]
      の順にクリック
   2. 設定
      - グループの詳細の指定: 
        - 基本的な設定: 
          - ターゲットタイプの選択: 
            - インスタンス
          - ターゲットグループ名: example-2022-11
          - プロトコル: HTTP
          - ポート: 80
          - VPC: InfraStack/vpc2
      - ターゲットの登録: 
        - 使用可能なインスタンス: 
          - InfraStack/general_purpose_ec2_on_vpc2: ✅
          - [保留中として以下を含める] をクリック
2. NLB の設置
   1. 操作画面への遷移
      [AWS Management Console]
      → [EC2] で検索
      → [(サイドバーの) ロードバランシング]
      → [(サイドバーの) ロードバランサー]
      → [ロードバランサーの作成]
      の順にクリック
   2. 設定
      - ロードバランサータイプの選択
        - Network Load Balancer
      - Network Load Balancer の作成
        - 基本的な設定: 
          - ロードバランサー名: example-2022-11
          - スキーム: 内部
          - IP アドレスタイプ: IPv4
        - Network mapping
          - VPC: InfraStack/vpc2
          - マッピング:
            - ap-northeast-1a: ✅ (EC2 の可動する AZ のみ指定すればよい．ここでは，ap-northeast-1a )
              - サブネット InfraStack/vpc2/Privatesubnet1 (EC2 の可動する Subnet を指定する)
              - プラオベート IPv4 アドレス: 
                - CIDR 10.1.0.96/27 から割り当て済み (恐らく，NLB のアドレス割り当ての話．なので，転送先のアドレスを指定すると重複エラーとなる)
            - ap-northeast-1c: □
            - ap-northeast-1d: □
        - リスナーとルーティング
          - リスナー: 
            - プロトコル: TCP
            - ポート: 80
            - デフォルトアクション: example-2022-11 (ターゲットグループが表示されない場合は，ネットワークが違うか，ターゲットグループの作成を間違えているのでやり直す)

※ VPC Endpoint 側の AZ ID と接続先の AZ ID が異なる場合は，クロスゾーン負荷分散を有効化する．

> クロスゾーン負荷分散
> 
> デフォルトでは、各ロードバランサーノードは、アベイラビリティーゾーン内の登録済みターゲット間でのみトラフィックを分散します。クロスゾーン負荷分散を有効にすると、各ロードバランサーノードは、有効なすべてのアベイラビリティーゾーンの登録済みターゲットにトラフィックを分散します。詳細については、Elastic Load Balancing ユーザーガイドのクロスゾーン負荷分散を参照してください。
>
> ref: [クロスゾーン負荷分散](https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/network/network-load-balancers.html#cross-zone-load-balancing)

### VPC2 側にエンドポイントサービスを設置する

1. エンドポイントサービスの作成
   1. 操作画面への遷移
      [AWS Management Console]
      → [VPC] で検索
      → [(サイドバーの) 仮想プライベートクラウド]
      → [(サイドバーの) エンドポイントサービス]
      → [エンドポイントサービスを作成]
      の順にクリック
   2. 設定
      - エンドポイントサービスを作成: 
        - エンドポイントサービスの設定: 
          - 名前: example-2022-11
          - ロードバランサーのタイプ: ネットワーク
        - 使用可能なロードバランサー: 
          - example-2022-11: ✅
      - 追加設定: 
        - 承認が必要: ✅

### VPC1 側でエンドポイントサービスに接続するエンドポイントを作成する

1. エンドポイントの作成
   1. 操作画面への遷移
      [AWS Management Console]
      → [VPC] で検索
      → [(サイドバーの) 仮想プライベートクラウド]
      → [(サイドバーの) エンドポイント]
      → [エンドポイントを作成]
      の順にクリック
   2. 設定
      - エンドポイントを作成: 
        - エンドポイントの設定: 
          - 名前: example-2022-11
          - サービスカテゴリ: その他のエンドポイントサービス
        - サービス設定: 
          - サービス名: com.amazonaws.vpce.ap-northeast-1.vpce-svc-xxxxxxxxxxxxxxxxx → [サービスの検証] をクリック
            ※ 別アカウントの VPC エンドポイントを参照する場合は認可が必要: [インターフェイス VPC エンドポイントを作成しているときに、検証済みサービスリストに VPC エンドポイントサービスが表示されないのはなぜですか?](https://aws.amazon.com/jp/premiumsupport/knowledge-center/vpc-troubleshoot-interface-endpoint/)
        - VPC: InfraStack/vpc1
        - サブネット: 
          - アベイラビリティーゾーン: ap-northeast-1a (apne1-az4)
          - サブネット ID: InfraStack/vpc1/PrivateSubnet1
          - IP アドレスタイプ: ◉ IPv4
      - セキュリティグループ:
        - InfraStack-generalpurposeec2onvpc1InstanceSecurityGroupXXXXXXXX-XXXXXXXXXXXX (ひとまず VPC1 の EC2 と同じセキュリティグループとする)
2. エンドポイント側からの接続を，エンドポイントサービス側で許可
   1. 操作画面への遷移
      [AWS Management Console]
      → [VPC] で検索
      → [(サイドバーの) 仮想プライベートクラウド]
      → [(サイドバーの) エンドポイントサービス]
      → ＜example-22022-11＞
      → [エンドポイント接続 (タブ)]
      の順にクリック
   2. 設定
      - エンドポイント接続
        - ＜vpce-xxxxxxxxxxxxxxxxx＞, ＜xxxxxxxxxxxx＞, ＜Pending acceptance＞: ◉ (許可したいリクエストを選択する) → [アクション] → [エンドポイント接続リスエストの承諾]

- 参考: [Access an inspection system using a Gateway Load Balancer endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-load-balancer-endpoints.html)

### VPC2 の EC2 のセキュリティグループの設定

1. VPC2 の EC2 のセキュリティグループの作成
   1. 操作画面への遷移
      [AWS Management Console]
      → [EC2] で検索
      → [インスタンス (実行中)]
      → ＜InfraStack/general_purpose_ec2_on_vpc2＞
      → [セキュリティ (タブ)]
      → [セキュリティグループ]
      → [インバウンドルール (タブ)]
      → [インバウンドルールを作成]
      → [ルールを追加]
      の順にクリック
   2. 設定
      - インバウンドルール: 
        - タイプ: HTTP
        - ソース: 0.0.0.0/0 (NLB からの接続を許可したい．アドレスが分からないのでひとまず全て許可する)

### 動作確認

1. VPC エンドポイントの DNS 名の取得
   1. 操作画面への遷移
      [AWS Management Console]
      → [VPC] で検索
      → [(サイドバーの) 仮想プライベートクラウド]
      → [(サイドバーの) エンドポイント]
      → ＜example-22022-11＞
      の順にクリック
   2. DNS 名の取得
      - DNS 名の項目から，DNS 名をコピーする
2. VPC1 の EC2 から，VPC2 の EC2 に立てた web サーバへのアクセス確認
   1. 操作画面への遷移
      [AWS Management Console]
      → [EC2] で検索
      → [インスタンス (実行中)]
      → ＜InfraStack/general_purpose_ec2_on_vpc1＞
      → [接続]
      → [セッションマネージャー (タブ)]
      → [接続]
      の順にクリック
   2. 動作確認
      ```
      curl vpce-xxxxxxxxxxxxxxxxx-xxxxxxxx.vpce-svc-xxxxxxxxxxxxxxxxx.ap-northeast-1.vpce.amazonaws.com
      ```
      下記のようなレスポンスがあれば成功
      ```html
      <!DOCTYPE html>
      <html>
      <head>
      <title>Welcome to nginx!</title>
      <style>
      html { color-scheme: light dark; }
      body { width: 35em; margin: 0 auto;
      font-family: Tahoma, Verdana, Arial, sans-serif; }
      </style>
      </head>
      <body>
      <h1>Welcome to nginx!</h1>
      <p>If you see this page, the nginx web server is successfully installed and
      working. Further configuration is required.</p>
      
      <p>For online documentation and support please refer to
      <a href="http://nginx.org/">nginx.org</a>.<br/>
      Commercial support is available at
      <a href="http://nginx.com/">nginx.com</a>.</p>
      
      <p><em>Thank you for using nginx.</em></p>
      </body>
      ```

## 参考

- [【初心者】AWS PrivateLink を使ってみる](https://qiita.com/mksamba/items/20903940b8b256ef2487)
- [AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html)
