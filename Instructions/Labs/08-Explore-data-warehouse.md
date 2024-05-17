---
lab:
  title: リレーショナル データ ウェアハウスについて調べる
  ilt-use: Suggested demo
---

# リレーショナル データ ウェアハウスについて調べる

Azure Synapse Analytics は、エンタープライズ データ ウェアハウスをサポートするスケーラブルなセット機能に基づいて構築されています。これには、データ レイクや大規模なリレーショナル データ ウェアハウスでのファイル ベースのデータ分析と、それらの読み込みに使用されるデータ転送および変換パイプラインが含まれます。 このラボでは、Azure Synapse Analytics の専用 SQL プールを使用して、リレーショナル データ ウェアハウスにデータを格納および照会する方法について説明します。

このラボは完了するまで、約 **45** 分かかります。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Synapse Analytics ワークスペースをプロビジョニングする

Azure Synapse Analytics "ワークスペース"は、データとデータ処理ランタイムを管理するための一元的な場所を提供するものです。** Azure portal の対話型インターフェイスを使用してワークスペースをプロビジョニングすることも、スクリプトまたはテンプレートを使用してワークスペースとその中のリソースをデプロイすることもできます。 ほとんどの運用シナリオでは、スクリプトとテンプレートを使用してプロビジョニングを自動化し、反復可能な開発と運用 ("DevOps") プロセスにリソースのデプロイを組み込めるようにすることをお勧めします。**

この演習では、Azure Synapse Analytics ワークスペースをプロビジョニングするために、PowerShell スクリプトと ARM テンプレートを組み合わせて使用します。

