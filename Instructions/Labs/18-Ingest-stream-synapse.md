---
lab:
  title: Azure Stream Analytics と Azure Synapse Analytics を使ってリアルタイム データを取り込む
  ilt-use: Lab
---

# Azure Stream Analytics と Azure Synapse Analytics を使ってリアルタイム データを取り込む

データ分析ソリューションには、多くの場合、データ "ストリーム" を取り込んで処理するための要件が含まれます。** ストリーム処理は、通常は "無限" であるという点でバッチ処理とは異なります。つまり、ストリームは、一定の間隔ではなく永続的に処理する必要がある連続したデータ ソースです。**

Azure Stream Analytics には、ストリーミング ソースからのデータ ストリーム (Azure Event Hubs や Azure IoT ハブなど) を操作する "クエリ" を定義するために使用できるクラウド サービスが用意されています。** Azure Stream Analytics クエリを使うと、データ ストリームをデータ ストアに直接取り込んでさらに分析することや、テンポラル ウィンドウに基づいてデータのフィルター処理、集計、要約を行うことができます。

この演習では、Azure Stream Analytics を使用して、販売注文データのストリーム (オンライン小売アプリケーションから生成されるものなど) を処理します。 注文データは Azure Event Hubs に送信され、そこから Azure Stream Analytics ジョブがデータを読み取り、Azure Synapse Analytics に取り込みます。

この演習の所要時間は約 **45** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure リソースをプロビジョニングする

この演習では、データ レイク ストレージにアクセスできる Azure Synapse Analytics ワークスペースと、専用 SQL プールが必要です。 ストリーミング注文データを送信できる Azure Event Hubs 名前空間も必要です。

これらのリソースをプロビジョニングするために、PowerShell スクリプトと ARM テンプレートを組み合わせて使用します。

