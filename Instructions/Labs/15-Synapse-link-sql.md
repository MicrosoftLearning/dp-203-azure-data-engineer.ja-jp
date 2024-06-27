---
lab:
  title: Azure Synapse Link for SQL を使用する
  ilt-use: Suggested demo
---

# Azure Synapse Link for SQL を使用する

Azure Synapse Link for SQL を使用すると、SQL Server または Azure SQL Database のトランザクション データベースを、Azure Synapse Analytics の専用 SQL プールと自動的に同期できます。 この同期を使用すると、ソースの運用データベースでクエリのオーバーヘッドを発生させることなく、Synapse Analytics で低遅延の分析ワークロードを実行できます。

この演習の所要時間は約 **35** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure リソースをプロビジョニングする

この演習では、Azure SQL Database リソースから Azure Synapse Analytics ワークスペースにデータを同期します。 まず、スクリプトを使用して Azure サブスクリプションでこれらのリソースをプロビジョニングします。

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
    cd dp-203/Allfiles/labs/15
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure SQL Database の適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 15 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Synapse Analytics ドキュメントの「[Azure Synapse Link for SQL とは](https://docs.microsoft.com/azure/synapse-analytics/synapse-link/sql-synapse-link-overview)」の記事を確認してください。

## Azure SQL Database を構成する

Azure SQL Database 用に Azure Synapse Link を設定する前に、Azure SQL Database サーバーに必要な構成設定が適用されていることを確認する必要があります。

1. [Azure portal](https://portal.azure.com) で、セットアップ スクリプトによって作成された **dp203-*xxxxxxx*** リソース グループを参照し、Azure SQL サーバー **sqldb*xxxxxxxx*** を選択します。

    > **注**: Azure SQL サーバー リソース **sqldb*xxxxxxxx***) と Azure Synapse Analytics 専用 SQL プール (** sql*xxxxxxxx***) を混同しないように注意してください。

2. Azure SQL Server リソースのページで、左側のペインの **[セキュリティ]** セクション (下部付近) にある **[ID]** を選びます。 次に、 **[システム割り当てマネージド ID]** で、 **[状態]** オプションを **[オン]** に設定します。 次に、 **[&#128427; 保存]** アイコンを使用して構成の変更を保存します。

    ![Azure portal の [Azure SQL サーバー ID] ページのスクリーンショット。](./images/sqldb-identity.png)

3. 左側のペインの **[セキュリティ]** セクションで、 **[ネットワーク]** を選択します。 次に、 **[ファイアウォール規則]** で **[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** の例外を選択します。

4. **[&#65291; ファイアウォール規則の追加]** ボタンを使用して、次の設定で新しいファイアウォール規則を追加します。

    | 規則名 | 開始 IP | 終了 IP |
    | -- | -- | -- |
    | AllClients | 0.0.0.0 | 255.255.255.255 |

    > **注**: この規則では、インターネットに接続されている任意のコンピューターからサーバーにアクセスできます。 演習を簡略化するためにこれを有効にしていますが、運用環境のシナリオでは、データベースを使用する必要があるネットワーク アドレスのみにアクセスを制限する必要があります。

5. **[保存]** ボタンを使用して構成の変更を保存します。

    ![Azure portal の Azure SQL サーバー ネットワーク ページのスクリーンショット。](./images/sqldb-network.png)

## トランザクション データベースを調べる

この Azure SQL サーバーで **AdventureWorksLT** という名前のサンプル データベースをホストします。 このデータベースは、運用アプリケーション データ用に使用するトランザクション データベースを表します。

1. Azure SQL サーバーの **[概要]** ページの下部にある **AdventureWorksLT** データベースを選択します。
2. **AdventureWorksLT** データベース ページで、 **[クエリ エディター]** タブを選択し、次の資格情報で SQL Server 認証を使用してログインします。
    - **ログイン** SQLUser
    - **パスワード**: "セットアップ スクリプトの実行時に指定したパスワード。"**
3. クエリ エディターが開いたら、 **[テーブル]** ノードを展開し、データベース内のテーブルの一覧を表示します。 **SalesLT** スキーマ (**SalesLT.Customer** など) にテーブルが含まれていることに注意してください。

## Azure Synapse Link を構成する

これで、Synapse Analytics ワークスペースに Azure Synapse Link for SQL を構成する準備ができました。

### 専用 SQL プールを起動する

1. Azure portal で、Azure SQL データベースのクエリ エディターを閉じて (変更を破棄します)、**dp203-*xxxxxxx*** リソース グループのページに戻ります。
2. Synapse ワークスペース **synapse*xxxxxxx*** を開き、その **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。メッセージが表示された場合はサインインします。
3. Synapse Studio の左側にある **&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。これにより、さまざまなページが Synapse Studio 内に表示されます。
4. **[管理]** ページの **[SQL プール]** タブで、専用 SQL プール **sql*xxxxxxx*** の行を選択し、 **&#9655;** アイコンを使用して起動します。メッセージが表示されたら、再開することを確認します。
5. SQL プールが再開されるまで待ちます。 これには数分かかることがあります。 **[&#8635; 最新の情報に更新]** ボタンを使用して、その状態を定期的に確認できます。 準備ができると、状態が **[オンライン]** と表示されます。

### ターゲット スキーマを作成する

1. Synapse Studio の **[データ]** ページの **[ワークスペース]** タブで、 **[SQL データベース]** を展開し、**sql*xxxxxxx*** データベースを選択します。
2. **sql*xxxxxxx*** データベースの **[...]** メニューで、 **[新しい SQL スクリプト]**  >  **[空のスクリプト]** の順に選択します。
3. **[SQL スクリプト 1]** ペインで、次の SQL コードを入力し、 **[&#9655; 実行]** ボタンを使用して実行します。

    ```sql
    CREATE SCHEMA SalesLT;
    GO
    ```

4. クエリが正常に完了するまで待ちます。 このコードで、専用 SQL プールのデータベース用に **SalesLT** という名前のスキーマを作成し、その名前のスキーマ内のテーブルを Azure SQL データベースから同期できるようにします。

### リンク接続を作成する

1. Synapse Studio の **[統合]** ページの **&#65291;** ドロップダウン メニューで、 **[リンク接続]** を選択します。 次の設定を使用して、新しいリンク接続を作成します。
    - **ソースの種類**: Azure SQL データベース
    - **ソースのリンク サービス**: 次の設定を使用して、新しいリンク サービスを追加します (新しいタブが開きます)。
        - **名前**: SqlAdventureWorksLT
        - **説明**: AdventureWorksLT データベースへの接続
        - **統合ランタイム経由で接続する**: AutoResolveIntegrationRuntime
        - **[バージョン]**: レガシ
        - **接続文字列**: 選択済み
        - **Azure サブスクリプションから**: 選択済み
        - **Azure サブスクリプション**: "使用する Azure サブスクリプションを選択します"**
        - **サーバー名**: "**sqldbxxxxxxx** Azure SQL サーバーを選択します"**
        - **データベース名**: AdventureWorksLT
        - **認証の種類**: SQL 認証
        - **ユーザー名**: SQLUser
        - **パスワード**: "セットアップ スクリプトの実行時に設定したパスワード"**

        "続行する前に、 **[接続テスト]** オプションを使用して接続設定が正しいことを確認します。その後に **[作成]** をクリックします。"**

    - **ソース テーブル**: 次のテーブルを選択します。
        - **SalesLT.Customer**
        - **SalesLT.Product**
        - **SalesLT.SalesOrderDetail**
        - **SalesLT.SalesOrderHeader**

        "続けて次の設定を構成します。"**

    > **注**: 一部のターゲット テーブルでは、カスタム データ型が使用されていることが原因で、またはソース テーブル内のデータが "クラスター化列ストア インデックス" の既定の構造の種類と互換性がないために、エラーが表示されます。**

    - **ターゲット プール**: "**sqlxxxxxxx** 専用 SQL プールを選択します"**

        "続けて次の設定を構成します。"**

    - **リンク接続名**: sql-adventureworkslt-conn
    - **コア数**: 4 (+ 4 ドライバー コア)

2. 作成された **sql-adventureworkslt-conn** ページで、作成されたテーブル マッピングを表示します。 **[プロパティ]** ボタン ( **&#128463;<sub>*</sub>** のような外観) を使用すると、 **[プロパティ]** ペインを非表示にして、すべてを見やすくすることができます。 

3. テーブル マッピングでこれらの構造の種類を次のように変更します。

    | ソース テーブル | ターゲット テーブル | 分散の種類 | ディストリビューション列 | 構造の種類 |
    |--|--|--|--|--|
    | SalesLT.Customer **&#8594;** | \[SalesLT] . \[Customer] | ラウンド ロビン | - | クラスター化列ストア インデックス |
    | SalesLT.Product **&#8594;** | \[SalesLT] . \[Product] | ラウンド ロビン | - | ヒープ |
    | SalesLT.SalesOrderDetail **&#8594;** | \[SalesLT] . \[SalesOrderDetail] | ラウンド ロビン | - | クラスター化列ストア インデックス |
    | SalesLT.SalesOrderHeader **&#8594;** | \[SalesLT] . \[SalesOrderHeader] | ラウンド ロビン | - | ヒープ |

4. 作成された **sql-adventureworkslt-conn** ページの上部にある **[&#9655; 開始]** ボタンを使用して同期を開始します。 メッセージが表示されたら、 **[OK]** を選択することで、リンク接続を発行して開始します。
5. 接続を開始した後、 **[監視]** ページで **[リンク接続]** タブを表示し、**sql-adventureworkslt-conn** 接続を選択します。 **[&#8635; 最新の情報に更新]** ボタンを使用して、状態を定期的に確認できます。 最初のスナップショット コピー プロセスが完了し、レプリケートが開始されるまで数分かかる場合があります。その後、ソース データベース テーブル内のすべての変更が同期先テーブルで自動的に再生されます。

### レプリケートされたデータを表示する

1. テーブルの状態が **[実行中]** に変わったら、 **[データ]** ページを選択し、右上にある **&#8635;** アイコンを使用して表示を更新します。
2. **[ワークスペース]** タブで、 **[SQL データベース]** 、**sql*xxxxxxx*** データベース、そしてその **[テーブル]** フォルダーを展開して、レプリケートされたテーブルを表示します。
3. **sql*xxxxxxx*** データベースの **[...]** メニューで、 **[新しい SQL スクリプト]**  >  **[空のスクリプト]** の順に選択します。 次に、新しいスクリプト ページで、次の SQL コードを入力します。

    ```sql
    SELECT  oh.SalesOrderID, oh.OrderDate,
            p.ProductNumber, p.Color, p.Size,
            c.EmailAddress AS CustomerEmail,
            od.OrderQty, od.UnitPrice
    FROM SalesLT.SalesOrderHeader AS oh
    JOIN SalesLT.SalesOrderDetail AS od 
        ON oh.SalesOrderID = od.SalesOrderID
    JOIN  SalesLT.Product AS p 
        ON od.ProductID = p.ProductID
    JOIN SalesLT.Customer as c
        ON oh.CustomerID = c.CustomerID
    ORDER BY oh.SalesOrderID;
    ```

4. **[&#9655; 実行]** ボタンを使用してスクリプトを実行し、結果を表示します。 クエリは、ソース データベースではなく、専用 SQL プール内のレプリケートされたテーブルに対して実行されるため、ビジネス アプリケーションに影響を与えることなく分析クエリを実行できます。
5. 完了したら、 **[管理]** ページで、専用 SQL プール **sql*xxxxxxx*** を一時停止します。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. この演習の開始時にセットアップ スクリプトによって作成された **dp203-*xxxxxxx*** リソース グループを選択します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、リソース グループとそれに含まれていたリソースが削除されます。
