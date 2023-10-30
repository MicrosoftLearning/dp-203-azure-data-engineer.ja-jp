---
lab:
  title: Azure Synapse Link for Azure Cosmos DB を使用する
  ilt-use: Lab
---

# Azure Synapse Link for Azure Cosmos DB を使用する

Azure Synapse Link for Azure Cosmos DB は、クラウドネイティブの*ハイブリッド トランザクション分析処理* (HTAP) 技術です。これを使用すると、Azure Cosmos DB に格納されているオペレーショナル データに対して Azure Synapse Analytics からほぼリアルタイムの分析を実行できます。

この演習の所要時間は約 **35** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure リソースをプロビジョニングする

Azure Synapse Link for Azure Cosmos DB を調べるには、Azure Synapse Analytics ワークスペースと Azure Cosmos DB アカウントが必要です。 この演習では、これらのリソースを Azure サブスクリプションにプロビジョニングするために、PowerShell スクリプトと ARM テンプレートを組み合わせて使用します。

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
    cd dp-203/Allfiles/labs/14
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 10 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Synapse Analytics ドキュメントの記事「[Azure Synapse Link for Azure Cosmos DB とは](https://docs.microsoft.com/azure/cosmos-db/synapse-link)」を確認してください。

## Azure Cosmos DB に Synapse Link を構成する

Synapse Link for Azure Cosmos DB を使用できるようになるには、その前に、Azure Cosmos DB アカウントでそれを有効にし、分析ストアとしてコンテナーを構成する必要があります。

### Cosmos DB アカウントで Synapse Link 機能を有効にする

1. [Azure portal](https://portal.azure.com) で、セットアップ スクリプトによって作成された **dp203-*xxxxxxx*** リソース グループを参照し、**cosmos*xxxxxxxx*** Cosmos DB アカウントを特定します。

    > **注**: 場合によっては、スクリプトで複数のリージョンに Cosmos DB アカウントを作成しようとしたために、"削除中" の状態のアカウントが 1 つ以上存在する可能性があります。** アクティブなアカウントは、名前の末尾の数字が最も大きいものです (たとえば **cosmos*xxxxxxx*3**)。

2. Azure Cosmos DB アカウントを開き、そのブレードの左側にある **[データ エクスプローラー]** ページを選択します。

    " **[ようこそ]** ダイアログ ボックスが表示された場合は、閉じます"**

3. **[データ エクスプローラー]** ページの上部にある **[Azure Synapse Link を有効にする]** ボタンを使用して、Synapse Link を有効にします。

    ![[Azure Synapse Link を有効にする] ボタンが強調表示された Cosmos DB データ エクスプローラー](./images/cosmos-enable-synapse-link.png)

4. ページの左側にある **[統合]** セクションで、 **[Azure Synapse Link]** ページを選択し、アカウントの状態が [有効] になっていることを確認します。**

### 分析ストア コンテナーを作成する

1. **[データ エクスプローラー]** ページに戻り、 **[新しいコンテナー]** ボタン (またはタイル) を使用して、次の設定で新しいコンテナーを作成します。
    - **データベース ID**: "(新規作成)" AdventureWorks**
    - **コンテナー間でスループットを共有する**: 選択<u>解除</u>
    - **コンテナー ID**: Sales
    - **パーティション キー**: /customerid
    - **コンテナーのスループット (自動スケーリング)** : 自動スケーリング
    - **コンテナーの最大 RU/s**: 4000
    - **分析ストア**: オン

    > **注**: このシナリオでは、**customerid** がパーティション キーに使用されます。これは架空のアプリケーションで顧客と販売注文の情報を取得するために多くのクエリで使用される可能性が高く、カーディナリティ (一意の値の数) が比較的高いため、顧客と販売注文の数の増加に合わせてコンテナーをスケーリングできるからです。 自動スケーリングを使用し、最大値を 4000 RU/s に設定することは、最初はクエリ ボリュームが少ない新しいアプリケーションに適しています。 最大値が 4,000 RU/秒であれば、コンテナーはこの値から、不要であればこの最大値の 10% (400 RU/秒) までの範囲で、自動的にスケーリングできます。

2. コンテナーが作成されたら、 **[データ エクスプローラー]** ページで **AdventureWorks** データベースとその **Sales** フォルダーを展開し、**Items** フォルダーを選択します。

    ![データ エクスプローラーの Adventure Works、Sales、Items フォルダー](./images/cosmos-items-folder.png)

3. **[新しい項目]** ボタンを使用して、次の JSON に基づいて新しい顧客品目を作成します。 次に、新しい項目を保存します (項目を保存すると、いくつかの追加のメタデータ フィールドが加えられます)。

    ```json
    {
        "id": "SO43701",
        "orderdate": "2019-07-01",
        "customerid": 123,
        "customerdetails": {
            "customername": "Christy Zhu",
            "customeremail": "christy12@adventure-works.com"
        },
        "product": "Mountain-100 Silver, 44",
        "quantity": 1,
        "price": 3399.99
    }
    ```

4. 次の JSON を使用して 2 つ目の項目を追加します。

    ```json
    {
        "id": "SO43704",
        "orderdate": "2019-07-01",
        "customerid": 124,
        "customerdetails": {
            "customername": "Julio Ruiz",
            "customeremail": "julio1@adventure-works.com"
        },
        "product": "Mountain-100 Black, 48",
        "quantity": 1,
        "price": 3374.99
    }
    ```

5. 次の JSON を使用して 3 つ目の項目を追加します。

    ```json
    {
        "id": "SO43707",
        "orderdate": "2019-07-02",
        "customerid": 125,
        "customerdetails": {
            "customername": "Emma Brown",
            "customeremail": "emma3@adventure-works.com"
        },
        "product": "Road-150 Red, 48",
        "quantity": 1,
        "price": 3578.27
    }
    ```

> **注**: 実際には、分析ストアには、アプリケーションによってストアに書き込まれた、はるかに大量のデータが含まれます。 この演習で原則を示すには、これらのいくつかの項目で十分です。

## Azure Synapse Analytics に Synapse Link を構成する

Azure Cosmos DB アカウントを準備し終わったので、Azure Synapse Analytics ワークスペースに Azure Synapse Link for Azure Cosmos DB を構成できます。

1. Azure portal で、Cosmos DB アカウントのブレードがまだ開いている場合は閉じ、**dp203-*xxxxxxx*** リソース グループに戻ります。
2. **synapse*xxxxxxx*** Synapse ワークスペースを開き、その **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。メッセージが表示された場合はサインインします。
3. Synapse Studio の左側にある **&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。これにより、さまざまなページが Synapse Studio 内に表示されます。
4. **[データ]** ページで、 **[リンク済み]** タブを表示します。ワークスペースには、Azure Data Lake Storage Gen2 ストレージ アカウントへのリンクが既に含まれているはずですが、Cosmos DB アカウントへのリンクは含まれていません。
5. **[+]** メニューで **[外部データに接続]** を選択し、次に **[Azure Cosmos DB for NoSQL]** を選択します。

    ![Azure Cosmos DB NoSQL API 外部データ リンクの追加](./images/add-cosmos-db-link.png)

6. 続けて、次の設定を使用して新しい Cosmos DB 接続を作成します。
    - **名前**: AdventureWorks
    - **説明**: AdventureWorks Cosmos DB データベース
    - **統合ランタイム経由で接続する**: AutoResolveIntegrationRuntime
    - **認証の種類**: アカウント キー
    - **接続文字列**: "選択済み"**
    - **アカウントの選択方法**: サブスクリプションから
    - **Azure サブスクリプション**: "使用する Azure サブスクリプションを選択します"**
    - **Azure Cosmos DB アカウント名**: "**cosmosxxxxxxx** アカウントを選択します"**
    - **データベース名**: AdventureWorks
7. 接続を作成したら、 **[データ]** ページの右上にある **&#8635;** ボタンを使用して、**Azure Cosmos DB** カテゴリが **[リンク済み]** ペインの一覧に表示されるまで表示を更新します。
8. **Azure Cosmos DB** カテゴリを展開して、作成した **AdventureWorks** 接続と、それに含まれる **Sales** コンテナーを表示します。

    ![Azure Cosmos DB SQL API 外部データ リンクの追加](./images/cosmos-linked-connection.png)

## Azure Synapse Analytics から Azure Cosmos DB に対してクエリを実行する

これで、Azure Synapse Analytics から Cosmos DB データベースに対してクエリを実行する準備ができました。

### Spark プールから Azure Cosmos DB にクエリを実行する

1. **[データ]** ペインで **Sales** コンテナーを選択し、その **[...]** メニューで **[新しいノートブック]**  >  **[データフレームに読み込む]** を選択します。
2. 新しく開いた **[Notebook 1]** タブ内の **[アタッチ先]** の一覧で、Spark プール (**spark*xxxxxxx***) を選択します。 次に、 **&#9655; [すべて実行]** ボタンを使用して、ノートブック内のすべてのセル (現時点では 1 つしかありません) を実行します。

    このセッション内で Spark コードを実行したのはこれが最初であるため、Spark プールを起動する必要があります。 これは、セッション内での最初の実行には、数分かかる場合があることを意味します。 それ以降は、短時間で実行できます。

3. Spark セッションが初期化されるのを待っている間に、生成されたコードを確認します (ツール バーの右端にある **[プロパティ]** ボタン ( **&#128463;<sub>*</sub>** のような外観) を使用して **[プロパティ]** ペインを閉じると、コードをより明確に確認できます)。 コードは次のようになります。

    ```python
    # Read from Cosmos DB analytical store into a Spark DataFrame and display 10 rows from the DataFrame
    # To select a preferred list of regions in a multi-region Cosmos DB account, add .option("spark.cosmos.preferredRegions", "<Region1>,<Region2>")

    df = spark.read\
        .format("cosmos.olap")\
        .option("spark.synapse.linkedService", "AdventureWorks")\
        .option("spark.cosmos.container", "Sales")\
        .load()

    display(df.limit(10))
    ```

4. コードの実行が完了したら、ノートブック内のセルの下にある出力を確認します。 結果には、Cosmos DB データベースに追加した項目ごとに 1 つずつ、3 つのレコードが含まれています。 各レコードには、項目の作成時に入力したフィールドと、自動的に生成された一部のメタデータ フィールドが含まれます。
5. 前のセルの結果の下で、 **[+ コード]** アイコンを使用して新しいセルをノートブックに追加し、それに次のコードを入力します。

    ```python
    customer_df = df.select("customerid", "customerdetails")
    display(customer_df)
    ```

6. それをセルの左側にある **&#9655;** アイコンを使用して実行し、結果を表示します。これは次のようになります。

    | customerid | customerdetails |
    | -- | -- |
    | 124 | "{"customername": "Julio Ruiz","customeremail": "julio1@adventure-works.com"}" |
    | 125 | "{"customername": "Emma Brown","customeremail": "emma3@adventure-works.com"}" |
    | 123 | "{"customername": "Christy Zhu","customeremail": "christy12@adventure-works.com"}" |

    このクエリにより、**customerid** および **customerdetails** 列のみを含む新しいデータフレームが作成されました。 **customerdetails** 列に、ソース項目の入れ子になったデータの JSON 構造が含まれていることを確認します。 表示された結果のテーブルで、JSON 値を横にある **&#9658;** アイコンを使用して展開し、それに含まれる個々のフィールドを表示できます。

7. 新しいセルをもう 1 つ追加し、次のコードを入力します。

    ```python
    customerdetails_df = df.select("customerid", "customerdetails.*")
    display(customerdetails_df)
    ```

8. セルを実行し、結果を確認します。**customerdetails** 値の **customername** と **customeremail** が列として含まれているはずです。

    | customerid | customername | customeremail |
    | -- | -- | -- |
    | 124 | Julio Ruiz |julio1@adventure-works.com |
    | 125 | Emma Brown |emma3@adventure-works.com |
    | 123 | Christy Zhu | christy12@adventure-works.com |

    Spark を使用すると、複雑なデータ操作コードを実行して、Cosmos DB からデータを再構築して探索できます。 この場合、PySpark 言語を使用して JSON プロパティ階層を移動し、**customerdetails** フィールドの子フィールドを取得できます。

9. 新しいセルをもう 1 つ追加し、次のコードを入力します。

    ```sql
    %%sql

    -- Create a logical database in the Spark metastore
    CREATE DATABASE salesdb;

    USE salesdb;

    -- Create a table from the Cosmos DB container
    CREATE TABLE salesorders using cosmos.olap options (
        spark.synapse.linkedService 'AdventureWorks',
        spark.cosmos.container 'Sales'
    );

    -- Query the table
    SELECT *
    FROM salesorders;
    ```

10. 新しいセルを実行して新しいデータベースを作成します。これには、Cosmos DB 分析ストアのデータが挿入されたテーブルが含まれます。
11. 別の新しいコード セルを追加し、次のコードを入力して実行します。

    ```sql
    %%sql

    SELECT id, orderdate, customerdetails.customername, product
    FROM salesorders
    ORDER BY id;
    ```

    このクエリの結果は次のようになります。

    | id | orderdate | customername | product |
    | -- | -- | -- | -- |
    | SO43701 | 2019-07-01 | Christy Zhu | Mountain-100 Silver, 44 |
    | SO43704 | 2019-07-01 | Julio Ruiz |Mountain-100 Black, 48 |
    | SO43707 | 2019-07-02 | Emma Brown |Road-150 Red, 48 |

    Spark SQL を使用したときに、JSON 構造の名前付きプロパティを列として取得できることを確認します。

12. 後で戻ってくるので、 **[ノートブック 1]** タブは開いたままにしておきます。

### サーバーレス SQL プールから Azure Cosmos DB のクエリを実行する

1. **[データ]** ペインで **Sales** コンテナーを選択し、その **[...]** メニューで **[新しい SQL スクリプト]**  >  **[上位 100 行を選択]** を選択します。
2. 開いた **[SQL スクリプト 1]** タブで、 **[プロパティ]** ペインを非表示にし、生成されたコードを表示します。これは次のようになります。

    ```sql
    IF (NOT EXISTS(SELECT * FROM sys.credentials WHERE name = 'cosmosxxxxxxxx'))
    THROW 50000, 'As a prerequisite, create a credential with Azure Cosmos DB key in SECRET option:
    CREATE CREDENTIAL [cosmosxxxxxxxx]
    WITH IDENTITY = ''SHARED ACCESS SIGNATURE'', SECRET = ''<Enter your Azure Cosmos DB key here>''', 0
    GO

    SELECT TOP 100 *
    FROM OPENROWSET(PROVIDER = 'CosmosDB',
                    CONNECTION = 'Account=cosmosxxxxxxxx;Database=AdventureWorks',
                    OBJECT = 'Sales',
                    SERVER_CREDENTIAL = 'cosmosxxxxxxxx'
    ) AS [Sales]
    ```

    SQL プールには、Cosmos DB アクセス時に使用する資格情報が必要です。これは Cosmos DB アカウントの承認キーに基づきます。 スクリプトには最初の `IF (NOT EXISTS(...` ステートメントが含まれており、この資格情報の有無を確認して、存在しない場合はエラーをスローします。

3. スクリプト内の `IF (NOT EXISTS(...` ステートメントを次のコードに置き換えて資格情報を作成します。*cosmosxxxxxxxx* は Cosmos DB アカウントの名前に置き換えます。

    ```sql
    CREATE CREDENTIAL [cosmosxxxxxxxx]
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
    SECRET = '<Enter your Azure Cosmos DB key here>'
    GO
    ```

    これで、スクリプト全体は次のようになります。

    ```sql
    CREATE CREDENTIAL [cosmosxxxxxxxx]
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
    SECRET = '<Enter your Azure Cosmos DB key here>'
    GO

    SELECT TOP 100 *
    FROM OPENROWSET(PROVIDER = 'CosmosDB',
                    CONNECTION = 'Account=cosmosxxxxxxxx;Database=AdventureWorks',
                    OBJECT = 'Sales',
                    SERVER_CREDENTIAL = 'cosmosxxxxxxxx'
    ) AS [Sales]
    ```

4. Azure portal を含むブラウザー タブに切り替えます (または、新しいタブを開き、[https://portal.azure.com](https://portal.azure.com) の Azure portal にサインインします)。 次に **dp203-*xxxxxxx*** リソース グループで、**cosmos*xxxxxxxx*** Azure Cosmos DB アカウントを開きます。
5. 左側のペインの **[設定]** セクションで、 **[キー]** ページを選択します。 次に、 **[主キー]** の値をクリップボードにコピーします。
6. Azure Synapse Studio で SQL スクリプトを含むブラウザー タブに戻り、プレースホルダー ***\<Enter your Azure Cosmos DB key here\>*** を置き換えてコードにキーを貼り付けます。スクリプトは次のようになります。

    ```sql
    CREATE CREDENTIAL [cosmosxxxxxxxx]
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
    SECRET = '1a2b3c....................................=='
    GO

    SELECT TOP 100 *
    FROM OPENROWSET(PROVIDER = 'CosmosDB',
                    CONNECTION = 'Account=cosmosxxxxxxxx;Database=AdventureWorks',
                    OBJECT = 'Sales',
                    SERVER_CREDENTIAL = 'cosmosxxxxxxxx'
    ) AS [Sales]
    ```

7. **[&#9655; 実行]** ボタンを使用してスクリプトを実行し、結果を確認します。これには、Cosmos DB データベースに追加した項目ごとに 1 つずつ、3 つのレコードが含まれています。

    これで資格情報が作成されたので、Cosmos DB データ ソースに対するクエリでそれを使用できます。

8. スクリプト内のすべてのコード (CREATE CREDENTIAL と SELECT の両方のステートメント) を次のコードに置き換えます (*cosmosxxxxxxxx* は Azure Cosmos DB アカウントの名前に置き換えます)。

    ```sql
    SELECT *
    FROM OPENROWSET(PROVIDER = 'CosmosDB',
                    CONNECTION = 'Account=cosmosxxxxxxxx;Database=AdventureWorks',
                    OBJECT = 'Sales',
                    SERVER_CREDENTIAL = 'cosmosxxxxxxxx'
    )
    WITH (
        OrderID VARCHAR(10) '$.id',
        OrderDate VARCHAR(10) '$.orderdate',
        CustomerID INTEGER '$.customerid',
        CustomerName VARCHAR(40) '$.customerdetails.customername',
        CustomerEmail VARCHAR(30) '$.customerdetails.customeremail',
        Product VARCHAR(30) '$.product',
        Quantity INTEGER '$.quantity',
        Price FLOAT '$.price'
    )
    AS sales
    ORDER BY OrderID;
    ```

9. スクリプトを実行し、結果を確認します。`WITH` 句で定義されているスキーマと一致するはずです。

    | OrderID | OrderDate | CustomerID | CustomerName | CustomerEmail | 製品 | Quantity | Price |
    | -- | -- | -- | -- | -- | -- | -- | -- |
    | SO43701 | 2019-07-01 | 123 | Christy Zhu | christy12@adventure-works.com | Mountain-100 Silver, 44 | 1 | 3399.99 |
    | SO43704 | 2019-07-01 | 124 | Julio Ruiz | julio1@adventure-works.com | Mountain-100 Black, 48 | 1 | 3374.99 |
    | SO43707 | 2019-07-02 | 125 | Emma Brown | emma3@adventure-works.com | Road-150 Red, 48 | 1 | 3578.27 |

10. 後で戻って来るので、 **[SQL スクリプト 1]** タブは開いたままにしておきます。

### Cosmos DB でのデータ変更が Synapse に反映されていることを確認する 

1. Synapse Studio を含むブラウザー タブを開いたままにして、Azure portal を含むタブに戻ります。そのタブには、Cosmos DB アカウントの **[キー]** ページが開いているはずです。
2. **[データ エクスプローラー]** ページで **AdventureWorks** データベースとその **Sales** フォルダーを展開し、**Items** フォルダーを選択します。
3. **[新しい項目]** ボタンを使用して、次の JSON に基づいて新しい顧客品目を作成します。 次に、新しい項目を保存します (項目を保存すると、いくつかの追加のメタデータ フィールドが加えられます)。

    ```json
    {
        "id": "SO43708",
        "orderdate": "2019-07-02",
        "customerid": 126,
        "customerdetails": {
            "customername": "Samir Nadoy",
            "customeremail": "samir1@adventure-works.com"
        },
        "product": "Road-150 Black, 48",
        "quantity": 1,
        "price": 3578.27
    }
    ```

4. [Synapse Studio] タブに戻り、 **[SQL スクリプト 1]** タブでクエリを再実行します。 最初は前と同じ結果が表示されることがありますが、2019-07-02 の Samir Nadoy への販売が結果に表示されるまで、1 分ほど待ってからもう一度クエリを再実行します。
5. **[ノートブック 1]** タブに戻り、Spark ノートブックの最後のセルを再実行して、Samir Nadoy への販売がクエリ結果に含まれていることを確認します。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースに対して **dp203-*xxxxxxx*** リソース グループ (管理対象リソース グループではなく) を選択し、そこに Synapse ワークスペース、ストレージ アカウント、ワークスペースおよび Azure Cosmos DB の Spark プールが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
