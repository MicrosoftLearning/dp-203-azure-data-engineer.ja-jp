---
lab:
  title: リレーショナル データ ウェアハウスにデータを読み込む
  ilt-use: Lab
---

# リレーショナル データ ウェアハウスにデータを読み込む

この演習では、専用 SQL プールにデータを読み込みます。

この演習の所要時間や約 **30** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Synapse Analytics ワークスペースをプロビジョニングする

データ レイク ストレージにアクセスできる Azure Synapse Analytics ワークスペースと、データ ウェアハウスをホストする専用 SQL プールが必要です。

この演習では、Azure Synapse Analytics ワークスペースをプロビジョニングするために、PowerShell スクリプトと ARM テンプレートを組み合わせて使用します。

1. [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示された場合は、***PowerShell*** 環境を選択して、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更でき、ペインの右上にある **&#9723;** アイコンと **X** アイコンを使用してペインを最小化または最大化、閉じることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、このリポジトリを複製します。

    ```powershell
    rm -r dp-203 -f
    git clone https://github.com/MicrosoftLearning/dp-203-azure-data-engineer dp-203
    ```

5. リポジトリが複製されたら、次のコマンドを入力してこの演習用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```powershell
    cd dp-203/Allfiles/labs/09
    ./setup.ps1
    ```

6. メッセージが表示されたら、使用するサブスクリプションを選択します (このオプションは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトが完了するまで待ちます。通常、約 10 分かかりますが、場合によってはそれ以上かかることもあります。 待っている間に、Azure Synapse Analytics ドキュメントの記事「[Azure Synapse Analytics の専用 SQL プールのデータ読み込み戦略](https://learn.microsoft.com/azure/synapse-analytics/sql-data-warehouse/design-elt-data-loading)」を確認してください。

## データを読み込む準備をする

1. スクリプトが完了したら、Azure portal で、作成された **dp203-*xxxxxxx*** リソース グループに移動して、Synapse ワークスペースを選択します。
2. Synapse ワークスペースの **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。メッセージが表示された場合はサインインします。
3. Synapse Studio の左側にある ›› アイコンを使用してメニューを展開すると、リソースの管理とデータ分析タスクの実行に使用するさまざまなページが Synapse Studio 内に表示されます。
4. **[管理]** ページの **[SQL プール]** タブで、この演習用にデータ ウェアハウスをホストする **sql*xxxxxxx*** 専用 SQL プールの行を選択し、その **&#9655;** アイコンを使用して起動します。メッセージが表示されたら、再開することを確認します。

    プールの再開には数分かかる場合があります。 **&#8635; [更新]** ボタンを使用して、その状態を定期的に確認できます。 準備ができると、状態が **[オンライン]** と表示されます。 待っている間に、次の手順に進み、読み込むデータ ファイルを表示します。

5. **[データ]** ページで **[リンク]** タブを表示して、**synapsexxxxxxx (Primary - datalakexxxxxxx)** のような名前の Azure Data Lake Storage Gen2 ストレージ アカウントへのリンクがワークスペースに含まれていることを確認します。
6. ストレージ アカウントを展開して、そこに **files (primary)** という名前のファイル システム コンテナーが含まれていることを確認します。
7. このファイル コンテナーを選択し、そこに **data** という名前のフォルダーが含まれていることに注目します。 このフォルダーに、データ ウェアハウスに読み込むデータ ファイルが含まれています。
8. **data** フォルダーを開き、顧客データと製品データの .csv ファイルが含まれていることを確認します。
9. いずれかのファイルを右クリックし、 **[プレビュー]** を選択して、そこに含まれているデータを表示します。 このファイルにはヘッダー行が含まれているため、列ヘッダーを表示するには、オプションを選択できることに注意してください。
10. **[管理]** ページに戻り、専用 SQL プールがオンラインであることを確認します。

## データ ウェアハウスのテーブルを読み込む

データ ウェアハウスにデータを読み込む SQL ベースのアプローチをいくつか見てみましょう。

1. **[データ]** ページで、 **[ワークスペース]** タブを選択します。
2. **[SQL Database]** を展開し、**sql*xxxxxxx*** データベースを選択します。 次に、その **[...]** メニューの **[新しい SQL スクリプト]**  > 
 **[空のスクリプト]** を選択します。

これで、空の SQL ページが作成されました。このページは、次の演習のためにインスタンスに接続されています。 このスクリプトを使用して、データの読み込みに使用できるいくつかの SQL 手法について説明します。

### COPY ステートメントを使用してデータ レイクからデータを読み込む

1. SQL スクリプトで、ウィンドウに次のコードを入力します。

    ```sql
    SELECT COUNT(1) 
    FROM dbo.StageProduct
    ```

2. ツール バーの **&#9655; [実行]** ボタンを使用して SQL コードを実行し、**StageProduct** テーブルの行数が現在 **0** であることを確認します。
3. コードを次の COPY ステートメントに置き換えます (**datalake*xxxxxx*** をデータ レイクの名前に変更します)。

    ```sql
    COPY INTO dbo.StageProduct
        (ProductID, ProductName, ProductCategory, Color, Size, ListPrice, Discontinued)
    FROM 'https://datalakexxxxxx.blob.core.windows.net/files/data/Product.csv'
    WITH
    (
        FILE_TYPE = 'CSV',
        MAXERRORS = 0,
        IDENTITY_INSERT = 'OFF',
        FIRSTROW = 2 --Skip header row
    );


    SELECT COUNT(1) 
    FROM dbo.StageProduct
    ```

4. スクリプトを実行して結果を確認します。 **StageProduct** テーブルに 11 行読み込まれているはずです。

    次に、同じ手法を使用して別のテーブルを読み込み、今度は発生する可能性のあるエラーをログに記録してみましょう。

5. スクリプト ペインの SQL コードを次のコードに置き換え、```FROM``` 句と ```ERRORFILE``` 句の両方で **datalake*xxxxxx*** をデータ レイクの名前に変更します。

    ```sql
    COPY INTO dbo.StageCustomer
    (GeographyKey, CustomerAlternateKey, Title, FirstName, MiddleName, LastName, NameStyle, BirthDate, 
    MaritalStatus, Suffix, Gender, EmailAddress, YearlyIncome, TotalChildren, NumberChildrenAtHome, EnglishEducation, 
    SpanishEducation, FrenchEducation, EnglishOccupation, SpanishOccupation, FrenchOccupation, HouseOwnerFlag, 
    NumberCarsOwned, AddressLine1, AddressLine2, Phone, DateFirstPurchase, CommuteDistance)
    FROM 'https://datalakexxxxxx.dfs.core.windows.net/files/data/Customer.csv'
    WITH
    (
    FILE_TYPE = 'CSV'
    ,MAXERRORS = 5
    ,FIRSTROW = 2 -- skip header row
    ,ERRORFILE = 'https://datalakexxxxxx.dfs.core.windows.net/files/'
    );
    ```

6. スクリプトを実行して結果のメッセージを確認します。 ソース ファイルに無効なデータを含む行が含まれているため、1 行拒否されます。 上記のコードでは最大 **5** つのエラーが指定されているため、1 つのエラーで有効な行の読み込みが妨げられていることはありません。 次のクエリを実行して、読み込み "済み" の行を表示できます。**

    ```sql
    SELECT *
    FROM dbo.StageCustomer
    ```

7. **[ファイル]** タブで、データ レイクのルート フォルダーを表示し、 **_rejectedrows** という名前の新しいフォルダーが作成されていることを確認します (このフォルダーが表示されない場合は、 **[その他]** メニューの **[更新]** を選択してビューを更新します)。
8. **_rejectedrows** フォルダーとそのフォルダーに含まれる日付と時刻の特定のサブフォルダーを開き、***QID123_1_2*.Error.Txt** や ***QID123_1_2*.Row.Txt** のような名前のファイルが作成されていることに注目します。 これらの各ファイルを右クリックし、 **[プレビュー]** を選択すると、エラーの詳細と拒否された行が表示されます。

    ステージング テーブルを使用すると、データを移動または使用して既存のディメンション テーブルに追加またはアップサートする前に、データを検証または変換できます。 COPY ステートメントから提供されるシンプルでありながらパフォーマンスの高い手法を使用して、データ レイク内のファイルからステージング テーブルにデータを簡単に読み込み、前述のように、無効な行を識別してリダイレクトできます。

### CREATE TABLE AS (CTAS) ステートメントを使用する

1. スクリプト ペインに戻り、そこに含まれているコードを次のコードに置き換えます。

    ```sql
    CREATE TABLE dbo.DimProduct
    WITH
    (
        DISTRIBUTION = HASH(ProductAltKey),
        CLUSTERED COLUMNSTORE INDEX
    )
    AS
    SELECT ROW_NUMBER() OVER(ORDER BY ProductID) AS ProductKey,
        ProductID AS ProductAltKey,
        ProductName,
        ProductCategory,
        Color,
        Size,
        ListPrice,
        Discontinued
    FROM dbo.StageProduct;
    ```

2. スクリプトを実行すると、**ProductAltKey** をそのハッシュ分散キーとして使用し、クラスター化された列ストア インデックスを持つ、ステージングされた製品データから **DimProduct** という名前の新しいテーブルが作成されます。
4. 次のクエリを使用して、新しい **DimProduct** テーブルの内容を表示します。

    ```sql
    SELECT ProductKey,
        ProductAltKey,
        ProductName,
        ProductCategory,
        Color,
        Size,
        ListPrice,
        Discontinued
    FROM dbo.DimProduct;
    ```

    CREATE TABLE AS SELECT (CTAS) 式には、次のようなさまざまな用途があります。

    - クエリ パフォーマンスを向上させるために、他のテーブルと一致するようにテーブルのハッシュ キーを再配布する。
    - 差分分析を実行した後、既存の値に基づいてステージング テーブルに代理キーを割り当てる。
    - レポート用に集計テーブルをすばやく作成する。

### INSERT ステートメントと UPDATE ステートメントを組み合わせて、緩やかに変化するディメンション テーブルを読み込む

**DimCustomer** テーブルでは、型 1 と型 2 の緩やかに変化するディメンション (SCD) がサポートされています。ここで、型 1 の変更によって既存の行がインプレース更新され、型 2 の変更によって、特定のディメンション エンティティ インスタンスの最新バージョンを示す新しい行が生成されます。 このテーブルを読み込むには、INSERT ステートメント (新しい顧客を読み込む) と UPDATE ステートメント (型 1 または型 2 の変更を適用する) の組み合わせが必要です。

1. クエリ ペインで、既存の SQL コードを次のコードに書き換えます。

    ```sql
    INSERT INTO dbo.DimCustomer ([GeographyKey],[CustomerAlternateKey],[Title],[FirstName],[MiddleName],[LastName],[NameStyle],[BirthDate],[MaritalStatus],
    [Suffix],[Gender],[EmailAddress],[YearlyIncome],[TotalChildren],[NumberChildrenAtHome],[EnglishEducation],[SpanishEducation],[FrenchEducation],
    [EnglishOccupation],[SpanishOccupation],[FrenchOccupation],[HouseOwnerFlag],[NumberCarsOwned],[AddressLine1],[AddressLine2],[Phone],
    [DateFirstPurchase],[CommuteDistance])
    SELECT *
    FROM dbo.StageCustomer AS stg
    WHERE NOT EXISTS
        (SELECT * FROM dbo.DimCustomer AS dim
        WHERE dim.CustomerAlternateKey = stg.CustomerAlternateKey);

    -- Type 1 updates (change name, email, or phone in place)
    UPDATE dbo.DimCustomer
    SET LastName = stg.LastName,
        EmailAddress = stg.EmailAddress,
        Phone = stg.Phone
    FROM DimCustomer dim inner join StageCustomer stg
    ON dim.CustomerAlternateKey = stg.CustomerAlternateKey
    WHERE dim.LastName <> stg.LastName OR dim.EmailAddress <> stg.EmailAddress OR dim.Phone <> stg.Phone

    -- Type 2 updates (address changes triggers new entry)
    INSERT INTO dbo.DimCustomer
    SELECT stg.GeographyKey,stg.CustomerAlternateKey,stg.Title,stg.FirstName,stg.MiddleName,stg.LastName,stg.NameStyle,stg.BirthDate,stg.MaritalStatus,
    stg.Suffix,stg.Gender,stg.EmailAddress,stg.YearlyIncome,stg.TotalChildren,stg.NumberChildrenAtHome,stg.EnglishEducation,stg.SpanishEducation,stg.FrenchEducation,
    stg.EnglishOccupation,stg.SpanishOccupation,stg.FrenchOccupation,stg.HouseOwnerFlag,stg.NumberCarsOwned,stg.AddressLine1,stg.AddressLine2,stg.Phone,
    stg.DateFirstPurchase,stg.CommuteDistance
    FROM dbo.StageCustomer AS stg
    JOIN dbo.DimCustomer AS dim
    ON stg.CustomerAlternateKey = dim.CustomerAlternateKey
    AND stg.AddressLine1 <> dim.AddressLine1;
    ```

2. スクリプトを実行し、出力を確認します。

## 読み込み後の最適化を実行する

データ ウェアハウスに新しいデータを読み込んだ後は、テーブルのインデックスを再構築し、クエリが実行されることが多い列の統計を更新することをお勧めします。

1. スクリプト ペイン内のコードを次のコードで置き換えます。

    ```sql
    ALTER INDEX ALL ON dbo.DimProduct REBUILD;
    ```

2. スクリプトを実行して、**DimProduct** テーブルのインデックスを再構築します。
3. スクリプト ペイン内のコードを次のコードで置き換えます。

    ```sql
    CREATE STATISTICS customergeo_stats
    ON dbo.DimCustomer (GeographyKey);
    ```

4. スクリプトを実行して、**DimCustomer** テーブルの **GeographyKey** 列の統計を作成または更新します。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースに対して **dp203-*xxxxxxx*** リソース グループ (管理対象リソース グループ以外) を選択し、そこに Synapse ワークスペース、ストレージ アカウント、ワークスペースの Spark プールが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
