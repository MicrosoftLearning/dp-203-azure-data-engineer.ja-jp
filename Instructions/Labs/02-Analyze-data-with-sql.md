---
lab:
  title: サーバーレス SQL プールを使用してファイルのクエリを実行する
  ilt-use: Lab
---

# サーバーレス SQL プールを使用してファイルのクエリを実行する

SQL はおそらく、データを操作するために世界中で最も使用されている言語です。 ほとんどのデータ アナリストは、SQL クエリを使用してリレーショナル データベースで最も一般的な操作 (データの取得、フィルター処理、集計) を行うことに習熟しています。 組織でデータ レイクの作成にスケーラブルなファイル ストレージが利用される場合が増えており、多くの場合、データのクエリには依然として SQL が優先的に選択されています。 Azure Synapse Analytics にはサーバーレス SQL プールが用意されており、これを使用して、SQL クエリ エンジンをデータ ストレージから切り離したり、区切りテキストや Parquet などの一般的なファイル形式のデータ ファイルに対してクエリを実行したりすることができます。

このラボは完了するまで、約 **40** 分かかります。

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

4. PowerShell ペインで、次のコマンドを手動で入力して、このリポジトリを複製します。

    ```
    rm -r dp203 -f
    git clone https://github.com/MicrosoftLearning/dp-203-azure-data-engineer dp203
    ```

