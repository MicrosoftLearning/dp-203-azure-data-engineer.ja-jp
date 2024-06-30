---
lab:
  title: Spark を使用してデータ レイク内のデータを分析する
  module: 'Model, query, and explore data in Azure Synapse'
---

# Spark を使用してデータ レイク内のデータを分析する

Apache Spark は、分散データ処理を行うためのオープン ソース エンジンであり、データ レイク ストレージ内の膨大な量のデータを探索、処理、分析するために広く使用されています。 Spark は、Azure HDInsight、Azure Databricks、Microsoft Azure クラウド プラットフォームの Azure Synapse Analytics など、多くのデータ プラットフォーム製品で処理オプションとして使用することができます。 Java、Scala、Python、SQL など、幅広いプログラミング言語に対応していることが Spark の利点の 1 つであり、これにより Spark は、データ クレンジングと操作、統計分析と機械学習、データ分析と視覚化など、データ処理ワークロードのソリューションとして高い柔軟性を実現しています。

このラボは完了するまで、約 **45** 分かかります。

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
    rm -r dp203 -f
    git clone https://github.com/MicrosoftLearning/DP-203-Azure-Data-Engineer dp203
    ```

5. リポジトリが複製されたら、次のコマンドを入力してこのラボ用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```
    cd dp203/Allfiles/labs/02
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 10 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Synapse Analytics ドキュメントの「[Azure Synapse Analytics での Apache Spark](https://docs.microsoft.com/azure/synapse-analytics/spark/apache-spark-overview)」の記事を確認してください。

## ファイル内のデータのクエリを実行する

このスクリプトを使用すると、Azure Synapse Analytics ワークスペースと、データ レイクをホストする Azure Storage アカウントがプロビジョニングされ、いくつかのデータ ファイルがデータ レイクにアップロードされます。

### データ レイク内のファイルを表示する

1. スクリプトが完了したら、Azure portal で、作成された **dp500-*xxxxxxx*** リソース グループに移動して、Synapse ワークスペースを選択します。
2. Synapse ワークスペースの **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。メッセージが表示された場合はサインインします。
3. Synapse Studio の左側にある **&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。これにより、リソースの管理とデータ分析タスクの実行に使用するさまざまなページが Synapse Studio 内に表示されます。
4. **[管理]** ページで、 **[Apache Spark pools]** タブを選択し、**spark*xxxxxxx*** のような名前の Spark プールがワークスペース内にプロビジョニングされていることに注意してください。 この Spark プールは、ワークスペースのデータ レイク ストレージ内のファイルからデータを読み込んで分析するために後で使用します。
5. **[データ]** ページで **[リンク]** タブを表示して、**synapse*xxxxxxx* (Primary - datalake*xxxxxxx*)** のような名前の Azure Data Lake Storage Gen2 ストレージ アカウントへのリンクがワークスペースに含まれていることを確認します。
6. ストレージ アカウントを展開して、そこに **files** という名前のファイル システム コンテナーが含まれていることを確認します。
7. この **files** コンテナーを選択し、そこに **sales** と **synapse** という名前のフォルダーが含まれていることに注意してください。 **synapse** フォルダーは、Azure Synapse によって使用されており、**sales** フォルダーには、これからクエリを実行するデータ ファイルが含まれています。
8. **sales** フォルダーと **orders** フォルダーを開き、3 年間の売上データの .csv ファイルが **orders** フォルダーに含まれていることを確認します。
9. いずれかのファイルを右クリックし、 **[プレビュー]** を選択して、そこに含まれているデータを表示します。 ファイルにはヘッダー行が含まれていないため、列ヘッダーを表示するには、オプションの選択を解除することに注意してください。

### Spark を使用してデータを探索する

1. **orders** フォルダー内でいずれかのファイルを選択したら、ツールバーの **[新しいノートブック]** の一覧で **[データフレームに読み込む]** を選択します。 データフレームは、表形式データセットを表す Spark の構造です。
2. 新しく開いた **[Notebook 1]** タブ内の **[アタッチ先]** の一覧で、Spark プール (**spark*xxxxxxx***) を選択します。 次に、 **&#9655; [すべて実行]** ボタンを使用して、ノートブック内のすべてのセル (現時点では 1 つしかありません) を実行します。

    このセッション内で Spark コードを実行したのはこれが最初であるため、Spark プールを起動する必要があります。 これは、セッション内での最初の実行には、数分かかる場合があることを意味します。 それ以降は、短時間で実行できます。

3. Spark セッションによる初期化を待っている間、生成されたコードを確認します。このコードは次のようになります。

    ```Python
    %%pyspark
    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/sales/orders/2019.csv', format='csv'
    ## If header exists uncomment line below
    ##, header=True
    )
    display(df.limit(10))
    ```

4. コードの実行が完了したら、ノートブック内のセルの下にある出力を確認します。 ここには、選択したファイル内の最初の 10 行と、 **_c0**、 **_c1**、 **_c2** などの形式の自動列名が表示されます。
5. **spark.read.load** 関数によってフォルダー内の<u>すべて</u>の CSV ファイルからデータが読み取られ、**display** 関数によって最初の 100 行が表示されるように、コードを変更します。 コードは次のようになります (*datalakexxxxxxx* はデータ レイク ストアの名前に一致します)。

    ```Python
    %%pyspark
    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/sales/orders/*.csv', format='csv'
    )
    display(df.limit(100))
    ```

6. コード セルの左側にある **&#9655;** ボタンを使用して、そのセルのみを実行し、結果を確認します。

    これで、このデータフレームにすべてのファイルからのデータが含まれるようになりましたが、列名はわかりにくいままです。 Spark では、"読み取り時のスキーマ" の方法を使用して、列に含まれているデータに基づき、列に適したデータ型を判断します。また、テキスト ファイル内にヘッダー行が存在する場合は、これを使用して列名を識別できます (この場合、**load** 関数に **header=True** パラメーターを指定します)。 または、データフレームの明示的なスキーマを定義することもできます。

7. 列名とデータ型が含まれているデータフレームの明示的なスキーマを定義するには、コードを次のように変更します (*datalakexxxxxxx* を置き換えます)。 セル内のコードを再実行します。

    ```Python
    %%pyspark
    from pyspark.sql.types import *
    from pyspark.sql.functions import *

    orderSchema = StructType([
        StructField("SalesOrderNumber", StringType()),
        StructField("SalesOrderLineNumber", IntegerType()),
        StructField("OrderDate", DateType()),
        StructField("CustomerName", StringType()),
        StructField("Email", StringType()),
        StructField("Item", StringType()),
        StructField("Quantity", IntegerType()),
        StructField("UnitPrice", FloatType()),
        StructField("Tax", FloatType())
        ])

    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/sales/orders/*.csv', format='csv', schema=orderSchema)
    display(df.limit(100))
    ```

8. 結果の下にある **[+ コード]** ボタンを使用して、ノートブックに新しいコード セルを追加します。 次に、新しいセルに次のコードを追加して、データフレームのスキーマを表示します。

    ```Python
    df.printSchema()
    ```

9. 新しいセルを実行し、そのデータフレームのスキーマが、定義した **orderSchema** に一致していることを確認します。 自動的に推論されたスキーマを持つデータフレームを使用する場合には、**printSchema** 関数が便利です。

## データフレーム内のデータを分析する

Spark の **dataframe** オブジェクトは、Python の Pandas データフレームに似ています。このオブジェクトには、そこに存在するデータの操作、フィルター処理、グループ化、または分析に使用できるさまざまな関数が含まれています。

### データフレームをフィルター処理する

1. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```Python
    customers = df['CustomerName', 'Email']
    print(customers.count())
    print(customers.distinct().count())
    display(customers.distinct())
    ```

2. この新しいコード セルを実行し、結果を確認します。 次の詳細を確認します。
    - データフレームに対して操作を実行すると、その結果として新しいデータフレームが作成されます (この場合、**df** データフレームから列の特定のサブセットを選択することで、新しい **customers** データフレームが作成されます)
    - データフレームには、そこに含まれているデータの集計やフィルター処理に使用できる **count** や **distinct** などの関数が用意されています。
    - `dataframe['Field1', 'Field2', ...]` 構文は、列のサブセットを簡単に定義できる方法です。 また、**select** メソッドを使用すると、上記のコードの最初の行を `customers = df.select("CustomerName", "Email")` のように記述することができます

3. コードを次のように変更します。

    ```Python
    customers = df.select("CustomerName", "Email").where(df['Item']=='Road-250 Red, 52')
    print(customers.count())
    print(customers.distinct().count())
    display(customers.distinct())
    ```

4. この変更したコードを実行すると、"*Road-250 Red, 52*" という製品を購入した顧客が表示されます。 複数の関数を "チェーン" にすると、1 つの関数の出力が次の関数の入力になることに注意してください。この場合、**select** メソッドによって作成されたデータフレームは、フィルター条件を適用するために使用される **where** メソッドのソース データフレームとなります。

### データフレーム内のデータを集計してグループ化する

1. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```Python
    productSales = df.select("Item", "Quantity").groupBy("Item").sum()
    display(productSales)
    ```

2. 追加したコード セルを実行し、その結果が、製品ごとにグループ化された注文数の合計を示していることに注意してください。 **groupBy** メソッドを使用すると、*Item* ごとに行がグループ化されます。その後の **sum** 集計関数は、残りのすべての数値列に適用されます (この場合は *Quantity*)

3. ノートブックに新しいコード セルをもう 1 つ追加し、そこに次のコードを入力します。

    ```Python
    yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
    display(yearlySales)
    ```

4. 追加したコード セルを実行し、その結果が 1 年あたりの販売注文数を示していることに注意してください。 **select** メソッドには *OrderDate* フィールドの year コンポーネントを抽出するための SQL **year** 関数が含まれており、抽出された year 値に列名を割り当てるために **alias** メソッドが使用されていることに注意してください。 次に、データは派生 *Year* 列によってグループ化され、各グループの行数が計算されます。その後、結果として生成されたデータフレームを並べ替えるために、最後に **orderBy** メソッドが使用されます。

## Spark SQL を使用してデータのクエリを実行する

ここまで確認してきたように、データフレーム オブジェクトのネイティブ メソッドを使用すると、データのクエリの実行と分析を効率的に行うことができます。 ただし、SQL 構文の方が扱いやすいと考えるデータ アナリストも多くいます。 Spark SQL は、Spark の SQL 言語 API であり、SQL ステートメントの実行だけではなく、リレーショナル テーブル内でのデータの永続化にも使用できます。

### PySpark コードで Spark SQL を使用する

Azure Synapse Studio ノートブックの既定の言語は、Spark ベースの Python ランタイムである PySpark です。 このランタイム内に **spark.sql** ライブラリを使用すると、Python コード内に Spark SQL 構文を埋め込んだり、テーブルやビューなどの SQL コンストラクトを扱ったりすることができます。

1. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```Python
    df.createOrReplaceTempView("salesorders")

    spark_df = spark.sql("SELECT * FROM salesorders")
    display(spark_df)
    ```

2. このセルを実行し、結果を確認します。 次の点に注意してください。
    - このコードによって、**df** データフレーム内のデータが、**salesorders** という名前の一時ビューとして永続化されます。 Spark SQL は、一時ビューまたは永続化されたテーブルを SQL クエリのソースとして使用することをサポートしています。
    - 次に、**spark.sql** メソッドを使用して、**salesorders** ビューに対して SQL クエリが実行されます。
    - クエリの結果は、データフレームに格納されます。

### SQL コードをセル内で実行する

PySpark コードが含まれているセルに SQL ステートメントを埋め込むことができるのは便利ですが、データ アナリストにとっては、SQL で直接作業できればよいという場合も多くあります。

1. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```sql
    %%sql
    SELECT YEAR(OrderDate) AS OrderYear,
           SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue
    FROM salesorders
    GROUP BY YEAR(OrderDate)
    ORDER BY OrderYear;
    ```

2. このセルを実行し、結果を確認します。 次の点に注意してください。
    - セルの先頭にある `%%sql` 行 (*magic* と呼ばれます) は、このセル内でこのコードを実行するには、PySpark ではなく、Spark SQL 言語ランタイムを使用する必要があることを示しています。
    - SQL コードを使用すると、前に PySpark を使用して作成した **salesorder** ビューが参照されます。
    - SQL クエリからの出力は、セルの下に自動的に結果として表示されます。

> **注**: Spark SQL とデータフレームの詳細については、[Spark SQL のドキュメント](https://spark.apache.org/docs/2.2.0/sql-programming-guide.html)を参照してください。

## Spark を使用してデータを視覚化する

画像が何千もの言葉を語り、表が何千行にも及ぶデータよりもわかりやすいことは、だれもが知っています。 Azure Synapse Analytics のノートブックには、データフレームまたは Spark SQL クエリから表示されるデータのグラフ ビューが組み込まれていますが、これは包括的なグラフ作成を目的としたものではありません。 ただし、**matplotlib** や **seaborn** などの Python グラフィックス ライブラリを使用して、データフレーム内のデータからグラフを作成することができます。

### 結果をグラフで表示する

1. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```sql
    %%sql
    SELECT * FROM salesorders
    ```

2. コードを実行し、前に作成した **salesorders** ビューからのデータが返されることを確認します。
3. セルの下の結果セクションで、 **[ビュー]** オプションを **[表]** から **[グラフ]** に変更します。
4. グラフの右上にある **[ビュー オプション]** ボタンを使用して、グラフのオプション ペインを表示します。 オプションを次のように設定し、 **[適用]** を選択します。
    - **グラフの種類**: 横棒グラフ
    - **キー**: Item
    - **値**: Quantity
    - **系列グループ**: "空白のままにする"**
    - **集計**: SUM
    - **積み上げ**: "未選択"**

5. グラフが次のように表示されていることを確認します。

    ![製品の合計注文数別の横棒グラフ](./images/notebook-chart.png)

### **matplotlib** の使用を開始する

1. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```Python
    sqlQuery = "SELECT CAST(YEAR(OrderDate) AS CHAR(4)) AS OrderYear, \
                    SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue \
                FROM salesorders \
                GROUP BY CAST(YEAR(OrderDate) AS CHAR(4)) \
                ORDER BY OrderYear"
    df_spark = spark.sql(sqlQuery)
    df_spark.show()
    ```

2. コードを実行し、年間収益を含む Spark データフレームが返されることを確認します。

    データをグラフとして視覚化するには、まず **matplotlib** Python ライブラリを使用します。 このライブラリは、他の多くのライブラリに基づいたコア プロット ライブラリであり、これを使用するとグラフ作成時の柔軟性が大幅に向上します。

3. ノートブックに新しいコード セルを追加し、そこに次のコードを追加します。

    ```Python
    from matplotlib import pyplot as plt

    # matplotlib requires a Pandas dataframe, not a Spark one
    df_sales = df_spark.toPandas()

    # Create a bar plot of revenue by year
    plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'])

    # Display the plot
    plt.show()
    ```

4. このセルを実行し、結果を確認します。結果には、各年の総収益が縦棒グラフで示されています。 このグラフの生成に使用されているコードの次の機能に注目してください。
    - **matplotlib** ライブラリには *Pandas* データフレームが必要であるため、Spark SQL クエリによって返される *Spark* データフレームをこの形式に変換する必要があります。
    - **matplotlib** ライブラリの中核となるのは、**pyplot** オブジェクトです。 これは、ほとんどのプロット機能の基礎となります。
    - 既定の設定では、使用可能なグラフが生成されますが、カスタマイズすべき範囲が大幅に増えます。

5. コードを次のように変更して、グラフをプロットします。

    ```Python
    # Clear the plot area
    plt.clf()

    # Create a bar plot of revenue by year
    plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')

    # Customize the chart
    plt.title('Revenue by Year')
    plt.xlabel('Year')
    plt.ylabel('Revenue')
    plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
    plt.xticks(rotation=45)

    # Show the figure
    plt.show()
    ```

6. コード セルを再実行し、結果を表示します。 これで、もう少し詳細な情報がグラフに含まれるようになりました。

    プロットは、技術的に**図**に含まれています。 前の例では、図が暗黙的に作成されていましたが、明示的に作成することもできます。

7. コードを次のように変更して、グラフをプロットします。

    ```Python
    # Clear the plot area
    plt.clf()

    # Create a Figure
    fig = plt.figure(figsize=(8,3))

    # Create a bar plot of revenue by year
    plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')

    # Customize the chart
    plt.title('Revenue by Year')
    plt.xlabel('Year')
    plt.ylabel('Revenue')
    plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
    plt.xticks(rotation=45)

    # Show the figure
    plt.show()
    ```

8. コード セルを再実行し、結果を表示します。 図によって、プロットの形状とサイズが決まります。

    図には複数のサブプロットが含まれており、それぞれに独自の "軸" があります**。

9. コードを次のように変更して、グラフをプロットします。

    ```Python
    # Clear the plot area
    plt.clf()

    # Create a figure for 2 subplots (1 row, 2 columns)
    fig, ax = plt.subplots(1, 2, figsize = (10,4))

    # Create a bar plot of revenue by year on the first axis
    ax[0].bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
    ax[0].set_title('Revenue by Year')

    # Create a pie chart of yearly order counts on the second axis
    yearly_counts = df_sales['OrderYear'].value_counts()
    ax[1].pie(yearly_counts)
    ax[1].set_title('Orders per Year')
    ax[1].legend(yearly_counts.keys().tolist())

    # Add a title to the Figure
    fig.suptitle('Sales Data')

    # Show the figure
    plt.show()
    ```

10. コード セルを再実行し、結果を表示します。 図には、コード内に指定したサブプロットが含まれています。

> **注**: matplotlib を使用したプロットの詳細については、[matplotlib のドキュメント](https://matplotlib.org/)を参照してください。

### **seaborn** ライブラリを使用する

**matplotlib** を使用すると、複数の種類の複雑なグラフを作成することができますが、優れた結果を得るには、複雑なコードを必要とする場合があります。 このため、複雑性の抽象化と機能の強化を目的に、matplotlib のベース上に数多くの新しいライブラリが何年もかけて構築されてきました。 そのようなライブラリの 1 つに **seaborn** があります。

1. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```Python
    import seaborn as sns

    # Clear the plot area
    plt.clf()

    # Create a bar chart
    ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
    plt.show()
    ```

2. コードを実行し、seaborn ライブラリを使用した横棒グラフが表示されていることを確認します。
3. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```Python
    # Clear the plot area
    plt.clf()

    # Set the visual theme for seaborn
    sns.set_theme(style="whitegrid")

    # Create a bar chart
    ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
    plt.show()
    ```

4. コードを実行し、seaborn によってプロットに一貫性のある配色テーマが設定されていることに注意します。

5. ノートブックに新しいコード セルを追加し、そこに次のコードを入力します。

    ```Python
    # Clear the plot area
    plt.clf()

    # Create a bar chart
    ax = sns.lineplot(x="OrderYear", y="GrossRevenue", data=df_sales)
    plt.show()
    ```

6. コードを実行し、年間収益を折れ線グラフで表示します。

> **注**: seaborn を使用したプロットの詳細については、[seaborn のドキュメント](https://seaborn.pydata.org/index.html)を参照してください。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースに対して **dp500-*xxxxxxx*** リソース グループ (マネージド リソース グループ以外) を選択し、そこに Synapse ワークスペース、ストレージ アカウント、ワークスペースの Spark プールが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp500-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
