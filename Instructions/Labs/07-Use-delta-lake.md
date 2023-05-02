---
lab:
  title: Azure Synapse Analytics で Delta Lake を使用する
  ilt-use: Lab
---

# Azure Synapse Analytics で Delta Lake と Spark を使用する

Delta Lake は、データ レイクの上にトランザクション データ ストレージ レイヤーを構築するためのオープンソース プロジェクトです。 Delta Lake では、バッチ データ操作とストリーミング データ操作の両方にリレーショナル セマンティクスのサポートが追加され、Apache Spark を使用して、データ レイク内の基になるファイルに基づくテーブル内のデータを処理しクエリを実行できる *Lakehouse* アーキテクチャを作成できます。

この演習の所要時間は約 **40** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Synapse Analytics ワークスペースをプロビジョニングする

データ レイク ストレージへのアクセス権を持つ Azure Synapse Analytics ワークスペースと、データ レイク内のファイルのクエリ実行と処理に使用できる Apache Spark プールが必要です。

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

5. リポジトリが複製されたら、次のコマンドを入力してこの演習用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```
    cd dp-203/Allfiles/labs/07
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 10 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Synapse Analytics ドキュメントの記事「[Delta Lake とは](https://docs.microsoft.com/azure/synapse-analytics/spark/apache-spark-what-is-delta-lake)」を確認してください。

## デルタ テーブルを作成する

このスクリプトを使用すると、Azure Synapse Analytics ワークスペースと、データ レイクをホストする Azure Storage アカウントがプロビジョニングされ、データ ファイルがデータ レイクにアップロードされます。

### データ レイク内のデータを探索する

1. スクリプトが完了したら、Azure portal で、作成された **dp203-xxxxxxx** リソース グループに移動して、Synapse ワークスペースを選択します。
2. Synapse ワークスペースの **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。メッセージが表示された場合はサインインします。
3. Synapse Studio の左側にある **&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。これにより、リソースの管理とデータ分析タスクの実行に使用するさまざまなページが Synapse Studio 内に表示されます。
4. **[データ]** ページで **[リンク]** タブを表示して、**synapse*xxxxxxx* (Primary - datalake*xxxxxxx*)** のような名前の Azure Data Lake Storage Gen2 ストレージ アカウントへのリンクがワークスペースに含まれていることを確認します。
5. ストレージ アカウントを展開して、そこに **files** という名前のファイル システム コンテナーが含まれていることを確認します。
6. **files** コンテナーを選択して、この中に **products** という名前のフォルダーがあることに注目します。 このフォルダーに、この演習で使用するデータがあります。
7. **products** フォルダーを開き、この中に **products.csv** という名前のファイルがあることを確認します。
8. **products.csv** を選択したら、ツール バーの **[新しいノートブック]** リストで **[データフレームに読み込む]** を選択します。
9. **[ノートブック 1]** ペインが表示されたら、 **[アタッチ先]** ボックスの一覧で、**sparkxxxxxxx** Spark プールを選択し、 **[言語]** が **[PySpark (Python)]** に設定されていることを確認します。
10. ノートブックの最初の (および唯一の) セルのコードを確認します。次のようになります。

    ```Python
    %%pyspark
    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/products/products.csv', format='csv'
    ## If header exists uncomment line below
    ##, header=True
    )
    display(df.limit(10))
    ```

11. *,header=True* 行のコメントを解除します (products.csv のファイルには最初の行に列ヘッダーがあるため)。コードは次のようになります。

    ```Python
    %%pyspark
    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/products/products.csv', format='csv'
    ## If header exists uncomment line below
    , header=True
    )
    display(df.limit(10))
    ```

12. コード セルの左側にある **&#9655;** アイコンを使用して実行し、結果が表示されるまで待ちます。 ノートブックで初めてセルを実行すると、Spark プールが開始されます。結果が返されるまでに 1 分ほどかかることがあります。 最終的には、セルの下に結果が表示され、次のようになります。

    | ProductID | ProductName | カテゴリ | ListPrice |
    | -- | -- | -- | -- |
    | 771 | Mountain-100 Silver, 38 | マウンテン バイク | 3399.9900 |
    | 772 | Mountain-100 Silver, 42 | マウンテン バイク | 3399.9900 |
    | ... | ... | ... | ... |

### ファイル データをデルタ テーブルに読み込む

1. 最初のコード セルから返された結果の下で、 **[+ コード]** ボタンを使用して新しいコード セルを追加します。 新しいセルに次のコードを入力して実行します。

    ```Python
    delta_table_path = "/delta/products-delta"
    df.write.format("delta").save(delta_table_path)
    ```

2. **files** タブで、ツール バーの **[&#8593;]** アイコンを使用して **files** コンテナーのルートに戻り、**delta** という名前の新しいフォルダーが作成されていることに注目します。 このフォルダーを開き、その中にある **products-delta** テーブルを開くと、データを含む Parquet 形式のファイルが表示されます。

3. **[ノートブック 1]** タブに戻り、別の新しいコード セルを追加します。 この新しいセルに次のコードを追加して実行します。

    ```Python
    from delta.tables import *
    from pyspark.sql.functions import *

    # Create a deltaTable object
    deltaTable = DeltaTable.forPath(spark, delta_table_path)

    # Update the table (reduce price of product 771 by 10%)
    deltaTable.update(
        condition = "ProductID == 771",
        set = { "ListPrice": "ListPrice * 0.9" })

    # View the updated data as a dataframe
    deltaTable.toDF().show(10)
    ```

    データが **DeltaTable** オブジェクトに読み込まれ、更新されます。 更新がクエリ結果に反映されていることを確認できます。

4. 別の新しいコード セルを追加して次のコードを含め、実行します。

    ```Python
    new_df = spark.read.format("delta").load(delta_table_path)
    new_df.show(10)
    ```

    このコードを実行して、**DeltaTable** オブジェクトを介して行った変更が永続化されていることを確認し、データ レイク内の場所からデータ フレームにデルタ テーブル データを読み込みます。

5. 先ほど実行したコードを次のように変更します。デルタ レイクの "タイム トラベル" 機能を使用するオプションを指定して、以前のバージョンのデータを表示します。**

    ```Python
    new_df = spark.read.format("delta").option("versionAsOf", 0).load(delta_table_path)
    new_df.show(10)
    ```

    変更したコードを実行すると、結果に元のバージョンのデータが表示されます。

6. 別の新しいコード セルを追加して次のコードを含め、実行します。

    ```Python
    deltaTable.history(10).show(20, False, True)
    ```

    テーブルの直近 20 件の変更履歴が表示されます。ここでは、2 件 (元の作成と行った更新) あるはずです。

## カタログ テーブルを作成する

ここまで、テーブルの基になった Parquet ファイルが含まれるフォルダーからデータを読み込むことで、デルタ テーブルを操作しました。 データをカプセル化する "カタログ テーブル" を定義し、SQL コードで参照できる名前付きテーブル エンティティを提供できます。** Spark では、デルタ レイク用に次の 2 種類のカタログ テーブルがサポートされています。

- テーブル データを含む Parquet ファイルへのパスで定義される "外部" テーブル。**
- Spark プール用の Hive メタストアに定義される "マネージド" テーブル。**

### 外部テーブルを作成する

1. 新しいコード セルに、次のコードを追加して実行します。

    ```Python
    spark.sql("CREATE DATABASE AdventureWorks")
    spark.sql("CREATE TABLE AdventureWorks.ProductsExternal USING DELTA LOCATION '{0}'".format(delta_table_path))
    spark.sql("DESCRIBE EXTENDED AdventureWorks.ProductsExternal").show(truncate=False)
    ```

    このコードを実行して、**AdventureWorks** という名前の新しいデータベースを作成し、先ほど定義した Parquet ファイルファイルへのパスに基づいて、そのデータベース内に **ProductsExternal** という外部テーブルを作成します。 次に、テーブルのプロパティの説明が表示されます。 **Location** プロパティが、指定したパスであることに注目します。

2. 新しいコード セルを追加し、次のコードを入力して実行します。

    ```sql
    %%sql

    USE AdventureWorks;

    SELECT * FROM ProductsExternal;
    ```

    このコードでは、SQL を使用してコンテキストを **AdventureWorks** データベースに切り替え (データは返されない)、**ProductsExternal** テーブル (Delta Lake テーブル内の製品データを含む結果セットが返される) に対してクエリを実行します。

### マネージド テーブルを作成する

1. 新しいコード セルに、次のコードを追加して実行します。

    ```Python
    df.write.format("delta").saveAsTable("AdventureWorks.ProductsManaged")
    spark.sql("DESCRIBE EXTENDED AdventureWorks.ProductsManaged").show(truncate=False)
    ```

    このコードを実行して、最初に **products.csv** ファイルから読み込んだ DataFrame (製品 771 の価格を更新する前) に基づいて **ProductsManaged** という名前のマネージド テーブルを作成します。 このテーブルで使用される Parquet ファイルのパスは指定しません。これは Hive メタストアで管理され、テーブルの説明の **Location** プロパティ (**files/synapse/workspaces/synapsexxxxxxx/warehouse** パス内) に表示されます。

2. 新しいコード セルを追加し、次のコードを入力して実行します。

    ```sql
    %%sql

    USE AdventureWorks;

    SELECT * FROM ProductsManaged;
    ```

    このコードでは、SQL を使用して **ProductsManaged** テーブルに対してクエリを実行します。

### 外部テーブルとマネージド テーブルを比較する

1. 新しいコード セルに、次のコードを追加して実行します。

    ```sql
    %%sql

    USE AdventureWorks;

    SHOW TABLES;
    ```

    このコードを実行して、**AdventureWorks** データベース内のテーブルを一覧表示します。

2. コード セルを次のように変更して、実行します。

    ```sql
    %%sql

    USE AdventureWorks;

    DROP TABLE IF EXISTS ProductsExternal;
    DROP TABLE IF EXISTS ProductsManaged;
    ```

    このコードを実行して、メタストアからテーブルを削除します。

3. **files** タブに戻り、**files/delta/products-delta** フォルダーを表示します。 データ ファイルがこの場所にまだ存在することに注目します。 外部テーブルを削除すると、テーブルはメタストアから削除されましたが、データ ファイルはそのまま残りました。
4. **files/synapse/workspaces/synapsexxxxxxx/warehouse** フォルダーを表示し、**ProductsManaged** テーブル データのフォルダーがないことに注目します。 マネージド テーブルを削除すると、テーブルはメタストアから削除され、テーブルのデータ ファイルも削除されます。

### SQL を使用してテーブルを作成する

1. 新しいコード セルを追加し、次のコードを入力して実行します。

    ```sql
    %%sql

    USE AdventureWorks;

    CREATE TABLE Products
    USING DELTA
    LOCATION '/delta/products-delta';
    ```

2. 新しいコード セルを追加し、次のコードを入力して実行します。

    ```sql
    %%sql

    USE AdventureWorks;

    SELECT * FROM Products;
    ```

    既存の Delta Lake テーブル フォルダーに対して新しいカタログ テーブルが作成され、以前に行った変更が反映されていることを確認します。

## ストリーミング データにデルタ テーブルを使用する

Delta Lake では、ストリーミング データがサポートされています。 デルタ テーブルは、Spark 構造化ストリーミング API を使用して作成されたデータ ストリームの "シンク" または "ソース" に指定できます。** ** この例では、モノのインターネット (IoT) のシミュレーション シナリオで、一部のストリーミング データのシンクにデルタ テーブルを使用します。

1. **[ノートブック 1]** タブに戻り、新しいコード セルを追加します。 この新しいセルに次のコードを追加して実行します。

    ```python
    from notebookutils import mssparkutils
    from pyspark.sql.types import *
    from pyspark.sql.functions import *

    # Create a folder
    inputPath = '/data/'
    mssparkutils.fs.mkdirs(inputPath)

    # Create a stream that reads data from the folder, using a JSON schema
    jsonSchema = StructType([
    StructField("device", StringType(), False),
    StructField("status", StringType(), False)
    ])
    iotstream = spark.readStream.schema(jsonSchema).option("maxFilesPerTrigger", 1).json(inputPath)

    # Write some event data to the folder
    device_data = '''{"device":"Dev1","status":"ok"}
    {"device":"Dev1","status":"ok"}
    {"device":"Dev1","status":"ok"}
    {"device":"Dev2","status":"error"}
    {"device":"Dev1","status":"ok"}
    {"device":"Dev1","status":"error"}
    {"device":"Dev2","status":"ok"}
    {"device":"Dev2","status":"error"}
    {"device":"Dev1","status":"ok"}'''
    mssparkutils.fs.put(inputPath + "data.txt", device_data, True)
    print("Source stream created...")
    ```

    メッセージ「*Source stream created...* 」が確実に出力されるようにします。 先ほど実行したコードによって、一部のデータが保存されているフォルダーに基づいてストリーミング データ ソースが作成されました。これは、架空の IoT デバイスからの読み取り値を表しています。

2. 新しいコード セルに、次のコードを追加して実行します。

    ```python
    # Write the stream to a delta table
    delta_stream_table_path = '/delta/iotdevicedata'
    checkpointpath = '/delta/checkpoint'
    deltastream = iotstream.writeStream.format("delta").option("checkpointLocation", checkpointpath).start(delta_stream_table_path)
    print("Streaming to delta sink...")
    ```

    このコードを実行して、ストリーミング デバイス データをデルタ形式で書き込みます。

3. 新しいコード セルに、次のコードを追加して実行します。

    ```python
    # Read the data in delta format into a dataframe
    df = spark.read.format("delta").load(delta_stream_table_path)
    display(df)
    ```

    このコードを実行して、ストリームされたデータをデルタ形式でデータフレームに読み取ります。 ストリーミング データを読み込むコードが、デルタ フォルダーから静的データを読み込むために使用するコードと変わらないことに注目します。

4. 新しいコード セルに、次のコードを追加して実行します。

    ```python
    # create a catalog table based on the streaming sink
    spark.sql("CREATE TABLE IotDeviceData USING DELTA LOCATION '{0}'".format(delta_stream_table_path))
    ```

    このコードを実行して、デルタ フォルダーに基づいて (**既定**のデータベース内に) **IotDeviceData** という名前のカタログ テーブルを作成します。 繰り返しになりますが、このコードは非ストリーミング データに使用されるコードと同じです。

5. 新しいコード セルに、次のコードを追加して実行します。

    ```sql
    %%sql

    SELECT * FROM IotDeviceData;
    ```

    このコードを実行して、ストリーミング ソースのデバイス データが含まれる **IotDeviceData** テーブルに対してクエリを実行します。

6. 新しいコード セルに、次のコードを追加して実行します。

    ```python
    # Add more data to the source stream
    more_data = '''{"device":"Dev1","status":"ok"}
    {"device":"Dev1","status":"ok"}
    {"device":"Dev1","status":"ok"}
    {"device":"Dev1","status":"ok"}
    {"device":"Dev1","status":"error"}
    {"device":"Dev2","status":"error"}
    {"device":"Dev1","status":"ok"}'''

    mssparkutils.fs.put(inputPath + "more-data.txt", more_data, True)
    ```

    このコードを実行して、さらに架空のデバイス データをストリーミング ソースに書き込みます。

7. 新しいコード セルに、次のコードを追加して実行します。

    ```sql
    %%sql

    SELECT * FROM IotDeviceData;
    ```

    このコードを実行して、**IotDeviceData** テーブルに対してもう一度クエリを実行します。今度は、ストリーミング ソースに追加された追加データが含まれるはずです。

8. 新しいコード セルに、次のコードを追加して実行します。

    ```python
    deltastream.stop()
    ```

    このコードを実行して、ストリームを停止します。

## サーバーレス SQL プールからデルタ テーブルに対してクエリを実行する

Spark プールに加え、Azure Synapse Analytics では、組み込みのサーバーレス SQL プールを利用できます。 このプールのリレーショナル データベース エンジンを使用して、SQL を使用してデルタ テーブルに対してクエリを実行できます。

1. **files** タブで、**files/delta** フォルダーを参照します。
2. **products-delta** フォルダーを選択し、ツール バーの **[新しい SQL スクリプト]** ドロップダウン リストで、 **[上位 100 行を選択]** を選択します。
3. **[上位 100 行を選択]** ペインの **[ファイルの種類]** リストで、 **[デルタ形式]** を選択し、 **[適用]** を選択します。
4. 生成される SQL コードを確認します。次のように表示されるはずです。

    ```sql
    -- This is auto-generated code
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/delta/products-delta/',
            FORMAT = 'DELTA'
        ) AS [result]
    ```

5. **[&#9655; 実行]** アイコンを使用してスクリプトを実行し、結果を確認します。 次のように表示されるはずです。

    | ProductID | ProductName | カテゴリ | ListPrice |
    | -- | -- | -- | -- |
    | 771 | Mountain-100 Silver, 38 | マウンテン バイク | 3059.991 |
    | 772 | Mountain-100 Silver, 42 | マウンテン バイク | 3399.9900 |
    | ... | ... | ... | ... |

    これは、サーバーレス SQL プールを使用して、Spark を使用して作成されたデルタ形式ファイルに対してクエリを実行し、その結果をレポートまたは分析に使用できる方法を示しています。

6. クエリを次の SQL コードに置き換えます。

    ```sql
    USE AdventureWorks;

    SELECT * FROM Products;
    ```

7. コードを実行し、サーバーレス SQL プールを使用して、Spark メタストアが定義されているカタログ テーブル内の Delta Lake データに対してもクエリを実行できることを確認します。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースに対して **dp203-*xxxxxxx*** リソース グループ (管理対象リソース グループ以外) を選択し、そこに Synapse ワークスペース、ストレージ アカウント、ワークスペースの Spark プールが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