5. リポジトリが複製されたら、次のコマンドを入力してこのラボ用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```
    cd dp203/Allfiles/labs/02
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 10 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Synapse Analytics ドキュメントの記事「[Azure Synapse Analytics のサーバーレス SQL プール](https://docs.microsoft.com/azure/synapse-analytics/sql/on-demand-workspace-overview)」を確認してください。

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
8. いずれかのファイルを右クリックし、 **[プレビュー]** を選択して、そこに含まれているデータを表示します。 ファイルにはヘッダー行が含まれていないため、列ヘッダーを表示するには、オプションの選択を解除することに注意してください。
9. プレビューを閉じ、 **&#8593;** ボタンを使用して、**sales** フォルダーに戻ります。
10. **sales** フォルダーで **json** フォルダーを開き、.json ファイルに何件かのサンプルの販売注文が含まれていることを確認します。 これらのファイルのいずれかをプレビューして、販売注文に使用されている JSON 形式を確認します。
11. プレビューを閉じ、 **&#8593;** ボタンを使用して、**sales** フォルダーに戻ります。
12. **sales** フォルダーで **parquet** フォルダーを開き、各年 (2019 年から 2021 年) のサブフォルダーが含まれており、各フォルダー内の **orders.snappy.parquet** という名前のファイルに、その年の注文データが含まれていることを確認します。 
13. **sales** フォルダーに戻ると、**csv**、**json**、**parquet** の各フォルダーが表示されます。

### SQL を使用して CSV ファイルのクエリを実行する

1. **csv** フォルダーを選択し、ツール バーの **[新しい SQL スクリプト]** の一覧で、 **[上位 100 行を選択]** を選択します。
2. **[ファイルの種類]** の一覧で、 **[テキスト形式]** を選択し、設定を適用して、フォルダー内のデータのクエリを実行する新しい SQL スクリプトを開きます。
3. 作成された **SQL Script 1** の **[プロパティ]** ペインで、名前を **Sales CSV query** に変更し、結果の設定を **[すべての行]** に変更します。 ツール バーで **[発行]** を選択してスクリプトを保存し、ツール バーの右端にある **[プロパティ]** ボタン ( **&#128463;** のような外観) を使用して、 **[プロパティ]** ペインを非表示にします。
4. 生成された SQL コードを確認します。これは次のようになります。

    ```SQL
    -- This is auto-generated code
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/csv/',
            FORMAT = 'CSV',
            PARSER_VERSION='2.0'
        ) AS [result]
    ```

    このコードでは、OPENROWSET を使用して、sales フォルダー内の CSV ファイルからデータを読み取り、最初の 100 行のデータを取得します。

5. **[接続先]** ボックスの一覧で、**[組み込み]** が選択されていることを確認します。これは、ワークスペースで作成された組み込みの SQL プールを表します。
6. ツールバーの **[&#9655; 実行]** ボタンを使用して SQL コードを実行し、結果を確認します。これは次のようになります。

    | C1 | C2 | C3 | C4 | C5 | C6 | C7 | C8 | C9 |
    | -- | -- | -- | -- | -- | -- | -- | -- | -- |
    | SO45347 | 1 | 2020-01-01 | Clarence Raji | clarence35@adventure-works.com |Road-650 黒、52 | 1 | 699.0982 | 55.9279 |
    | ... | ... | ... | ... | ... | ... | ... | ... | ... |

7. 結果は、C1、C2 などの名前の列で構成されていることに注意してください。 この例の CSV ファイルには、列ヘッダーは含まれていません。 割り当てられた汎用の列名を使用して、または序数位置でデータを操作できますが、表形式のスキーマを定義すると、データを理解しやすくなります。 これを実現するには、次に示すように OPENROWSET 関数に WITH 句を追加して (*datalakexxxxxxx* を Data Lake Storage アカウントの名前に置き換えます)、クエリを再実行します。

    ```SQL
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/csv/',
            FORMAT = 'CSV',
            PARSER_VERSION='2.0'
        )
        WITH (
            SalesOrderNumber VARCHAR(10) COLLATE Latin1_General_100_BIN2_UTF8,
            SalesOrderLineNumber INT,
            OrderDate DATE,
            CustomerName VARCHAR(25) COLLATE Latin1_General_100_BIN2_UTF8,
            EmailAddress VARCHAR(50) COLLATE Latin1_General_100_BIN2_UTF8,
            Item VARCHAR(30) COLLATE Latin1_General_100_BIN2_UTF8,
            Quantity INT,
            UnitPrice DECIMAL(18,2),
            TaxAmount DECIMAL (18,2)
        ) AS [result]
    ```

    結果は次のようになります。

    | SalesOrderNumber | SalesOrderLineNumber | OrderDate | CustomerName | EmailAddress | Item | Quantity | UnitPrice | TaxAmount |
    | -- | -- | -- | -- | -- | -- | -- | -- | -- |
    | SO45347 | 1 | 2020-01-01 | Clarence Raji | clarence35@adventure-works.com |Road-650 黒、52 | 1 | 699.10 | 55.93 |
    | ... | ... | ... | ... | ... | ... | ... | ... | ... |

8. 変更をスクリプトに発行し、スクリプト ペインを閉じます。

### SQL を使用して Parquet ファイルのクエリを実行する

CSV は使いやすい形式ですが、ビッグ データ処理のシナリオでは、圧縮、インデックス作成、パーティション分割用に最適化されたファイル形式を使用するのが一般的です。 このような形式の中で最も一般的なものの 1 つは、*Parquet* です。

1. データ レイクのファイル システムが含まれている **[ファイル]** タブで、**sales** フォルダーに戻ると、**csv**、**json**、**parquet** の各フォルダーが表示されます。
2. **parquet** フォルダーを選択し、ツール バーの **[新しい SQL スクリプト]** の一覧で、 **[上位 100 行を選択]** を選択します。
3. **[ファイルの種類]** の一覧で、 **[Parquet 形式]** を選択し、設定を適用して、フォルダー内のデータのクエリを実行する新しい SQL スクリプトを開きます。 スクリプトは次のようになります。

    ```SQL
    -- This is auto-generated code
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/parquet/**',
            FORMAT = 'PARQUET'
        ) AS [result]
    ```

4. コードを実行します。先ほど調べた CSV ファイルと同じスキーマで販売注文データが返されることに注意してください。 スキーマ情報は Parquet ファイルに埋め込まれているため、結果には適切な列名が表示されます。
5. コードを次のように変更して (*datalakexxxxxxx* を Data Lake Storage アカウントの名前に置き換えます)、クエリを実行します。

    ```sql
    SELECT YEAR(OrderDate) AS OrderYear,
           COUNT(*) AS OrderedItems
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/parquet/**',
            FORMAT = 'PARQUET'
        ) AS [result]
    GROUP BY YEAR(OrderDate)
    ORDER BY OrderYear
    ```

6. 結果には、3 年間すべての注文件数が含まれます。BULK パスでワイルドカードを使用すると、クエリによってすべてのサブフォルダーのデータが返されます。

    サブフォルダーには、Parquet データの "パーティション" が反映されます。これは、データの複数のパーティションを並行して処理できるシステムのパフォーマンスを最適化するためによく使用される手法です。** また、パーティションを使用してデータをフィルター処理することもできます。

7. コードを次のように変更して (*datalakexxxxxxx* を Data Lake Storage アカウントの名前に置き換えます)、クエリを実行します。

    ```sql
    SELECT YEAR(OrderDate) AS OrderYear,
           COUNT(*) AS OrderedItems
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/parquet/year=*/',
            FORMAT = 'PARQUET'
        ) AS [result]
    WHERE [result].filepath(1) IN ('2019', '2020')
    GROUP BY YEAR(OrderDate)
    ORDER BY OrderYear
    ```

8. 結果を確認し、2019 念と 2020 年の販売件数のみが含まれていることに注意してください。 このフィルター処理は、BULK パスのパーティション フォルダー値のワイルドカード (*year=\** ) と、OPENROWSET によって返される結果の *filepath* プロパティに基づく WHERE 句 (この場合はエイリアス *[result]* ) を含めることによって実現されます。

9. スクリプトに **Sales Parquet query** という名前を付けて、発行します。 スクリプト ペインを閉じます。

### SQL を使用して JSON ファイルのクエリを実行する

JSON は、もう 1 つのよく使用されるデータ形式であるため、サーバーレス SQL プール内の .json ファイルに対してクエリを実行できると便利です。

1. データ レイクのファイル システムが含まれている **[ファイル]** タブで、**sales** フォルダーに戻ると、**csv**、**json**、**parquet** の各フォルダーが表示されます。
2. **json** フォルダーを選択し、ツール バーの **[新しい SQL スクリプト]** の一覧で、 **[上位 100 行を選択]** を選択します。
3. **[ファイルの種類]** の一覧で、 **[テキスト形式]** を選択し、設定を適用して、フォルダー内のデータのクエリを実行する新しい SQL スクリプトを開きます。 スクリプトは次のようになります。

    ```sql
    -- This is auto-generated code
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/json/',
            FORMAT = 'CSV',
            PARSER_VERSION = '2.0'
        ) AS [result]
    ```

    このスクリプトは、JSON ではなくコマンド区切り (CSV) データに対してクエリを実行するように設計されているため、正常に動作するには、いくつかの変更を行う必要があります。

4. スクリプトを次のように変更します (*datalakexxxxxxx* を Data Lake Storage アカウントの名前に置き換えます)。
    - パーサーのバージョンのパラメーターを削除します。
    - フィールド ターミネータ、引用符文字、行ターミネータのパラメーターを文字コード *0x0b* に設定して追加します。
    - 結果の書式を、データの JSON 行を NVARCHAR(MAX) 文字列として含む単一フィールドとして設定します。

    ```sql
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/json/',
            FORMAT = 'CSV',
            FIELDTERMINATOR ='0x0b',
            FIELDQUOTE = '0x0b',
            ROWTERMINATOR = '0x0b'
        ) WITH (Doc NVARCHAR(MAX)) as rows
    ```

5. 変更したコードを実行し、結果に各注文の JSON ドキュメントが含まれていることを確認します。

6. JSON_VALUE 関数を使用して JSON データから個々のフィールド値を抽出するようにクエリを次のように変更します (*datalakexxxxxxx* を Data Lake Storage アカウントの名前に置き換えます)。

    ```sql
    SELECT JSON_VALUE(Doc, '$.SalesOrderNumber') AS OrderNumber,
           JSON_VALUE(Doc, '$.CustomerName') AS Customer,
           Doc
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/json/',
            FORMAT = 'CSV',
            FIELDTERMINATOR ='0x0b',
            FIELDQUOTE = '0x0b',
            ROWTERMINATOR = '0x0b'
        ) WITH (Doc NVARCHAR(MAX)) as rows
    ```

7. スクリプトに **Sales JSON query**という名前を付けて発行します。 スクリプト ペインを閉じます。

## データベース内の外部データにアクセスする

ここまでは、SELECT クエリで OPENROWSET 関数を使用して、データ レイク内のファイルからデータを取得しました。 クエリは、サーバーレス SQL プール内の **master** データベースのコンテキストで実行されました。 この方法は、データを最初に調べるときには適していますが、より複雑なクエリを作成する場合は、Synapse SQL の *PolyBase* 機能を使用して、外部データの場所を参照するオブジェクトをデータベース内に作成する方が効果的な場合があります。

### 外部データ ソースを作成する

データベースに外部データ ソースを定義すると、それを使用して、ファイルが格納されているデータ レイクの場所を参照できます。

1. Synapse Studio の **[開発]** ページの **[+]** メニューで、 **[SQL スクリプト]** を選択します。
2. 新しいスクリプト ペインで、次のコードを追加して (*datalakexxxxxxx* を Data Lake Storage アカウントの名前に置き換えます)、新しいデータベースを作成し、それに外部データ ソースを追加します。

    ```sql
    CREATE DATABASE Sales
      COLLATE Latin1_General_100_BIN2_UTF8;
    GO;

    Use Sales;
    GO;

    CREATE EXTERNAL DATA SOURCE sales_data WITH (
        LOCATION = 'https://datalakexxxxxxx.dfs.core.windows.net/files/sales/'
    );
    GO;
    ```

3. スクリプトのプロパティを変更して名前を **Create Sales DB**に変更し、発行します。
4. スクリプトが **Built-in** SQL プールと **master** データベースに接続されていることを確認し、スクリプトを実行します。
5. **[データ]** ページに戻り、Synapse Studio の右上にある **&#8635;** ボタンを使用してページを更新します。 **[データ]** ペインの **[ワークスペース]** タブを表示します。ここに、 **[SQL データベース]** の一覧が表示されるようになりました。 この一覧を展開して、**Sales** データベースが作成されたことを確認します。
6. **Sales** データベース、その **External Resources** フォルダー、その下の **External data sources** フォルダーを順に展開すると、作成した **sales_data** 外部データ ソースが表示されます。
7. **Sales** データベースの **[...]** メニューで、 **[新しい SQL スクリプト]**  >  **[空のスクリプト]** の順に選択します。 次に、新しいスクリプト ペインで次のクエリを入力し、実行します。

    ```sql
    SELECT *
    FROM
        OPENROWSET(
            BULK 'csv/*.csv',
            DATA_SOURCE = 'sales_data',
            FORMAT = 'CSV',
            PARSER_VERSION = '2.0'
        ) AS orders
    ```

    クエリでは外部データ ソースを使用してデータ レイクに接続するため、OPENROWSET 関数では、.csv ファイルへの相対パスのみを参照するだけで済むようになりました。

8. データ ソースを使用して Parquet ファイルのクエリを実行するには、コードを次のように変更します。

    ```sql
    SELECT *
    FROM  
        OPENROWSET(
            BULK 'parquet/year=*/*.snappy.parquet',
            DATA_SOURCE = 'sales_data',
            FORMAT='PARQUET'
        ) AS orders
    WHERE orders.filepath(1) = '2019'
    ```

### 外部テーブルを作成する

外部データ ソースを使用すると、データ レイク内のファイルに簡単にアクセスできますが、SQL を使用しているほとんどのデータ アナリストは、データベース内のテーブルの操作に慣れています。 幸い、データ テーブル内のファイルから行セットをカプセル化する外部ファイル形式と外部テーブルを定義することもできます。

1. SQL コードを次のステートメントに置き換えて、CSV ファイルの外部データ形式と、CSV ファイルを参照する外部テーブルを定義し、コードを実行します。

    ```sql
    CREATE EXTERNAL FILE FORMAT CsvFormat
        WITH (
            FORMAT_TYPE = DELIMITEDTEXT,
            FORMAT_OPTIONS(
            FIELD_TERMINATOR = ',',
            STRING_DELIMITER = '"'
            )
        );
    GO;

    CREATE EXTERNAL TABLE dbo.orders
    (
        SalesOrderNumber VARCHAR(10),
        SalesOrderLineNumber INT,
        OrderDate DATE,
        CustomerName VARCHAR(25),
        EmailAddress VARCHAR(50),
        Item VARCHAR(30),
        Quantity INT,
        UnitPrice DECIMAL(18,2),
        TaxAmount DECIMAL (18,2)
    )
    WITH
    (
        DATA_SOURCE =sales_data,
        LOCATION = 'csv/*.csv',
        FILE_FORMAT = CsvFormat
    );
    GO
    ```

2. **[データ]** ペインで **External tables** フォルダーを更新して展開し、**Sales** データベースに **dbo.orders** という名前のテーブルが作成されていることを確認します。
3. **dbo.orders** テーブルの **[...]** メニューで、 **[新しい SQL スクリプト]**  >  **[上位 100 行を選択]** の順に選択します。
4. 生成された SELECT スクリプトを実行し、データの最初の 100 行をテーブルから取得し、データ レイク内のファイルを参照していることを確認します。

    >**注:** 特定のニーズとユース ケースに最適な方法を常に選択する必要があります。 詳細については、「[Azure Synapse Analytics でサーバーレス SQL プールを使う際の OPENROWSET の使用方法](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/develop-openrowset)」と「[Azure Synapse Analytics のサーバーレス SQL プールを使用して外部ストレージにアクセスする](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/develop-storage-files-overview?tabs=impersonation)」の記事をご確認ください。

## クエリ結果を視覚化する

SQL クエリを使用してデータ レイク内のファイルに対してクエリを実行するためのさまざまな方法を調べたので、これらのクエリの結果を分析してデータに関する分析情報を得ることができます。 多くの場合、クエリ結果をグラフで視覚化すると、分析情報を見つけやすくなります。視覚化は、Synapse Studio クエリ エディターに統合されたグラフ作成機能を使用することで簡単に行うことができます。

1. **[開発]** ページで、新しい空の SQL クエリを作成します。
2. スクリプトが **Built-in** SQL プールと **Sales** データベースに接続されていることを確認します。
3. 次の SQL コードを入力して実行します。

    ```sql
    SELECT YEAR(OrderDate) AS OrderYear,
           SUM((UnitPrice * Quantity) + TaxAmount) AS GrossRevenue
    FROM dbo.orders
    GROUP BY YEAR(OrderDate)
    ORDER BY OrderYear;
    ```

4. **[結果]** ペインで、 **[グラフ]** を選択すると、グラフが自動的に作成されて表示されます。これは折れ線グラフになります。
5. 折れ線グラフに 2019 年から 2021 年までの 3 年間の収益傾向が表示されるように、 **[カテゴリ列]** を **OrderYear** に変更します。

    ![年別の収益を示す折れ線グラフ](./images/yearly-sales-line.png)

6. **[グラフの種類]** を **[縦棒]** に切り替えて、年間収益を縦棒グラフとして表示します。

    ![年別の収益を示す縦棒グラフ](./images/yearly-sales-column.png)

7. クエリ エディターでグラフ作成機能を試してみましょう。 データを対話形式で調べるときに使用できる基本的なグラフ作成機能がいくつか用意されており、グラフを画像として保存し、レポートに含めることができます。 ただし、Microsoft Power BI などのエンタープライズ データ視覚化ツールと比べると、機能は限られます。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースの **dp203-xxxxxxx** リソース グループ (管理対象リソース グループではありません) を選択し、Synapse ワークスペースとお使いのワークスペースのストレージ アカウントが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。