1. [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示された場合は、***PowerShell*** 環境を選択して、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#9723;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

    ```
    rm -r dp203 -f
    git clone  https://github.com/MicrosoftLearning/Dp-203-azure-data-engineer dp203
    ```

5. リポジトリが複製されたら、次のコマンドを入力してこのラボ用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```
    cd dp203/Allfiles/labs/08
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 15 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Synapse Analytics ドキュメントの記事「[Azure Synapse Analytics の専用 SQL プールとは](https://docs.microsoft.com/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-overview-what-is)」を確認してください。

## データ ウェアハウス スキーマを調べる

このラボでは、データ ウェアハウスは Azure Synapse Analytics の専用 SQL プールでホストされます。

### 専用 SQL プールを起動する

1. スクリプトが完了したら、Azure portal で、作成された **dp500-*xxxxxxx*** リソース グループに移動して、Synapse ワークスペースを選択します。
2. Synapse ワークスペースの **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。メッセージが表示された場合はサインインします。
3. Synapse Studio の左側にある **&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。これにより、リソースの管理とデータ分析タスクの実行に使用されるさまざまなページが Synapse Studio 内に表示されます。
4. **[管理]** ページで、 **[SQL プール]** タブが選択されていることを確認した後、専用 SQL プール **sql*xxxxxxx*** を選択し、 **&#9655;** アイコンを使用して起動します。メッセージが表示されたら、再開することを確認します。
5. SQL プールが再開されるまで待ちます。 これには数分かかることがあります。 **&#8635; [更新]** ボタンを使用して、その状態を定期的に確認してください。 準備ができると、状態が **[オンライン]** と表示されます。

### データベース内のテーブルを表示する

1. Synapse Studio で、 **[データ]** ページを選択し、 **[ワークスペース]** タブが選択されていることと、 **[SQL データベース]** カテゴリが含まれていることを確認します。
2. **[SQL データベース]** 、 **[sql*xxxxxxx]** * プール、およびその **[Tables]** フォルダーを展開して、データベース内のテーブルを表示します。

    リレーショナル データ ウェアハウスは、通常、"ファクト" テーブルと "ディメンション" テーブルで構成されるスキーマに基づいています。** ** テーブルは分析クエリ用に最適化されていて、ファクト テーブル内の数値メトリックは、ディメンション テーブルによって表されるエンティティの属性によって集計されます。たとえば、製品別、顧客別、日付別などでインターネット売上収益を集計できます。
    
3. **dbo.FactInternetSales** テーブルとその **[Columns]** フォルダーを展開して、このテーブルの列を表示します。 列の多くは、ディメンション テーブル内の行を参照する "キー" であることに注意してください。** その他は、分析用の数値 ("メジャー") です。**
    
    キーは、ファクト テーブルを 1 つ以上のディメンション テーブル (多くの場合 "スター" スキーマ内) に関連付けるために使用されます。ファクト テーブルは各ディメンション テーブルに直接関連付けられます (ファクト テーブルを中心に複数のポイントを持つ "スター" を形成します)。**

4. **dbo.DimPromotion** テーブルの列を表示します。テーブル内の各行を一意に識別する、一意の **PromotionKey** があることに注意してください。 また、**AlternateKey** もあります。

    通常、データ ウェアハウス内のデータは、1 つ以上のトランザクション ソースからインポートされています。 "代替" キーはソース内のこのエンティティのインスタンスのビジネス識別子を反映しますが、通常は、データ ウェアハウス ディメンション テーブル内の各行を一意に識別するために、"サロゲート" キーという一意の数値が生成されます。** ** この方法の利点の 1 つは、同じエンティティについて、異なる時点での複数のインスタンス (たとえば、同じ顧客の注文時点での住所を反映するレコード) をデータ ウェアハウスに含めることができるということです。

5. **dbo.DimProduct** の列を表示します。**ProductSubcategoryKey** 列が含まれていて、それが **dbo.DimProductSubcategory** を参照していることに注意してください。さらに、そのテーブルに **ProductCategoryKey** 列が含まれていて、それが **dbo.DimProductCategory** を参照しています。

    場合によっては、製品をサブカテゴリやカテゴリにグループ化できるようにするなど、さまざまなレベルの粒度に対応するために、ディメンションが複数の関連テーブルに部分的に正規化されます。 その場合は、単純なスターが "スノーフレーク" スキーマに拡張されます。スノーフレーク スキーマでは、中央のファクト テーブルがディメンション テーブルに関連付けられ、そこからさらに複数のディメンション テーブルへと関連付けられます。**

6. **dbo.DimDate** テーブルの列を表示します。日付のさまざまなテンポラル属性を反映する、複数の列が含まれていることに注意してください (曜日、日、月、年、日名、月名など)。

    データ ウェアハウスの時間ディメンションは、通常、ファクト テーブル内のメジャーを集計するための最小時間単位 (ディメンションの "グレイン" と呼ばれます) ごとの行を含んだディメンション テーブルとして実装されます。** この場合、メジャーを集計できる最も低いグレインは個々の日付であり、テーブルには、データで参照される最初の日付から最後の日付までの各日付の行が含まれます。 **DimDate** テーブルの属性を使用すると、アナリストは、一貫性のある一連のテンポラル属性を使用し、ファクト テーブル内の日付キーに基づいてメジャーを集計できます (たとえば、注文日に基づいて月単位で注文を表示するなど)。 **FactInternetSales** テーブルには、**DimDate** テーブルに関連する 3 つのキー (**OrderDateKey**、**DueDateKey**、**ShipDateKey**) が含まれています。

## データ ウェアハウス テーブルの作成

データ ウェアハウス スキーマの特に重要な側面についておわかりいただけたと思うので、次は、テーブルのクエリを実行してデータを取得してみましょう。

### ファクト テーブルとディメンション テーブルに対してクエリを実行する

リレーショナル データ ウェアハウス内の数値は、関連するディメンション テーブルと共にファクト テーブルに格納され、複数の属性でデータを集計できるようになっています。 このような設計であるため、リレーショナル データ ウェアハウスのほとんどのクエリでは、(JOIN 句を使用して) 関連するテーブル間で (集計関数と GROUP BY 句を使用して) データが集計およびグループ化されます。

1. **[データ]** ページで、SQL プール **sql*xxxxxxx*** を選択し、その **[...]** メニューで **[新しい SQL スクリプト]**  >  **[空のスクリプト**] を選択します。
2. 新しい **[SQL スクリプト 1]** タブが開いたら、 **[プロパティ]** ウィンドウでスクリプトの名前を「**Analyze Internet Sales**」に変更し、 **[クエリごとの結果設定]** を変更してすべての行が返されるようにします。 その後、ツール バーの **[発行]** ボタンを使用してスクリプトを保存し、ツール バーの右端にある **[プロパティ]** ボタン ( **&#128463;** のような外観) を使用して **[プロパティ]** ペインを閉じ、スクリプト ペインが見えるようにします。
3. 空のスクリプトで、次のコードを追加します。

    ```sql
    SELECT  d.CalendarYear AS Year,
            SUM(i.SalesAmount) AS InternetSalesAmount
    FROM FactInternetSales AS i
    JOIN DimDate AS d ON i.OrderDateKey = d.DateKey
    GROUP BY d.CalendarYear
    ORDER BY Year;
    ```

4. **&#9655; [実行]** ボタンを使用してスクリプトを実行し、結果を確認します。この結果には、各年のインターネット売上の合計が表示されます。 このクエリは、インターネット販売のファクト テーブルを注文日に基づいて時間ディメンション テーブルに結合し、ディメンション テーブルのカレンダー月属性によってファクト テーブルの売上金額メジャーを集計するものです。

5. クエリを次のように変更して、時間ディメンションから月属性を追加し、変更後のクエリを実行します。

    ```sql
    SELECT  d.CalendarYear AS Year,
            d.MonthNumberOfYear AS Month,
            SUM(i.SalesAmount) AS InternetSalesAmount
    FROM FactInternetSales AS i
    JOIN DimDate AS d ON i.OrderDateKey = d.DateKey
    GROUP BY d.CalendarYear, d.MonthNumberOfYear
    ORDER BY Year, Month;
    ```

    時間ディメンションの属性を使用すると、ファクト テーブル内のメジャーを複数の階層レベル (この場合は年と月) で集計できます。 これは、データ ウェアハウスでの一般的なパターンです。

6. クエリを次のように変更して月を削除し、2 番目のディメンションを集計に追加した後、それを実行して結果を表示します (各リージョンの年間インターネット売上の合計が表示されます)。

    ```sql
    SELECT  d.CalendarYear AS Year,
            g.EnglishCountryRegionName AS Region,
            SUM(i.SalesAmount) AS InternetSalesAmount
    FROM FactInternetSales AS i
    JOIN DimDate AS d ON i.OrderDateKey = d.DateKey
    JOIN DimCustomer AS c ON i.CustomerKey = c.CustomerKey
    JOIN DimGeography AS g ON c.GeographyKey = g.GeographyKey
    GROUP BY d.CalendarYear, g.EnglishCountryRegionName
    ORDER BY Year, Region;
    ```

    geography は、顧客ディメンションを通じてインターネット販売ファクト テーブルに関連付けられる "スノーフレーク" ディメンションであることに注意してください。** そのため、インターネットの売上を地域別に集計するには、クエリに 2 つの結合が必要です。

7. クエリを変更して別のスノーフレーク ディメンションを追加し、クエリを再実行して、製品カテゴリ別に年間地域売上を集計します。

    ```sql
    SELECT  d.CalendarYear AS Year,
            pc.EnglishProductCategoryName AS ProductCategory,
            g.EnglishCountryRegionName AS Region,
            SUM(i.SalesAmount) AS InternetSalesAmount
    FROM FactInternetSales AS i
    JOIN DimDate AS d ON i.OrderDateKey = d.DateKey
    JOIN DimCustomer AS c ON i.CustomerKey = c.CustomerKey
    JOIN DimGeography AS g ON c.GeographyKey = g.GeographyKey
    JOIN DimProduct AS p ON i.ProductKey = p.ProductKey
    JOIN DimProductSubcategory AS ps ON p.ProductSubcategoryKey = ps.ProductSubcategoryKey
    JOIN DimProductCategory AS pc ON ps.ProductCategoryKey = pc.ProductCategoryKey
    GROUP BY d.CalendarYear, pc.EnglishProductCategoryName, g.EnglishCountryRegionName
    ORDER BY Year, ProductCategory, Region;
    ```

    ここでは、製品、サブカテゴリ、カテゴリ間の階層関係を反映するために、製品カテゴリのスノーフレーク ディメンションに 3 つの結合が必要となっています。

8. スクリプトを発行して保存します。

### 順位付け関数を使用する

大量のデータを分析する際のもう 1 つの一般的な要件は、パーティションごとにデータをグループ化し、特定のメトリックに基づいてパーティション内の各エンティティの "ランク" を決定することです。**

1. 既存のクエリの下に次の SQL を追加して、国/地域名に基づくパーティションごとの 2022 年の売上値を取得します。

    ```sql
    SELECT  g.EnglishCountryRegionName AS Region,
            ROW_NUMBER() OVER(PARTITION BY g.EnglishCountryRegionName
                              ORDER BY i.SalesAmount ASC) AS RowNumber,
            i.SalesOrderNumber AS OrderNo,
            i.SalesOrderLineNumber AS LineItem,
            i.SalesAmount AS SalesAmount,
            SUM(i.SalesAmount) OVER(PARTITION BY g.EnglishCountryRegionName) AS RegionTotal,
            AVG(i.SalesAmount) OVER(PARTITION BY g.EnglishCountryRegionName) AS RegionAverage
    FROM FactInternetSales AS i
    JOIN DimDate AS d ON i.OrderDateKey = d.DateKey
    JOIN DimCustomer AS c ON i.CustomerKey = c.CustomerKey
    JOIN DimGeography AS g ON c.GeographyKey = g.GeographyKey
    WHERE d.CalendarYear = 2022
    ORDER BY Region;
    ```

2. 新しいクエリ コードのみを選択し、 **&#9655; [実行]** ボタンを使用して実行します。 その後、結果を確認します。次の表のようになります。

    | リージョン | RowNumber | OrderNo | LineItem | SalesAmount | RegionTotal | RegionAverage |
    |--|--|--|--|--|--|--|
    |オーストラリア|1|SO73943|2|2.2900|2172278.7900|375.8918|
    |オーストラリア|2|SO74100|4|2.2900|2172278.7900|375.8918|
    |...|...|...|...|...|...|...|
    |オーストラリア|5779|SO64284|1|2443.3500|2172278.7900|375.8918|
    |Canada|1|SO66332|2|2.2900|563177.1000|157.8411|
    |Canada|2|SO68234|2|2.2900|563177.1000|157.8411|
    |...|...|...|...|...|...|...|
    |Canada|3568|SO70911|1|2443.3500|563177.1000|157.8411|
    |フランス|1|SO68226|3|2.2900|816259.4300|315.4016|
    |フランス|2|SO63460|2|2.2900|816259.4300|315.4016|
    |...|...|...|...|...|...|...|
    |フランス|2588|SO69100|1|2443.3500|816259.4300|315.4016|
    |ドイツ|1|SO70829|3|2.2900|922368.2100|352.4525|
    |ドイツ|2|SO71651|2|2.2900|922368.2100|352.4525|
    |...|...|...|...|...|...|...|
    |ドイツ|2617|SO67908|1|2443.3500|922368.2100|352.4525|
    |イギリス|1|SO66124|3|2.2900|1051560.1000|341.7484|
    |イギリス|2|SO67823|3|2.2900|1051560.1000|341.7484|
    |...|...|...|...|...|...|...|
    |イギリス|3077|SO71568|1|2443.3500|1051560.1000|341.7484|
    |United States|1|SO74796|2|2.2900|2905011.1600|289.0270|
    |United States|2|SO65114|2|2.2900|2905011.1600|289.0270|
    |...|...|...|...|...|...|...|
    |United States|10051|SO66863|1|2443.3500|2905011.1600|289.0270|

    これらの結果について、次の事実を確認してください。

    - 販売注文品目ごとに 1 つの行があります。
    - 行は、販売が行われた地域に基づいてパーティション別に編成されています。
    - 各地理的パーティション内の行には、売上金額の順に番号が付けられています (最小から最大)。
    - 各行に対して、品目の売上金額と、地域別の合計および平均売上金額が含まれています。

3. 既存のクエリの下に次のコードを追加して、GROUP BY クエリ内にウィンドウ関数を適用し、販売額の合計に基づいて各地域の都市をランク付けします。

    ```sql
    SELECT  g.EnglishCountryRegionName AS Region,
            g.City,
            SUM(i.SalesAmount) AS CityTotal,
            SUM(SUM(i.SalesAmount)) OVER(PARTITION BY g.EnglishCountryRegionName) AS RegionTotal,
            RANK() OVER(PARTITION BY g.EnglishCountryRegionName
                        ORDER BY SUM(i.SalesAmount) DESC) AS RegionalRank
    FROM FactInternetSales AS i
    JOIN DimDate AS d ON i.OrderDateKey = d.DateKey
    JOIN DimCustomer AS c ON i.CustomerKey = c.CustomerKey
    JOIN DimGeography AS g ON c.GeographyKey = g.GeographyKey
    GROUP BY g.EnglishCountryRegionName, g.City
    ORDER BY Region;
    ```

4. 新しいクエリ コードのみを選択し、 **&#9655; [実行]** ボタンを使用して実行します。 その後、結果を確認し、次の点を確認してください。
    - 結果には、地域別にグループ化された各都市の行が含まれています。
    - 都市ごとの合計売上 (個々の売上金額の合計) が計算されています
    - 地域パーティションに基づいて、地域の売上合計 (地域内の各都市の売上金額の合計) が計算されています。
    - 都市ごとの合計売上金額を降順で並べ替えることで、地域パーティション内の各都市のランクが計算されています。

5. 更新されたスクリプトを発行して変更を保存します。

> **ヒント**: ROW_NUMBER と RANK は、Transact-SQL で使用できるランク付け関数の例です。 詳細については、Transact-SQL 言語ドキュメントの[ランク付け関数](https://docs.microsoft.com/sql/t-sql/functions/ranking-functions-transact-sql)のリファレンスを参照してください。

### 概算数を取得する

非常に大量のデータを探索する場合、クエリの実行にかなりの時間とリソースが必要になる場合があります。 多くの場合、データ分析では完全に正確な値は必要ありません。近似値の比較で十分な場合があります。

1. 既存のクエリの下に次のコードを追加して、各暦年の販売注文の数を取得します。

    ```sql
    SELECT d.CalendarYear AS CalendarYear,
        COUNT(DISTINCT i.SalesOrderNumber) AS Orders
    FROM FactInternetSales AS i
    JOIN DimDate AS d ON i.OrderDateKey = d.DateKey
    GROUP BY d.CalendarYear
    ORDER BY CalendarYear;
    ```

2. 新しいクエリ コードのみを選択し、 **&#9655; [実行]** ボタンを使用して実行します。 その後、返された出力を確認します。
    - クエリの **[結果]** タブで、各年の注文数を表示します。
    - **[メッセージ]** タブで、クエリの合計実行時間を表示します。
3. クエリを次のように変更して、各年の概算数が返されるようにします。 その後、クエリを再実行します。

    ```sql
    SELECT d.CalendarYear AS CalendarYear,
        APPROX_COUNT_DISTINCT(i.SalesOrderNumber) AS Orders
    FROM FactInternetSales AS i
    JOIN DimDate AS d ON i.OrderDateKey = d.DateKey
    GROUP BY d.CalendarYear
    ORDER BY CalendarYear;
    ```

4. 返された出力を確認します。
    - クエリの **[結果]** タブで、各年の注文数を表示します。 これらは、前のクエリで取得された実際のカウントの 2% 以内になっているはずです。
    - **[メッセージ]** タブで、クエリの合計実行時間を表示します。 これは、前のクエリよりも短くなっているはずです。

5. スクリプトを発行して変更を保存します。

> **ヒント**: 詳しくは、[APPROX_COUNT_DISTINCT](https://docs.microsoft.com/sql/t-sql/functions/approx-count-distinct-transact-sql) 関数の資料を参照してください。

## チャレンジ - リセラーの売上を分析する

1. SQL プール **sql*xxxxxxx*** に対する新しい空のスクリプトを作成し、「**Analyze Reseller Sales**」という名前で保存します。
2. スクリプトで SQL クエリを作成し、**FactResellerSales** ファクト テーブルと、それが関連付けられているディメンション テーブルに基づいて、次の情報を検索します。
    - 会計年度および四半期ごとの販売品目の合計数量。
    - 販売を行った従業員に関連付けられている、会計年度、四半期、および販売地域ごとの販売品目の合計数量。
    - 会計年度、四半期、および販売地域ごとの、製品カテゴリ別の販売品目の合計数量。
    - 年度の総売上金額に基づく、会計年度ごとの各販売区域のランク。
    - 各販売地域の 1 年あたりの販売注文の概算数。

    > **ヒント**: 作成したクエリを、Synapse Studio の **[開発]** ページにある **[ソリューション]** スクリプトのクエリと比較してください。

3. 興味があれば、色々なクエリを試して、データ ウェアハウス スキーマ内の残りのテーブルを調べてみてください。
4. 完了したら、 **[管理]** ページで、専用 SQL プール **sql*xxxxxxx*** を一時停止します。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースに対して **dp500-*xxxxxxx*** リソース グループ (管理対象リソース グループ以外) を選択し、そこに Synapse ワークスペース、ストレージ アカウント、ワークスペースの専用 SQL プールが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp500-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
