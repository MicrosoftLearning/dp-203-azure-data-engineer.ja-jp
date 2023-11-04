---
lab:
  title: Azure Synapse Analytics の詳細
  ilt-use: Lab
---

# Azure Synapse Analytics の詳細

Azure Synapse Analytics は、エンドツーエンドのデータ分析用の単一の統合データ分析プラットフォームを提供するものです。 この演習では、データを取り込んで探索するさまざまな方法について調べます。 この演習は、Azure Synapse Analytics のさまざまなコア機能の概要として設計されています。 その他の演習では、特定の機能について詳しく調べることができます。

この演習の所要時間は約 **60** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Synapse Analytics ワークスペースをプロビジョニングする

Azure Synapse Analytics "ワークスペース"は、データとデータ処理ランタイムを管理するための一元的な場所を提供するものです。** Azure portal の対話型インターフェイスを使用してワークスペースをプロビジョニングすることも、スクリプトまたはテンプレートを使用してワークスペースとその中のリソースをデプロイすることもできます。 ほとんどの運用シナリオでは、スクリプトとテンプレートを使用してプロビジョニングを自動化し、反復可能な開発と運用 ("DevOps") プロセスにリソースのデプロイを組み込めるようにすることをお勧めします。**

この演習では、Azure Synapse Analytics ワークスペースをプロビジョニングするために、PowerShell スクリプトと ARM テンプレートを組み合わせて使用します。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示されたら、***PowerShell*** 環境を選んで、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

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
    cd dp-203/Allfiles/labs/01
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。 また、パスワードにログイン名の全部または一部を含めることはできません。