1. [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示された場合は、***PowerShell*** 環境を選択して、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#9723;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、この演習を含むリポジトリを複製します。

    ```
    rm -r dp-203 -f
    git clone https://github.com/MicrosoftLearning/dp-203-azure-data-engineer dp-203
    ```

5. リポジトリが複製されたら、次のコマンドを入力してこの演習用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```
    cd dp-203/Allfiles/labs/18
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 15 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Stream Analytics ドキュメントの「[Azure Stream Analytics へようこそ](https://learn.microsoft.com/azure/stream-analytics/stream-analytics-introduction)」の記事を確認しましょう。

## 専用 SQL プールにストリーミング データを取り込む

まず、Azure Synapse Analytics 専用 SQL プールのテーブルにデータ ストリームを直接取り込むことから始めましょう。

### ストリーミング ソースとデータベース テーブルを確認する

1. 設定スクリプトの実行が完了したら、[Cloud Shell] ペインを最小化します (後で戻ります)。 Azure portal で、作成した **dp203-*xxxxxxx*** リソース グループに移動します。このリソース グループには、Azure Synapse ワークスペース、データ レイク用のストレージ アカウント、専用 SQL プール、Event Hubs 名前空間が含まれていることに注目してください。
2. Synapse ワークスペースを選び、 **[概要]** ページの **[Synapse Studio を開く]** カードで、 **[開く]** を選び、新しいブラウザー タブで Synapse Studio を開きます。Synapse Studio は、Synapse Analytics ワークスペースの操作に使用できる Web ベースのインターフェイスです。
3. Synapse Studio の左側にある **&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。これにより、リソースの管理とデータ分析タスクの実行に使用するさまざまなページが Synapse Studio 内に表示されます。
4. **[管理]** ページの **[SQL プール]** セクションで、**sql*xxxxxxx*** 専用 SQL プールの行を選び、 **&#9655;** アイコンを使って再開します。
5. SQL プールの開始を待っている間に、Azure portal が表示されているブラウザー タブに戻り、[Cloud Shell] ペインを再び開きます。
6. [Cloud Shell] ペインで次のコマンドを入力して、100 個のシミュレートされた注文を Azure Event Hubs に送信するクライアント アプリを実行します。

    ```
    node ~/dp-203/Allfiles/labs/18/orderclient
    ```

7. 送信された注文データを観察します。各注文は製品 ID と数量で構成されています。
8. 注文クライアント アプリが完了したら、[Cloud Shell] ペインを最小化し、Synapse Studio ブラウザー タブに戻ります。
9. Synapse Studio の **[管理]** ページで、専用 SQL プールの状態が "**オンライン**" であることを確認し、**データ** ページに切り替え、 **[ワークスペース]** ペインで **SQL データベース**、**sql*xxxxxx*** SQL プール、**テーブル** を展開して **dbo. FactOrder** テーブルを表示します。
10. **dbo.FactOrder** テーブルの **[...]** メニューで **[New SQL script] (新しい SQL スクリプト)**  >  **[Select TOP 100 rows] (上位 100 行を選択する)** を選び、結果を確認します。 テーブルに **OrderDateTime**、**ProductID**、**Quantity** の列はありますが、現在はデータ行がないことを確認します。

### 注文データを取り込む Azure Stream Analytics ジョブを作成する

1. Azure portal が表示されているブラウザー タブに戻り、**dp203-*xxxxxxx*** リソース グループがプロビジョニングされたリージョンをメモしておきます。Stream Analytics ジョブはこれと<u>同じリージョン</u>に作成します。
2. **[ホーム]** ページで **[+ リソースの作成]** を選び、`Stream Analytics job` を検索します。 次に、次のプロパティを使用して **Stream Analytics ジョブ**を作成します。
    - **[基本]** :
        - **サブスクリプション**:お使いの Azure サブスクリプション
        - **リソース グループ**: 既存の **dp203-*xxxxxxx*** リソース グループを選択します。
        - **名前**: `ingest-orders`
        - **リージョン**: Synapse Analytics ワークスペースがプロビジョニングされているのと<u>同じ</u>リージョンを選びます。
        - **ホスティング環境**: クラウド
        - **ストリーミング ユニット**: 1
    - **ストレージ**:
        - **ストレージ アカウントの追加**: オン
        - **サブスクリプション**:お使いの Azure サブスクリプション
        - **ストレージ アカウント**: **datalake*xxxxxxx*** ストレージ アカウントを選択します
        - **認証モード**: 接続文字列
        - **Secure private data in storage account (ストレージ アカウントのプライベート データをセキュリティで保護する)** : オン
    - **タグ**:
        - *なし*
3. デプロイが完了するのを待ってから、デプロイされた Stream Analytics ジョブ リソースに移動します。

### イベント データ ストリームの入力を作成する

1. **ingest-orders** の概要ページで、 **[入力]** ページを選択します。 **[入力の追加]** メニューを使用して、次のプロパティを含む**イベント ハブ**入力を追加します。
    - **入力エイリアス**: `orders`
    - **サブスクリプションからイベント ハブを選択する:** オン
    - **サブスクリプション**:お使いの Azure サブスクリプション
    - **イベント ハブ名前空間**: **events*xxxxxxx*** Event Hubs 名前空間を選択します
    - **イベント ハブ名**: 既存の **eventhub*xxxxxxx*** イベント ハブを選択します
    - **イベント ハブ コンシューマー グループ**: **[既存のものの使用]** を選択し、 **$Default** コンシューマー グループを選択します
    - **認証モード**: システム割り当てマネージド ID を作成します
    - **パーティション キー**: "空白のままにします"**
    - **イベント シリアル化形式**: JSON
    - **[エンコード]**: UTF-8
2. 入力を保存し、作成されるまで待ちます。 いくつかの通知が表示されます。 "**接続テストが成功しました**" という通知を待ちます。

### SQL テーブルの出力を作成する

1. **ingest-orders** Stream Analytics ジョブの **[出力]** ページを表示します。 次に、 **[出力の追加]** メニューを使用して、次のプロパティを含む **Azure Synapse Analytics** の出力を追加します。
    - **出力エイリアス**: `FactOrder`
    - **Azure Synapse Analytics をサブスクリプションから選択する**: オン
    - **サブスクリプション**:お使いの Azure サブスクリプション
    - **データベース**: **sql*xxxxxxx* (synapse*xxxxxxx *)* * データベースを選びます
    - **認証モード**: SQL Server 認証
    - **ユーザー名**: SQLUser
    - **パスワード**: "セットアップ スクリプトの実行時に SQL プールに対して指定したパスワード。"**
    - **テーブル**: `FactOrder`
2. 出力を保存し、作成されるまで待ちます。 いくつかの通知が表示されます。 "**接続テストが成功しました**" という通知を待ちます。

### イベント ストリームを取り込むクエリを作る

1. **ingest-orders** Stream Analytics ジョブの **[クエリ]** ページを表示します。 次に、(イベント ハブに以前に取り込んだ販売注文イベントに基づいた) 入力プレビューが表示されるまでしばらく待ちます。
2. クライアント アプリから送信されたメッセージの **ProductID** と **Quantity** のフィールドと、その他の Event Hubs フィールド (イベントがイベント ハブに追加された日時を示す **EventProcessedUtcTime** フィールドなど) が入力データに含まれていることを確認します。
3. 次のように、既定のクエリを変更します。

    ```
    SELECT
        EventProcessedUtcTime AS OrderDateTime,
        ProductID,
        Quantity
    INTO
        [FactOrder]
    FROM
        [orders]
    ```

    このクエリを実行して、入力 (イベント ハブ) のフィールドを受け取り、出力 (SQL テーブル) に直接書き込むことを確認します。

4. クエリを保存します。

### ストリーミング ジョブを実行して注文データを取り込む

1. **ingest-orders** Stream Analytics ジョブの **[概要]** ページを表示し、 **[プロパティ]** タブでジョブの **[入力]** 、 **[クエリ]** 、 **[出力]** 、 **[関数]** を確認します。 **[入力]** と **[出力]** の数が 0 の場合は、 **[概要]** ページの **[&#8635; 更新]** ボタンを使って、**orders** 入力と **FactTable** 出力を表示します。
2. **[&#9655; 開始]** ボタンを選び、ストリーミング ジョブを今すぐ開始します。 ストリーミング ジョブが正常に開始されたことを通知されるまで待ちます。
3. [Cloud Shell] ペインを再度開き、次のコマンドを実行して、さらに 100 件の注文を送信します。

    ```
    node ~/dp-203/Allfiles/labs/18/orderclient
    ```

4. 注文クライアント アプリの実行中に Synapse Studio のブラウザー タブに切り替え、**dbo.FactOrder** テーブルから上位 100 行を選ぶために以前に実行したクエリを表示します。
5. **[&#9655; 実行]** ボタンを使ってクエリを再実行し、今回はテーブルにイベント ストリームの注文データが含まれていることを確認します (含まれていない場合は、1 分待ってクエリを再実行します)。 ジョブが実行され、注文イベントがイベント ハブに送信されている限り、Stream Analytics ジョブからすべての新しいイベント データがテーブルにプッシュされます。
6. **[管理]** ページで、**sql*xxxxxxxx*** 専用 SQL プールを一時停止します (不要な Azure の課金を防ぐため)。
7. Azure Portal が表示されているブラウザー タブに戻り、[Cloud Shell] ペインを最小化します。 次に **[&#128454; 停止]** ボタンを使って Stream Analytics ジョブを停止し、Stream Analytics ジョブが正常に停止したという通知を待ちます。

## データ レイクでストリーミング データを集計する

ここまでは、Stream Analytics ジョブを使ってストリーミング ソースから SQL テーブルにメッセージを取り込む方法について説明しました。 次は、Azure Stream Analytics を使って、テンポラル ウィンドウでデータを集計する方法 (このケースでは、5 秒ごとに販売された各製品の合計数量を計算します) を確認しましょう。 また、データ レイク BLOB ストアに結果を CSV 形式書き込むことで、ジョブに異なる種類の出力を使う方法についても確認します。

### 注文データを集計する Azure Stream Analytics ジョブを作成する

1. Azure portal の **[ホーム]** ページで、 **[+ リソースの作成]** を選び、`Stream Analytics job` を検索します。 次に、次のプロパティを使用して **Stream Analytics ジョブ**を作成します。
    - **[基本]** :
        - **サブスクリプション**:お使いの Azure サブスクリプション
        - **リソース グループ**: 既存の **dp203-*xxxxxxx*** リソース グループを選択します。
        - **名前**: `aggregate-orders`
        - **リージョン**: Synapse Analytics ワークスペースがプロビジョニングされているのと<u>同じ</u>リージョンを選びます。
        - **ホスティング環境**: クラウド
        - **ストリーミング ユニット**: 1
    - **ストレージ**:
        - **ストレージ アカウントの追加**: オン
        - **サブスクリプション**:お使いの Azure サブスクリプション
        - **ストレージ アカウント**: **datalake*xxxxxxx*** ストレージ アカウントを選択します
        - **認証モード**: 接続文字列
        - **Secure private data in storage account (ストレージ アカウントのプライベート データをセキュリティで保護する)** : オン
    - **タグ**:
        - *なし*

2. デプロイが完了するのを待ってから、デプロイされた Stream Analytics ジョブ リソースに移動します。

### 生の注文データの入力を作成する

1. **aggregate-orders** の概要ページで、 **[入力]** ページを選択します。 **[入力の追加]** メニューを使用して、次のプロパティを含む**イベント ハブ**入力を追加します。
    - **入力エイリアス**: `orders`
    - **サブスクリプションからイベント ハブを選択する:** オン
    - **サブスクリプション**:お使いの Azure サブスクリプション
    - **イベント ハブ名前空間**: **events*xxxxxxx*** Event Hubs 名前空間を選択します
    - **イベント ハブ名**: 既存の **eventhub*xxxxxxx*** イベント ハブを選択します
    - **イベント ハブ コンシューマー グループ**: 既存の **$Default** コンシューマー グループを選択します
    - **認証モード**: システム割り当てマネージド ID を作成します
    - **パーティション キー**: "空白のままにします"**
    - **イベント シリアル化形式**: JSON
    - **[エンコード]**: UTF-8
2. 入力を保存し、作成されるまで待ちます。 いくつかの通知が表示されます。 "**接続テストが成功しました**" という通知を待ちます。

### データ レイク ストアの出力を作成する

1. **aggregate-orders** Stream Analytics ジョブの **[出力]** ページを表示します。 次に **[出力の追加]** メニューを使用して、次のプロパティを含む **BLOB Storage/ADLS Gen2** 出力を追加します。
    - **出力エイリアス:** `datalake`
    - **Select Blob storage/ADLS Gen2 from your subscriptions from your subscriptions (サブスクリプションから BLOB ストレージ/ADLS Gen2 を選択する)** : オン
    - **サブスクリプション**:お使いの Azure サブスクリプション
    - **ストレージ アカウント**: **datalake*xxxxxxx*** ストレージ アカウントを選びます
    - **コンテナー**: **[既存のものの使用]** を選択し、一覧から**ファイル** コンテナーを選択します
    - **認証モード**: 接続文字列
    - **イベント シリアル化形式**: CSV - コンマ (,)
    - **[エンコード]**: UTF-8
    - **書き込みモード**: 結果の到着時に追加します
    - **パス パターン**: `{date}`
    - **日付の形式**: YYYY/MM/DD
    - **時刻の形式**: "適用なし"**
    - **最小行数**: 20
    - **最大時間**: 0 時間、1 分、0 秒
2. 出力を保存し、作成されるまで待ちます。 いくつかの通知が表示されます。 "**接続テストが成功しました**" という通知を待ちます。

### イベント データを集計するクエリを作成する

1. **aggregate-orders** Stream Analytics ジョブの **[クエリ]** ページを表示します。
2. 次のように、既定のクエリを変更します。

    ```
    SELECT
        DateAdd(second,-5,System.TimeStamp) AS StartTime,
        System.TimeStamp AS EndTime,
        ProductID,
        SUM(Quantity) AS Orders
    INTO
        [datalake]
    FROM
        [orders] TIMESTAMP BY EventProcessedUtcTime
    GROUP BY ProductID, TumblingWindow(second, 5)
    HAVING COUNT(*) > 1
    ```

    このクエリで、**System.Timestamp** (**EventProcessedUtcTime** フィールドに基づく) を使って、各製品 ID の合計数量が計算される各 5 秒の "タンブリング" (重複しないシーケンシャル) 枠の開始と終了が定義されることを確認します。**

3. クエリを保存します。

### 注文データを集計するストリーミング ジョブを実行する

1. **aggregate-orders** Stream Analytics ジョブの **[概要]** ページを表示し、 **[プロパティ]** タブでジョブの **[入力]** 、 **[クエリ]** 、 **[出力]** 、 **[関数]** を確認します。 **[入力]** と **[出力]** の数が 0 の場合は、 **[概要]** ページの **[&#8635; 更新]** ボタンを使って、**orders** 入力と **datalake** 出力を表示します。
2. **[&#9655; 開始]** ボタンを選び、ストリーミング ジョブを今すぐ開始します。 ストリーミング ジョブが正常に開始されたことを通知されるまで待ちます。
3. [Cloud Shell] ペインを再度開き、次のコマンドを実行して、さらに 100 件の注文を送信します。

    ```
    node ~/dp-203/Allfiles/labs/18/orderclient
    ```

4. 注文アプリが完了したら、[Cloud Shell] ペインを最小化します。 次に Synapse Studio ブラウザー タブに切り替え、 **[データ]** ページの **[リンク済み]** タブで **Azure Data Lake Storage Gen2** > **synapse*xxxxxx* (primary - datalake*xxxxx *)* * を展開して **[ファイル (プライマリ)]** コンテナーを選びます。
5. **ファイル** コンテナーが空の場合は、1 分ほど待ってから **[&#8635; 更新]** を使って表示を更新します。 最終的に、現在の年の名前が付いたフォルダーが表示されます。 これには、月と日のフォルダーが含まれています。
6. 年のフォルダーを選び、 **[New SQL script] (新しい SQL スクリプト)** メニューで **[Select TOP 100 rows] (上位 100 行を選択する)** を選びます。 次に **[ファイルの種類]** を **[テキスト形式]** に設定し、設定を適用します。
7. 開いたクエリ ペインでクエリを変更し、次のように `HEADER_ROW = TRUE` パラメーターを追加します。

    ```sql
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/2023/**',
            FORMAT = 'CSV',
            PARSER_VERSION = '2.0',
            HEADER_ROW = TRUE
        ) AS [result]
    ```

8. **[&#9655; 実行]** ボタンを使って SQL クエリを実行し、結果を確認します。5 秒単位で注文された各製品の数量が表示されます。
9. Azure Portal が表示されているブラウザー タブに戻り、 **[&#128454; 停止]** ボタンを使って Stream Analytics ジョブを停止し、Stream Analytics ジョブが正常に停止したという通知を待ちます。

## Azure リソースを削除する

Azure Stream Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Azure Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. (管理対象リソース グループではなく) Azure Synapse、Event Hubs、Stream Analytics リソースが含まれる **dp203-*xxxxxxxx*** リソース グループを選びます。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後、この演習で作成されたリソースは削除されます。
