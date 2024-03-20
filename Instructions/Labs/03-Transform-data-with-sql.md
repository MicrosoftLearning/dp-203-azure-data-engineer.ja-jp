---
lab:
  title: サーバーレス SQL プールを使用してデータを変換する
  ilt-use: Lab
---

# サーバーレス SQL プールを使用してファイルを変換する

データ "アナリスト" は、多くの場合、SQL を使用して、分析とレポートのためにデータのクエリを実行します。** データ "エンジニア" は、SQL を使用してデータを操作および変換することもできます。多くの場合、データ インジェスト パイプラインまたは抽出、変換、読み込み (ETL) プロセスの一部として使用します。**

この演習では、Azure Synapse Analytics でサーバーレス SQL プールを使って、ファイル内のデータを変換します。

この演習の所要時間は約 **30** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Synapse Analytics ワークスペースをプロビジョニングする

Data Lake Storage にアクセスできる Azure Synapse Analytics ワークスペースが必要です。 データ レイク内のファイルに対してクエリを実行するには、組み込みのサーバーレス SQL プールを使用できます。

この演習では、Azure Synapse Analytics ワークスペースをプロビジョニングするために、PowerShell スクリプトと ARM テンプレートを組み合わせて使用します。

1. [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示された場合は、***PowerShell*** 環境を選択して、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#9723;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

    ```
    rm -r dp-203 -f
    git clone https://github.com/MicrosoftLearning/dp-203-azure-data-engineer dp-203
    ```

5. リポジトリがクローンされたら、次のコマンドを入力してこの演習用のフォルダーに移動し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```
    cd dp-203/Allfiles/labs/03
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 10 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Synapse Analytics ドキュメントの「[Synapse SQL での CETAS](https://docs.microsoft.com/azure/synapse-analytics/sql/develop-tables-cetas)」の記事をご確認ください。

## ファイル内のデータのクエリを実行する

このスクリプトを使用すると、Azure Synapse Analytics ワークスペースと、データ レイクをホストする Azure Storage アカウントがプロビジョニングされ、いくつかのデータ ファイルがデータ レイクにアップロードされます。

### データ レイク内のファイルを表示する

1. スクリプトが完了したら、Azure portal で、作成された **dp203-xxxxxxx** リソース グループに移動して、Synapse ワークスペースを選択します。
2. Synapse ワークスペースの **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。メッセージが表示された場合はサインインします。
3. Synapse Studio の左側にある **&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。これにより、リソースの管理とデータ分析タスクの実行に使用するさまざまなページが Synapse Studio 内に表示されます。
4. **[データ]** ページで **[リンク]** タブを表示して、**synapse*xxxxxxx* (Primary - datalake*xxxxxxx*)** のような名前の Azure Data Lake Storage Gen2 ストレージ アカウントへのリンクがワークスペースに含まれていることを確認します。
5. ストレージ アカウントを展開して、そこに **files** という名前のファイル システム コンテナーが含まれていることを確認します。
6. この **files** コンテナーを選択し、そこに **sales** という名前のフォルダーが含まれていることに注意してください。 このフォルダーには、クエリを実行するデータ ファイルが含まれています。
7. 含まれている **sales** フォルダーと **csv** フォルダーを開き、3 年間の売上データの .csv ファイルがこのフォルダーに含まれていることを確認します。
8. いずれかのファイルを右クリックし、 **[プレビュー]** を選択して、そこに含まれているデータを表示します。 ファイルにヘッダー行が含まれていることに注目してください。
9. プレビューを閉じ、 **&#8593;** ボタンを使用して、**sales** フォルダーに戻ります。

### SQL を使用して CSV ファイルのクエリを実行する

1. **csv** フォルダーを選択し、ツール バーの **[新しい SQL スクリプト]** の一覧で、 **[上位 100 行を選択]** を選択します。
2. **[ファイルの種類]** の一覧で、 **[テキスト形式]** を選択し、設定を適用して、フォルダー内のデータのクエリを実行する新しい SQL スクリプトを開きます。
3. 作成された **SQL Script 1** の **[プロパティ]** ペインで、名前を **Query Sales CSV files** に変更し、結果の設定を **[すべての行]** に変更します。 ツール バーで **[発行]** を選択してスクリプトを保存し、ツール バーの右端にある **[プロパティ]** ボタン ( **&#128463;<sub>*</sub>** に似ています) を使用して、 **[プロパティ]** ペインを非表示にします。
4. 生成された SQL コードを確認します。これは次のようになります。

    ```SQL
    -- This is auto-generated code
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/csv/**',
            FORMAT = 'CSV',
            PARSER_VERSION='2.0'
        ) AS [result]
    ```

    このコードでは、OPENROWSET を使用して、sales フォルダー内の CSV ファイルからデータを読み取り、最初の 100 行のデータを取得します。

5. この場合、データ ファイルには最初の行に列名が含まれます。次に示すように、`HEADER_ROW = TRUE` パラメーターを `OPENROWSET` 句に追加するようにクエリを変更します (前のパラメーターの後にコンマを追加することを忘れないでください)。

    ```SQL
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/csv/**',
            FORMAT = 'CSV',
            PARSER_VERSION='2.0',
            HEADER_ROW = TRUE
        ) AS [result]
    ```

6. **[接続先]** ボックスの一覧で、**[組み込み]** が選択されていることを確認します。これは、ワークスペースで作成された組み込みの SQL プールを表します。 次に、ツールバーの **[&#9655; 実行]** ボタンを使用して SQL コードを実行し、結果を確認します。これは次のようになります。

    | SalesOrderNumber | SalesOrderLineNumber | OrderDate | CustomerName | EmailAddress | Item | Quantity | UnitPrice | TaxAmount |
    | -- | -- | -- | -- | -- | -- | -- | -- | -- |
    | SO43701 | 1 | 2019-07-01 | Christy Zhu | christy12@adventure-works.com |Mountain-100 Silver, 44 | 1 | 3399.99 | 271.9992 |
    | ... | ... | ... | ... | ... | ... | ... | ... | ... |

7. 変更をスクリプトに発行し、スクリプト ペインを閉じます。

## CREATE EXTERNAL TABLE AS SELECT (CETAS) ステートメントを使用してデータを変換する

SQL を使用してファイル内のデータを変換し、結果を別のファイルに保持する簡単な方法は、CREATE EXTERNAL TABLE AS SELECT (CETAS) ステートメントを使用することです。 このステートメントは、クエリの要求に基づいてテーブルを作成しますが、テーブルのデータはデータ レイク内のファイルとして保存されます。 変換されたデータは、外部テーブルを介してクエリを実行することも、ファイル システムで直接アクセスすることもできます (たとえば、変換されたデータをデータ ウェアハウスに読み込むダウンストリーム プロセスに含める場合など)。

### 外部データ ソースとファイル形式を作成する

データベースに外部データ ソースを定義すると、それを使用して、外部テーブルのファイルを保存するデータ レイクの場所を参照できます。 外部ファイル形式を使用すると、Parquet や CSV などのファイルの形式を定義できます。 これらのオブジェクトを使用して外部テーブルを操作するには、既定の**マスター** データベース以外のデータベースに作成する必要があります。

1. Synapse Studio の **[開発]** ページの **[+]** メニューで、 **[SQL スクリプト]** を選択します。
2. 新しいスクリプト ペインで、次のコードを追加して (*datalakexxxxxxx* を Data Lake Storage アカウントの名前に置き換えます)、新しいデータベースを作成し、それに外部データ ソースを追加します。

    ```sql
    -- Database for sales data
    CREATE DATABASE Sales
      COLLATE Latin1_General_100_BIN2_UTF8;
    GO;
    
    Use Sales;
    GO;
    
    -- External data is in the Files container in the data lake
    CREATE EXTERNAL DATA SOURCE sales_data WITH (
        LOCATION = 'https://datalakexxxxxxx.dfs.core.windows.net/files/'
    );
    GO;
    
    -- Format for table files
    CREATE EXTERNAL FILE FORMAT ParquetFormat
        WITH (
                FORMAT_TYPE = PARQUET,
                DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'
            );
    GO;
    ```

3. スクリプトのプロパティを変更して名前を **Create Sales DB**に変更し、発行します。
4. スクリプトが **Built-in** SQL プールと **master** データベースに接続されていることを確認し、スクリプトを実行します。
5. **[データ]** ページに戻り、Synapse Studio の右上にある **&#8635;** ボタンを使用してページを更新します。 **[データ]** ペインの **[ワークスペース]** タブを表示します。ここに、 **[SQL データベース]** の一覧が表示されるようになりました。 この一覧を展開して、**Sales** データベースが作成されたことを確認します。
6. **Sales** データベース、その **External Resources** フォルダー、その下の **External data sources** フォルダーを順に展開すると、作成した **sales_data** 外部データ ソースが表示されます。

### 外部テーブルを作成する

1. Synapse Studio の **[開発]** ページの **[+]** メニューで、 **[SQL スクリプト]** を選択します。
2. 新しいスクリプト ペインで、外部データ ソースを使用して CSV 販売ファイルからデータを取得および集計する次のコードを追加します。**BULK** パスは、データ ソースが定義されているフォルダーの場所から相対的であることにご注意ください。

    ```sql
    USE Sales;
    GO;
    
    SELECT Item AS Product,
           SUM(Quantity) AS ItemsSold,
           ROUND(SUM(UnitPrice) - SUM(TaxAmount), 2) AS NetRevenue
    FROM
        OPENROWSET(
            BULK 'sales/csv/*.csv',
            DATA_SOURCE = 'sales_data',
            FORMAT = 'CSV',
            PARSER_VERSION = '2.0',
            HEADER_ROW = TRUE
        ) AS orders
    GROUP BY Item;
    ```

3. スクリプトを実行します。 結果は次のようになります。

    | 製品 | ItemsSold | NetRevenue |
    | -- | -- | -- |
    | AWC Logo Cap | 1063 | 8791.86 |
    | ... | ... | ... |

4. 次のように、SQL コードを変更して、クエリの結果を外部テーブルに保存します。

    ```sql
    CREATE EXTERNAL TABLE ProductSalesTotals
        WITH (
            LOCATION = 'sales/productsales/',
            DATA_SOURCE = sales_data,
            FILE_FORMAT = ParquetFormat
        )
    AS
    SELECT Item AS Product,
        SUM(Quantity) AS ItemsSold,
        ROUND(SUM(UnitPrice) - SUM(TaxAmount), 2) AS NetRevenue
    FROM
        OPENROWSET(
            BULK 'sales/csv/*.csv',
            DATA_SOURCE = 'sales_data',
            FORMAT = 'CSV',
            PARSER_VERSION = '2.0',
            HEADER_ROW = TRUE
        ) AS orders
    GROUP BY Item;
    ```

5. スクリプトを実行します。 今回は出力はありませんが、クエリの結果に基づいて外部テーブルを作成する必要があります。
6. スクリプトに **Create ProductSalesTotals table** という名前を付け、発行します。
7. **[データ]** ページの **[ワークスペース]** タブで、**Sales** SQL データベースの **External tables** フォルダーの内容を表示して、**ProductSalesTotals** という名前の新しいテーブルが作成されたことを確認します。
8. **ProductSalesTotals** テーブルの **[...]** メニューで、 **[新しい SQL スクリプト]**  >  **[上位 100 行を選択]** の順に選択します。 次に、結果のスクリプトを実行し、集計された製品売上データが返されることを確認します。
9. データ レイクのファイル システムが含まれている **[ファイル]** タブで、**sales** フォルダーの内容を表示し (必要に応じてビューを更新します)、新しい **productsales** フォルダーが作成されていることを確認します。
10. **productsales** フォルダーで、ABC123DE----.parquet のような名前の 1 つ以上のファイルが作成されていることを確認します。 これらのファイルには、集計された製品売上データが含まれています。 これを証明するには、いずれかのファイルを選択し、 **[新しい SQL スクリプト]**  >  **[上位 100 行を選択]** メニューを使用して直接クエリを実行できます。

## ストアド プロシージャにデータ変換をカプセル化する

データを頻繁に変換する必要がある場合は、ストアド プロシージャを使用して CETAS ステートメントをカプセル化できます。

1. Synapse Studio の **[開発]** ページの **[+]** メニューで、 **[SQL スクリプト]** を選択します。
2. 新しいスクリプト ペインで、次のコードを追加して **Sales** データベースにストアド プロシージャを作成します。これにより、売上が年別に集計され、結果が外部テーブルに保存されます。

    ```sql
    USE Sales;
    GO;
    CREATE PROCEDURE sp_GetYearlySales
    AS
    BEGIN
        -- drop existing table
        IF EXISTS (
                SELECT * FROM sys.external_tables
                WHERE name = 'YearlySalesTotals'
            )
            DROP EXTERNAL TABLE YearlySalesTotals
        -- create external table
        CREATE EXTERNAL TABLE YearlySalesTotals
        WITH (
                LOCATION = 'sales/yearlysales/',
                DATA_SOURCE = sales_data,
                FILE_FORMAT = ParquetFormat
            )
        AS
        SELECT YEAR(OrderDate) AS CalendarYear,
                SUM(Quantity) AS ItemsSold,
                ROUND(SUM(UnitPrice) - SUM(TaxAmount), 2) AS NetRevenue
        FROM
            OPENROWSET(
                BULK 'sales/csv/*.csv',
                DATA_SOURCE = 'sales_data',
                FORMAT = 'CSV',
                PARSER_VERSION = '2.0',
                HEADER_ROW = TRUE
            ) AS orders
        GROUP BY YEAR(OrderDate)
    END
    ```

3. スクリプトを実行してストアド プロシージャを作成します。
4. 先ほど実行したコードの下に、次のコードを追加してストアド プロシージャを呼び出します。

    ```sql
    EXEC sp_GetYearlySales;
    ```

5. 追加した `EXEC sp_GetYearlySales;` ステートメントのみを選択し、 **[&#9655; 実行]** ボタンを使ってこれを実行します。
6. データ レイクのファイル システムが含まれている **[ファイル]** タブで、**sales** フォルダーの内容を表示し (必要に応じてビューを更新します)、新しい **yearlysales** フォルダーが作成されていることを確認します。
7. **yearlysales** フォルダーで、年間売上の集計データを含む Parquet ファイルが作成されていることを確認します。
8. SQL スクリプトに戻り、`EXEC sp_GetYearlySales;` ステートメントを再実行し、エラーが発生することを確認します。

    スクリプトによって外部テーブルが切断されても、データを含むフォルダーは削除されません。 ストアド プロシージャを再実行するには (たとえば、スケジュールされたデータ変換パイプラインの一部として)、古いデータを削除する必要があります。

9. **[ファイル]** タブに戻り、**sales** フォルダーを表示します。 次に、**yearlysales** フォルダーを選択して削除します。
10. SQL スクリプトに戻り、`EXEC sp_GetYearlySales;` ステートメントを再実行します。 今回は、操作が成功し、新しいデータ ファイルが生成されます。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースの **dp203-xxxxxxx** リソース グループ (管理対象リソース グループではありません) を選択し、Synapse ワークスペースとお使いのワークスペースのストレージ アカウントが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