8. スクリプトが完了するまで待ちます。通常、約 20 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Synapse Analytics ドキュメントの「[Azure Synapse Analytics とは](https://docs.microsoft.com/azure/synapse-analytics/overview-what-is)」の記事を確認してください。

## Synapse Studio を探索する

*Synapse Studio* は、Azure Synapse Analytics ワークスペース内のリソースを管理および操作できる Web ベースのポータルです。

1. セットアップ スクリプトの実行が完了したら、Azure portal で、作成された **dp203-*xxxxxxx*** リソース グループに移動し、このリソース グループに Synapse ワークスペース、データ レイク用のストレージ アカウント、Apache Spark プール、Data Explorer プール、専用 SQL プールが含まれていることを確認します。
2. Synapse ワークスペースを選択し、 **[概要]** ページの **[Synapse Studio を開く]** カードで、 **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。Synapse Studio は、Synapse Analytics ワークスペースの操作に使用できる Web ベースのインターフェイスです。
3. Synapse Studio の左側で、**&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。次に示すように、リソースの管理とデータ分析タスクの実行に使用するさまざまなページが Synapse Studio 内に表示されます。

    ![リソースの管理とデータ分析タスクの実行用に展開した Synapse Studio メニューを示す画像](./images/synapse-studio.png)

4. **[データ]** ページを表示し、データ ソースを含む 2 つのタブがあることに注目します。
    - **[ワークスペース]** タブには、ワークスペースに定義されているデータベース (専用 SQL データベースと Data Explorer データベースを含む) が含まれます
    - **[リンク済み]** タブには、ワークスペースにリンクされている Azure Data Lake ストレージなどのデータ ソースが含まれます。

5. **[開発]** ページを表示します。現在これは空です。 ここでは、データ処理ソリューションの開発に使用するスクリプトやその他の資産を定義できます。
6. **[統合]** ページを表示します。これも空です。 このページを使用して、データ インジェストと統合資産を管理します。データ ソース間でデータを転送および変換するためのパイプラインなどです。
7. **[監視]** ページを表示します。 ここでは、データ処理ジョブを実行して履歴を表示しながら観察できます。
8. **[管理]** ページを表示します。 ここでは、Azure Synapse ワークスペースで使用するプール、ランタイム、その他の資産を管理します。 **[Analytics プール]** セクションの各タブを表示し、ワークスペースに次のプールが含まれていることに注目します。
    - **SQL プール**:
        - **組み込み**: SQL コマンドを使用してデータ レイク内のデータをオンデマンドで探索または処理できる、"サーバーレス" SQL プール。**
        - **sql*xxxxxxx***: リレーショナル データ ウェアハウス データベースをホストする "専用" SQL プール。**
    - **Apache Spark プール**:
        - **spark*xxxxxxx***: Scala や Python などのプログラミング言語を使用して、データ レイク内のデータをオンデマンドで探索または処理できます。
<!---    - **Data Explorer pools**:
        - **adx*xxxxxxx***: A Data Explorer pool that you can use to analyze data by using Kusto Query Language (KQL). --->

## パイプラインを使用してデータを取り込む

Azure Synapse Analytics を使用して実行できる主なタスクの 1 つは、さまざまなソースから分析用にデータをワークスペースに転送 (必要に応じて変換) する*パイプライン*を定義することです。

### データのコピー タスクを使用してパイプラインを作成する

1. Synapse Studio の **[ホーム]** ページで、**[取り込み]** を選択して、**データ コピー** ツールを開きます。
2. データ コピー ツールの **[プロパティ]** ステップで、**[組み込みコピー タスク]** と **[1 回実行する]** が選択されていることを確認し、**[次へ >]** をクリックします。
3. **[ソース]** ステップの **[データセット]** サブステップで、次の設定を選択します。
    - **[ソースの種類]**: すべて
    - **接続**: "新しい接続を作成し、表示される **[リンク サービス]** ペインの **[汎用プロトコル]** タブで、 **[HTTP]** を選びます。続けて、次の設定を使ってデータ ファイルへの接続を作成します。"**
        - **名前**: 製品
        - **説明**: HTTP 経由の製品一覧
        - **統合ランタイム経由で接続する**: AutoResolveIntegrationRuntime
        - **ベース URL**: `https://raw.githubusercontent.com/MicrosoftLearning/dp-203-azure-data-engineer/master/Allfiles/labs/01/adventureworks/products.csv`
        - **サーバー証明書の検証**: 有効にする
        - **認証の種類**: 匿名
4. 接続を作成したら、 **[ソース データ ストア]** ページで、次の設定が選択されていることを確認し、 **[次へ >]** を選択します。
    - **相対 URL**: *空白のまま*
    - **要求メソッド**:GET
    - **追加ヘッダー** :*空白のまま*
    - **バイナリ コピー**: 選択<u>解除</u>
    - **要求タイムアウト**: *空白のまま*
    - **最大同時接続数**: *空白のまま*
5. **[ソース]** ステップの **[構成]** サブステップで、**[データのプレビュー]** を選択して、パイプラインに取り込む製品データのプレビューを表示し、プレビューを閉じます。
6. データをプレビューしたら、 **[ファイル形式設定]** ページで、次の設定が選択されていることを確認し、 **[次へ >]** を選択します。
    - **ファイル形式**: DelimitedText
    - **列区切り記号**: コンマ (,)
    - **行区切り記号**: 改行 (\n)
    - **最初の行をヘッダーとして使用**: 選択
    - **圧縮の種類**: なし
7. **[ターゲット]** ステップの **[データセット]** サブステップで、次の設定を選択します。
    - **[ターゲットの種類]** : Azure Data Lake Storage Gen 2
    - **[接続]**: データ レイク ストアへの既存の接続を選びます (これは、ワークスペースを作成したときに自動的に作成されました)。**
8. 接続を選択したら、 **[ターゲット/データセット]** ステップで、次の設定が選択されていることを確認し、 **[次へ]** を選択します。
    - **フォルダー パス**: files/product_data
    - **ファイル名**: products.csv
    - **コピー動作**: なし
    - **最大同時接続数**: *空白のまま*
    - **ブロック サイズ (MB)**: *空白のまま*
9. **[ターゲット]** ステップの **[構成]** サブステップの **[ファイル形式設定]** ページで、次のプロパティが選択されていることを確認します。 次に、**[次へ]** を選択します。
    - **ファイル形式**: DelimitedText
    - **列区切り記号**: コンマ (,)
    - **行区切り記号**: 改行 (\n)
    - **ヘッダーをファイルに追加**: 選択
    - **圧縮の種類**: なし
    - **ファイルごとの最大行数**: *空白のまま*
    - **ファイル名のプレフィックス**: *空白のまま*
10. **[設定]** ステップで、次の設定を入力し、**[次へ >]** をクリックします。
    - **タスク名**: 製品のコピー
    - **タスクの説明**: 製品データのコピー
    - **フォールト トレランス**: *空白のまま*
    - **ログを有効にする**: 選択<u>解除</u>
    - **ステージングを有効にする**: 選択<u>解除</u>
11. **[確認と完了]** ステップの **[確認]** サブステップで、概要を読み、**[次へ >]** をクリックします。
12. **[デプロイ]** ステップで、パイプラインがデプロイされるまで待ち、**[完了]** をクリックします。
13. Synapse Studio で、 **[監視]** ページを選択し、 **[パイプラインの実行]** タブで、 **[製品のコピー]** パイプラインが完了して **[成功]** の状態になるまで待ちます ([パイプラインの実行] ページの **[&#8635; 最新の情報に更新]** ボタンを使用して状態を更新できます)。
14. **[統合]** ページを表示し、 **[製品のコピー]** という名前のパイプラインが含まれていることを確認します。

### 取り込んだデータを表示する

1. **[データ]** ページで、 **[リンク済み]** タブを選択し、Synapse ワークスペースの **files** ファイル ストレージが表示されるまで、**synapse*xxxxxxx* (Primary) datalake** コンテナー階層を展開します。 次に示すように、ファイル ストレージを選択して、**products.csv** という名前のファイルを含む **product_data** という名前のフォルダーが、この場所にコピーされていることを確認します。

    ![Synapse ワークスペースのファイル ストレージで Azure Data Lake Storage 階層が展開された Synapse Studio を示す画像](./images/product_files.png)

2. **products.csv** データ ファイルを右クリックし、 **[プレビュー]** を選択して取り込まれたデータを表示します。 その後、プレビューを閉じます。

## サーバーレス SQL プールを使用してデータを分析する

ワークスペースにデータを取り込んだので、Synapse Analytics を使用してクエリと分析を行うことができます。 データのクエリを実行する最も一般的な方法の 1 つは、SQL を使用することです。Synapse Analytics では、サーバーレス SQL プールを使用して、データ レイク内のデータに対して SQL コードを実行できます。

1. Synapse Studio で、Synapse ワークスペースのファイル ストレージ の **products.csv** ファイルを右クリックし、**[新しい SQL スクリプト]** をポイントして、**[上位 100 行を選択する]** を選択します。
2. **[SQL スクリプト 1]** ペインが開いたら、生成された SQL コードを確認します。これは次のようになります。

    ```SQL
    -- This is auto-generated code
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/product_data/products.csv',
            FORMAT = 'CSV',
            PARSER_VERSION='2.0'
        ) AS [result]
    ```

    このコードでは、インポートしたテキストファイルから行セットを開き、最初の 100 行のデータを取得します。

3. **[接続先]** ボックスの一覧で、**[組み込み]** が選択されていることを確認します。これは、ワークスペースで作成された組み込みの SQL プールを表します。
4. ツールバーの **[&#9655; 実行]** ボタンを使用して SQL コードを実行し、結果を確認します。これは次のようになります。

    | C1 | C2 | C3 | C4 |
    | -- | -- | -- | -- |
    | ProductID | ProductName | カテゴリ | ListPrice |
    | 771 | Mountain-100 Silver, 38 | マウンテン バイク | 3399.9900 |
    | 772 | Mountain-100 Silver, 42 | マウンテン バイク | 3399.9900 |
    | ... | ... | ... | ... |

5. 結果は、C1、C2、C3、C4 の名前の 4 つの列で構成されています。結果の最初の行には、データ フィールドの名前が含まれています。 この問題を解決するには、次に示すように OPENROWSET 関数に HEADER_ROW = TRUE パラメーターを追加し (*datalakexxxxxxx* はデータ レイク ストレージ アカウントの名前に置き換えます)、クエリを再実行します。

    ```SQL
    SELECT
        TOP 100 *
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/product_data/products.csv',
            FORMAT = 'CSV',
            PARSER_VERSION='2.0',
            HEADER_ROW = TRUE
        ) AS [result]
    ```

    結果は次のようになります。

    | ProductID | ProductName | カテゴリ | ListPrice |
    | -- | -- | -- | -- |
    | 771 | Mountain-100 Silver, 38 | マウンテン バイク | 3399.9900 |
    | 772 | Mountain-100 Silver, 42 | マウンテン バイク | 3399.9900 |
    | ... | ... | ... | ... |

6. クエリを次のように変更します (*datalakexxxxxxx* はデータ レイク ストレージ アカウントの名前に置き換えます)。

    ```SQL
    SELECT
        Category, COUNT(*) AS ProductCount
    FROM
        OPENROWSET(
            BULK 'https://datalakexxxxxxx.dfs.core.windows.net/files/product_data/products.csv',
            FORMAT = 'CSV',
            PARSER_VERSION='2.0',
            HEADER_ROW = TRUE
        ) AS [result]
    GROUP BY Category;
    ```

7. 次のように、変更されたクエリを実行します。これにより、各カテゴリの製品数を含む結果セットが返されます。

    | カテゴリ | ProductCount |
    | -- | -- |
    | Bib-Shorts | 3 |
    | バイク ラック | 1 |
    | ... | ... |

8. **[SQL スクリプト 1]** の **[プロパティ]** ペインで、**[名前]** を "**カテゴリ別の製品数**" に変更します。 次に、ツールバーの **[公開]** を選択して、スクリプトを保存します。

9. **[カテゴリ別の製品数]** スクリプト ペインを閉じます。

10. Synapse Studio で、**[開発]** ページを選択し、公開した **[カテゴリ別の製品数]** SQL スクリプトが保存されていることを確認します。

11. **[カテゴリ別の製品数]** SQL スクリプトを選択し、もう一度開きます。 次に、スクリプトが **[組み込み]** SQL プールに接続されていることを確認し、それを実行して製品数を取得します。

12. **[結果]** ペインで、**[グラフ]** ビューを選択し、グラフの次の設定を選択します。
    - **グラフの種類**: 列
    - **カテゴリ列**: カテゴリ
    - **凡例 (系列) 列**: productcount
    - **凡例の位置**: 下中央
    - **凡例 (系列) ラベル**: *空白のまま*
    - **凡例 (系列) の最小値**: *空白のまま*
    - **凡例 (系列) の最大値**: *空白のまま*
    - **カテゴリ ラベル**: *空白のまま*

    結果のグラフは次のようになります。

    ![製品数グラフ ビューを示す画像](./images/column-chart.png)

## Spark プールを使用してデータを分析する

SQL は構造化データセットでクエリを実行するための共通言語ですが、多くのデータ アナリストは、Python など、分析用のデータの探索と準備に役立つ言語を見つけています。 Azure Synapse Analytics では、*Spark プール*で Python (およびその他の) コードを実行できます。これは Apache Spark に基づく分散データ処理エンジンを使用します。

1. Synapse Studio で、先ほど開いた、**products.csv** ファイルを含む **[files]** タブがもう開いていない場合は、 **[データ]** ページで **product_data** フォルダーを参照します。 次に、**products.csv** を右クリックし、**[新しいノートブック]** をポイントして、**[データ フレームに読み込む]** を選択します。
2. **[ノートブック 1]** ペインが表示されたら、 **[アタッチ先]** ボックスの一覧で、**sparkxxxxxxx** Spark プールを選択し、 **[言語]** が **[PySpark (Python)]** に設定されていることを確認します。
3. ノートブックの最初の (および唯一の) セルのコードを確認します。次のようになります。

    ```Python
    %%pyspark
    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/product_data/products.csv', format='csv'
    ## If header exists uncomment line below
    ##, header=True
    )
    display(df.limit(10))
    ```

4. コード セルの左側にある **&#9655;** アイコンを使用して実行し、結果が表示されるまで待ちます。 ノートブックで初めてセルを実行すると、Spark プールが開始されます。結果が返されるまでに 1 分ほどかかることがあります。
5. 最終的には、セルの下に結果が表示され、次のようになります。

    | _c0_ | _c1_ | _c2_ | _c3_ |
    | -- | -- | -- | -- |
    | ProductID | ProductName | カテゴリ | ListPrice |
    | 771 | Mountain-100 Silver, 38 | マウンテン バイク | 3399.9900 |
    | 772 | Mountain-100 Silver, 42 | マウンテン バイク | 3399.9900 |
    | ... | ... | ... | ... |

6. *,header=True* 行のコメントを解除します (products.csv のファイルには最初の行に列ヘッダーがあるため)。コードは次のようになります。

    ```Python
    %%pyspark
    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/product_data/products.csv', format='csv'
    ## If header exists uncomment line below
    , header=True
    )
    display(df.limit(10))
    ```

7. セルを再実行し、結果が次のようになっていることを確認します。

    | ProductID | ProductName | カテゴリ | ListPrice |
    | -- | -- | -- | -- |
    | 771 | Mountain-100 Silver, 38 | マウンテン バイク | 3399.9900 |
    | 772 | Mountain-100 Silver, 42 | マウンテン バイク | 3399.9900 |
    | ... | ... | ... | ... |

    Spark プールは既に開始されているので、セルの再実行にかかる時間は短くなります。

8. 結果の下にある **[&#65291; コード]** アイコンを使用して、ノートブックに新しいコード セルを追加します。
9. 新しい空のコード セルに、次のコードを追加します。

    ```Python
    df_counts = df.groupby(df.Category).count()
    display(df_counts)
    ```

10. **&#9655;** アイコンをクリックして新しいコード セルを実行し、結果を確認します。これは次のようになります。

    | カテゴリ | count |
    | -- | -- |
    | ヘッドセット | 3 |
    | ホイール | 14 |
    | ... | ... |

11. セルの結果出力で、**[グラフ]** ビューを選択します。 結果のグラフは次のようになります。

    ![カテゴリ数グラフ ビューを示す画像](./images/bar-chart.png)

12. **[プロパティ]** ページがまだ表示されていない場合は、ツール バーの右端にある **[プロパティ]** ボタン ( **&#128463;<sub>*</sub>** のような外観) を選択して表示します。 次に、 **[プロパティ]** ペインでノートブック名を「**製品の探索**」に変更し、ツール バーの **[発行]** ボタンを使用して保存します。

13. ノートブック ペインを閉じ、メッセージが表示されたら Spark セッションを停止します。 次に、 **[開発]** ページを表示して、ノートブックが保存されていることを確認します。

## 専用 SQL プールを使用してデータ ウェアハウスにクエリを実行する

ここまでは、データ レイク内のファイル ベースのデータを探索および処理するための手法をいくつか見てきました。 多くの場合、エンタープライズ分析ソリューションでデータ レイクを使用して、非構造化データを格納および準備し、ビジネス インテリジェンス (BI) ワークロードをサポートするためにリレーショナル データ ウェアハウスに読み込むことができます。 Azure Synapse Analytics では、これらのデータ ウェアハウスを専用 SQL プールに実装できます。

1. Synapse Studio の **[管理]** ページの **[SQL プール]** セクションで、**sql*xxxxxxx*** 専用 SQL プールの行を選択し、 **&#9655;** アイコンを使用して再開します。
2. SQL プールが開始されるまで待ちます。 これには数分かかることがあります。 **&#8635; [更新]** ボタンを使用して、その状態を定期的に確認してください。 準備ができると、状態が **[オンライン]** と表示されます。
3. SQL プールが開始されたら、 **[データ]** ページを選択します。 **[ワークスペース]** タブで **[SQL データベース]** を展開し、**sql*xxxxxxx*** が一覧に表示されていることを確認します (必要に応じて、ページの左上にある **&#8635;** アイコンを使用して表示を更新します)。
4. **sql*xxxxxxx*** データベースとその **Tables** フォルダーを展開し、**FactInternetSales** テーブルの **[...]** メニューで **[新しい SQL スクリプト]** をポイントし、 **[上位 100 行を選択]** を選択します。
5. クエリの結果を確認します。これには、テーブル内の最初の 100 件の販売トランザクションが表示されます。 このデータはセットアップ スクリプトによってデータベースに読み込まれたもので、専用 SQL プールに関連付けられたデータベースに永続的に保存されています。
6. SQL クエリを次のコードに置き換えます。

    ```sql
    SELECT d.CalendarYear, d.MonthNumberOfYear, d.EnglishMonthName,
           p.EnglishProductName AS Product, SUM(o.OrderQuantity) AS UnitsSold
    FROM dbo.FactInternetSales AS o
    JOIN dbo.DimDate AS d ON o.OrderDateKey = d.DateKey
    JOIN dbo.DimProduct AS p ON o.ProductKey = p.ProductKey
    GROUP BY d.CalendarYear, d.MonthNumberOfYear, d.EnglishMonthName, p.EnglishProductName
    ORDER BY d.MonthNumberOfYear
    ```

7. **[&#9655; 実行]** ボタンを使用して変更されたクエリを実行します。これにより、年と月別に販売された各製品の数量が返されます。
8. **[プロパティ]** ページがまだ表示されていない場合は、ツール バーの右端にある **[プロパティ]** ボタン ( **&#128463;<sub>*</sub>** のような外観) を選択して表示します。 次に、 **[プロパティ]** ペインで、クエリ名を **[製品売上の集計]** に変更し、ツール バーの **[発行]** ボタンを使用して保存します。

9. クエリ ペインを閉じ、 **[開発]** ページを表示して、SQL スクリプトが保存されていることを確認します。

10. **[管理]** ページで、**sql*xxxxxxx*** 専用 SQL プールの行を選択し、&#10074;&#10074; アイコンを使用して一時停止します。

<!--- ## Explore data with a Data Explorer pool

Azure Synapse Data Explorer provides a runtime that you can use to store and query data by using Kusto Query Language (KQL). Kusto is optimized for data that includes a time series component, such as realtime data from log files or IoT devices.

### Create a Data Explorer database and ingest data into a table

1. In Synapse Studio, on the **Manage** page, in the **Data Explorer pools** section, select the **adx*xxxxxxx*** pool row and then use its **&#9655;** icon to resume it.
2. Wait for the pool to start. It can take some time. Use the **&#8635; Refresh** button to check its status periodically. The status will show as **online** when it is ready.
3. When the Data Explorer pool has started, view the **Data** page; and on the **Workspace** tab, expand **Data Explorer Databases** and verify that **adx*xxxxxxx*** is listed (use **&#8635;** icon at the top-left of the page to refresh the view if necessary)
4. In the **Data** pane, use the **&#65291;** icon to create a new **Data Explorer database** in the **adx*xxxxxxx*** pool with the name **sales-data**.
5. In Synapse Studio, wait for the database to be created (a notification will be displayed).
6. Switch to the **Develop** page, and in the **+** menu, add a KQL script. Then, when the script pane opens, in the **Connect to** list, select your **adx*xxxxxxx*** pool, and in the **Database** list, select **sales-data**.
7. In the new script, add the following code:

    ```kusto
    .create table sales (
        SalesOrderNumber: string,
        SalesOrderLineItem: int,
        OrderDate: datetime,
        CustomerName: string,
        EmailAddress: string,
        Item: string,
        Quantity: int,
        UnitPrice: real,
        TaxAmount: real)
    ```

8. On the toolbar, use the **&#9655; Run** button to run the selected code, which creates a table named **sales** in the **sales-data** database you created previously.
9. After the code has run successfully, replace it with the following code, which loads data into the table:

    ```kusto
    .ingest into table sales 'https://raw.githubusercontent.com/microsoftlearning/dp-203-azure-data-engineer/master/Allfiles/labs/01/files/sales.csv' 
    with (ignoreFirstRecord = true)
    ```

10. Run the new code to ingest the data.

> **Note**: In this example, you imported a very small amount of batch data from a file, which is fine for the purposes of this exercise. In reality, you can use Data Explorer to analyze much larger volumes of data; including realtime data from a streaming source such as Azure Event Hubs.

### Use Kusto query language to query the table

1. Switch back to the **Data** page and in the **...** menu for the **sales-data** database, select **Refresh**.
2. Expand the **sales-data** database's **Tables** folder. Then in the **...** menu for the **sales** table, select **New KQL script** > **Take 1000 rows**.
3. Review the generated query and its results. The query should contain the following code:

    ```kusto
    sales
    | take 1000
    ```

    The results of the query contain the first 1000 rows of data.

4. Modify the query as follows:

    ```kusto
    sales
    | where Item == 'Road-250 Black, 48'
    ```

5. Use the **&#9655; Run** button to run the query. Then review the results, which should contain only the rows for sales orders for the *Road-250 Black, 48* product.

6. Modify the query as follows:

    ```kusto
    sales
    | where Item == 'Road-250 Black, 48'
    | where datetime_part('year', OrderDate) > 2020
    ```

7. Run the query and review the results, which should contain only sales orders for *Road-250 Black, 48* made after 2020.

8. Modify the query as follows:

    ```kusto
    sales
    | where OrderDate between (datetime(2020-01-01 00:00:00) .. datetime(2020-12-31 23:59:59))
    | summarize TotalNetRevenue = sum(UnitPrice) by Item
    | sort by Item asc
    ```

9. Run the query and review the results, which should contain the total net revenue for each product between January 1st and December 31st 2020 in ascending order of product name.

10. If it is not already visible, show the **Properties** page by selecting the **Properties** button (which looks similar to **&#128463;<sub>*</sub>**) on the right end of the toolbar. Then in the **Properties** pane, change the query name to **Explore sales data** and use the **Publish** button on the toolbar to save it.

11. Close the query pane, and then view the **Develop** page to verify that the KQL script has been saved.

12. On the **Manage** page, select the **adx*xxxxxxx*** Data Explorer pool row and use its &#10074;&#10074; icon to pause it. --->

## Azure リソースを削除する

Azure Synapse Analytics の探索が終了したので、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースの **dp203-*xxxxxxx*** リソース グループ (管理対象リソース グループでななく) を選択し、そこに Synapse ワークスペース、ストレージ アカウント、ワークスペースの SQL プール、Data Explorer プール、Spark プールが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
