# 最低限の動画配信を達成するチュートリアル

今回はリージョンを`東京`に設定して行う。  

使うAWSサービスは以下の通りである

- S3 (バケットの作成は東京リージョン)
- MediaConvert : 東京リージョン
- Lambda : 東京リージョン
- CloudFront
- CloudWatch (ログを取得)


## AWS Elemental MediaConvertで使用するIAMロールを作成する  

MediaConvertはS3バケットのファイルを読み書きし、動画を処理するときにCloudWatchにイベントを生成する権限が付与されていなければならない。  

1. [ロールの作成]ボタンをクリック  
2. [MediaConvert]をクリックして次にすすむ  
    次のポリシーがアタッチされているはず  
    - AmazonAPIGatewayInvokeFullAccess  
    - AmazonS3FullAccess  
3. `vod-MediaConvertRole`と名前をつけてロールを作成  

> ちなみに以下の公式ドキュメントを参考にロール名を`MediaConvert_Default_Role`とするとMediaConvertにロールを設定しなかった場合にデフォルトでこれが呼ばれるようになる。    
> [To set up your MediaConvert role in IAM](https://docs.aws.amazon.com/mediaconvert/latest/ug/iam-role.html)    



## S3バケットの作成  

1. [バケットの作成]ボタンをクリック  
2. `vod-XXXX`って感じでバケット名をつける  
3. リージョンに`東京`とかを指定  
4. 左下の[作成]ボタンをクリックしてバケットを作成する  
5. 作成したバケットを選択して[プロパティ]タブを選択  
6. 静的サイトホスティングの項目を選択し、インデックスドキュメントに`index.html`を記入、そのまま[保存]ボタンをクリック  
7. [アクセス権限]タブを選択し[ブロックパブリックアクセス]の編集から`パブリックアクセスをすべてブロックをオフ`にする  
8. そのまま[バケットポリシー]を選択して以下のコードを入力して[保存]  
    これはバケットに対してどこからでも権限なしでアクセスできる設定になる。  
    ```{  
    "Version": "2012-10-17",  
    "Statement": [  
        {  
            "Sid": "AddPerm",  
            "Effect": "Allow",  
            "Principal": "*",  
            "Action": "s3:GetObject",  
            "Resource": "arn:aws:s3:::vod-streaming-backet/*"  
              }  
          ]  
      }  
    ```  
9. [CORSの設定]ボタンを選択して以下を挿入する  
  他のドメインからアクセスする場合のCORS対策  
    ```  
    <?xml version="1.0" encoding="UTF-8"?>  
    <CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">  
    <CORSRule>  
        <AllowedOrigin>*</AllowedOrigin>  
        <AllowedMethod>GET</AllowedMethod>  
        <MaxAgeSeconds>3000</MaxAgeSeconds>  
        <AllowedHeader>*</AllowedHeader>  
    </CORSRule>  
    </CORSConfiguration>  
    ```  


### 必要に応じて管理者権限でないユーザーに対してもポリシーがアタッチできるように、IAMで管理者ポリシーを作成する  

権限が制限されているようなユーザーにはこの権限を作成・付与してあげるといい？  

[aws-media-services-simple-vod-workflow/README-user.md at master · aws-samples/aws-media-services-simple-vod-workflow · GitHub](https://github.com/aws-samples/aws-media-services-simple-vod-workflow/blob/master/1-IAMandS3/README-user.md)  


## AWS Elemental MediaConvertのジョブを作成  

MediaConvertのジョブを動かすには`MediaConvert`がS3から動画を読み込み、複数のフォーマットに変換・出力する必要がある。  
このため、以下のリソースが必要になる。  

- MediaConvertRole : MediaConvertがアカウント内のリソースにアクセするための権限を与えるためのロール  
- MediaBacket : MediaConvertからの出力を保存するためのバケット  

### ジョブの作成  

MediaConvertはS3に保存された動画を読み取り、複数のフォーマットに変化する。  

>    
> [動画変換の一例](<https://github.com/aws-samples/aws-media-services-simple-vod-workflow/blob/master/2-MediaConvertJobs/>)
> ![画像ファイル](https://github.com/aws-samples/aws-media-services-simple-vod-workflow/blob/master/images/mediaconvert-job.png?raw=true)  
>   


[2-MediaConvertJobs](https://github.com/aws-samples/aws-media-services-simple-vod-workflow/blob/master/2-MediaConvertJobs/README.md)では`HLS`, `MP4`, `サムネイル`に分けてフォルダに出力されるような構成で作成している。  

ジョブの作成・ジョブテンプレートの作成などに関しては以下の文献が一番わかりやすいかもしれない。  
まずは、以下をみて試してみるのも手だと思う。  

> [Elemental MediaConvertを利用してmp4をHLS変換 + サムネイル出力 ｜ Developers.IO](https://dev.classmethod.jp/cloud/elemental-mediaconvert-mp4-to-hls/)  
>また公式のクソ長いドキュメントはこちら  
>[MediaConvert ユーザーガイド](https://docs.aws.amazon.com/ja_jp/mediaconvert/latest/ug/mediaconvert-guide.pdf)  
>  


### 変換した動画を確認してみる  

実際にジョブを実行して動画が変換されてもChromeでは`HLS`をサポートしていないので`Video.js`や`hls.js`, Chrome拡張機能などを導入しない限り動画をみることができない。  
Safariで確認してもいいが以下のサイトであればどのデバイスでも安定して動画の再生確認を行えるのでおすすめ。  

> [JW Developer - Stream Tester](https://developer.jwplayer.com/tools/stream-tester/)  


### 入力動画を編集して出力する  

以下のドキュメントでは入力した動画を編集して出力する方法について記述されている。  

動画コンテンツの多くは動画を組み合わせて構成している。  
例えばYoutubeの動画コンテンツは本編の冒頭にチャンネルクリップを再生し、本編の終わりにはチャンネル登録や他のコンテンツを促したりすると思います。これらはバンパー広告などと呼ばれたりもする。  

これらの挿入を自動で行えるのもMediaConvertの強みだ。自身のサービスで動画再生の前に何かテロップなどを流したい場合は効果的に働くと思う。  

> [Modifying AWS Elemental MediaConvert Inputs](https://github.com/aws-samples/aws-media-services-simple-vod-workflow/blob/master/3-Inputs/README.md)  

### 動画に画像やタイムレコードなどを挿入する  

動画に画像・透過画像やテロップなどを挿入したい場合は以下のような手続きで実現できるようだ。  

> [Modifying AWS Elemental MediaConvert Outputs](https://github.com/aws-samples/aws-media-services-simple-vod-workflow/blob/master/4-Outputs/README.md)  


### 動画に字幕を挿入する  

字幕ファイル`.srt`を用いて動画を合成することで、自然な字幕を指定通りに挿入することも可能  


> [Working with Captions](https://github.com/aws-samples/aws-media-services-simple-vod-workflow/blob/master/5-Captions/README.md)  
>   

### 動画にメタデータを挿入する  

前節の動画に字幕を挿入するのも動画に対するメタデータの一種である。  
これに加えて広告マーカーや多言語オーディオトラックなどのメタデータを埋め込むことが可能らしい。  

> [Working with embedded input metadata](https://github.com/aws-samples/aws-media-services-simple-vod-workflow/blob/master/6-EmbeddedMetadata/README.md)  
>   

## MediaConvertジョブをLambdaによって自動化する  

これまではMediaConvertに対して手動でジョブを実行し、動画の変換を行ってきた。  

実際のサービスではユーザーの投稿に合わせて動画を変換する必要があり自動化が必要不可欠に思える。  

このため`Lambda`を用いでS3に動画がアップロードされたらMediaConvertのジョブをキックする処理を実装する。  

### まずは動画をアップロードするための`watchfolder`バケットを作成する  

なんか命名をこんな感じにすることを推奨されている。  

> Keep in mind that your bucket's name must be globally unique across all regions and customers. We recommend using a name like `vod-watchfolder-firstname-lastname`.  

自分は以下のような感じで作成した  

```  
# 入力  
vod-streaming-watchfolder  

# 出力  
vod-streaming-backet  
```  

### Lambda関数用のIAMロールを作成  

自動化を行うために、LambdaというAWSのサービスを使用する。  
全てのLambda関数にはIAMロールが関連付けられており、このロールは関数が利用できる他のAWSサービスを定義しています。  

MediaConvertを自動的に実行させるために、専用の新しいロールを作成します。  

1. IAMコンソールに移動して[ロール]をクリック  
2. [ロールの作成]をクリックし、AWSサービスの`Lambda`を選択して[次のステップ]  
3. アクセス権限の入力欄に`AWSLambdaBasicExecutionRole`を入力しヒットした項目で該当のリージョンのものにチェックをいれる  
4. さらに`AmazonS3FullAccess`も追加する  
5. `VODLambdaRole`といった感じで名前をつけて[作成]  
6. ロール一覧画面で、作成した`VODLambdaRole`を選択し[権限]タブ > [インラインポリシーの追加]をクリック, [JSON]タブを選択  


```json  
{  
    "Version": "2012-10-17",  
    "Statement": [  
        {  
            "Action": [  
                "logs:CreateLogGroup",  
                "logs:CreateLogStream",  
                "logs:PutLogEvents"  
            ],  
            "Resource": "*",  
            "Effect": "Allow",  
            "Sid": "Logging"  
        },  
        {  
            "Action": [  
                "iam:PassRole"  
            ],  
            "Resource": [  
                // 最初の方に作ったRoleのリンクに置き換える  
                //　arn:aws:iam::000000000000:role/ロール名  
                "<ARN for vod-MediaConvertRole>"  
            ],  
            "Effect": "Allow",  
            "Sid": "PassRole"  
        },  
        {  
            "Action": [  
                "mediaconvert:*"  
            ],  
            "Resource": [  
                "*"  
            ],  
            "Effect": "Allow",  
            "Sid": "MediaConvertService"  
        }  
    ]  
}  
```  
7. [ポリシーの確認ボタン]を押して  
8. `VODLambdaPolicy`と名付けて作成  


### 動画を変換するためのLambda関数を作成する  

AWS LambdaはS3へのputObjectやHTTPリクエストなどのイベントに応答してコードを実行する。  
ここではMediaConvert Python SDKを用いて動画を処理する主要な関数を構築する。  

今回の関数はS3に動画を追加するたびにMediaConvertジョブをキックするようにする。  

このセクションではジョブをキックするのみでジョブが終了するまでLambdaは待機しない。  
このため将来的にはCloudWatchイベントを使用して、MediaConvertジョブを自動的に監視し、終了時になんらかのアクションを実行したい。  


1. Lambdaを開いて、[関数の作成]をクリック  
2. [一から作成]ボタンを選択した状態で`関数名`に`VODLambdaConvert`を入力  
3. ランタイムにはPython3.7 or 3.8 を選択する  
4. [実行ロールの選択または作成]をクリックして既存のロールから`VODLambdaRole`(さっき作った)を選択  
5. [関数の作成]をクリックする  
6. 関数の作成に成功したら、関数の設定をする画面が表示される。その中の[関数コード]パネルの中で以下のような操作を行う  
   > A. リージョンをオレゴン州に設定している場合  
   >   - [コード エントリ タイプ]のドロップダウンで[Amazon S3からのファイルアップロード]を選択  
   >   - URLを次の物を入力(リージョンをオレゴン州を選択している場合のみ)  
   >    
   > B. リージョンをオレゴン州以外に設定している場合  
   >   - [MediaConvertJobLambda](https://github.com/aws-samples/aws-media-services-simple-vod-workflow/tree/master/7-MediaConvertJobLambda)から`convert.py`と`job.json`をzip化する  
   >   (最も確実なのはこのリポジトリをcloneして`7-MediaConvertJobLambda`の`ziplambda.sh`を実行することだと思う)  
   >   - [コード エントリ タイプ]のドロップダウンで[.zipファイルをアップロード]を選択し、作成したzipをアップロード  
7. アップロードが完了したら`Handler`に`convert.handler`と入力  
8. それぞれの環境変数設定を行う   
    - DestinationBucket = <最初に作った出力用のバケットの名前>  
    - MediaConvertRole = arn:aws:iam::ACCOUNT NUMBER:role/vod-MediaConvertRole  
    - Application = VOD  
9. 基本設定の[タイムアウト]を3秒から2分に増やす  
10. 一番上に戻って[保存]ボタンをクリック  

### 作成したLambda関数が正しく実行できるか確認するテスト  

1. Lambda関数の編集画面のトップにある[テスト]ボタンをクリック  
2. [新しいテストイベントの作成] => イベント名に`ConvertTest`  
3. 以下のコードを入力する  

```json  
{  
  "Records": [  
    {  
      "eventVersion": "2.0",  
      "eventTime": "2017-08-08T00:19:56.995Z",  
      "requestParameters": {  
        "sourceIPAddress": "54.240.197.233"  
      },  
      "s3": {  
        "configurationId": "90bf2f16-1bdf-4de8-bc24-b4bb5cffd5b2",  
        "object": {  
          "eTag": "2fb17542d1a80a7cf3f7643da90cc6f4-18",  
          "key": "vodconsole/TRAILER.mp4",  
          "sequencer": "005989030743D59111",  
          "size": 143005084  
        },  
        "bucket": {  
          "ownerIdentity": {  
            "principalId": ""  
          },  
          "name": "rodeolabz-us-west-2",  
          "arn": "arn:aws:s3:::rodeolabz-us-west-2"  
        },  
        "s3SchemaVersion": "1.0"  
      },  
      "responseElements": {  
        "x-amz-id-2": "K5eJLBzGn/9NDdPu6u3c9NcwGKNklZyY5ArO9QmGa/t6VH2HfUHHhPuwz2zH1Lz4",  
        "x-amz-request-id": "E68D073BC46031E2"  
      },  
      "awsRegion": "us-west-2",  
      "eventName": "ObjectCreated:CompleteMultipartUpload",  
      "userIdentity": {  
        "principalId": ""  
      },  
      "eventSource": "aws:s3"  
    }  
  ]  
}  
```  
4. 作成ボタンを押したら、前の  
5. [テスト]ボタンの左に`ConvertTest`と表示されているはずなので、再び[テスト]ボタンを押し、テストを実行する  
6. 以下のレスポンスが返ってきたら成功  
```  
{  
  "statusCode": 200,  
  "body": "{}",  
  "headers": {  
    "Content-Type": "application/json",  
    "Access-Control-Allow-Origin": "*"  
  }  
}  
```  

### S3に動画を投稿した時のイベントトリガーを作成する  
   
前節でLambda関数の作成に成功したので、いよいよバケットをLambdaに接続してトリガーを配置します。  

1. Lambda関数編集画面でデザイナーの[トリガーを追加]ボタンをクリック  
2. [トリガーの選択]からドロップダウンで`S3`を選択する  
3. バケットに`入力用に作成したバケット`を選択する  
4. イベントタイプで`PUT`を選択する  
5. 残りの設定はデフォルトのままにして[追加ボタン]をクリック  

