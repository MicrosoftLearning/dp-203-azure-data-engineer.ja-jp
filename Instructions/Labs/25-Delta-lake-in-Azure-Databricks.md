---
lab:
  title: Azure Databricks で Delta Lake を使用する
  ilt-use: Optional demo
---

# Azure Databricks で Delta Lake を使用する

Delta Lake は、データ レイクの上に Spark 用のトランザクション データ ストレージ レイヤーを構築するためのオープンソース プロジェクトです。 Delta Lake では、バッチ データ操作とストリーミング データ操作の両方にリレーショナル セマンティクスのサポートが追加され、Apache Spark を使用して、データ レイク内の基になるファイルに基づくテーブル内のデータを処理しクエリを実行できる *Lakehouse* アーキテクチャを作成できます。

この演習の所要時間は約 **40** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Databricks ワークスペースをプロビジョニングする

この演習では、スクリプトを使用して新しい Azure Databricks ワークスペースをプロビジョニングします。

> **ヒント**: *Standard* または*試用版*の Azure Databricks ワークスペースが既にある場合は、この手順をスキップできます。

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

5. リポジトリが複製されたら、次のコマンドを入力してこのラボ用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```
    cd dp-203/Allfiles/labs/25
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。

7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの [Delta テクノロジの概要](https://learn.microsoft.com/azure/databricks/introduction/delta-comparison)に関する記事を参照してください。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS ランタイム バージョンのクラスターが既にある場合は、それを使用してこの演習を完了し、この手順をスキップできます。

1. Azure portal で、スクリプトによって作成された **dp203-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) を参照します。
1. Azure Databricks Service リソース (セットアップ スクリプトを使用して作成した場合は **databricks*xxxxxxx*** という名前) を選択します。
1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

1. 次に、Azure Databricks ワークスペース ポータルを表示し、左側のサイド バーに、実行できるさまざまなタスクのアイコンが含まれていることに注目します。

1. **(+) [新規]** タスクを選択し、 **[クラスター]** を選択します。
1. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
    - **クラスター名**: "ユーザー名の" クラスター (既定のクラスター名)**
    - **クラスター モード**: 単一ノード
    - **アクセス モード**: 単一ユーザー (*自分のユーザー アカウントを選択*)
    - **Databricks Runtime バージョン**: 13.3 LTS (Spark 3.4.1, Scala 2.12)
    - **Photon Acceleration を使用する**: 選択済み
    - **ノードの種類**: Standard_DS3_v2
    - **アクティビティが** *30* **分ない場合は終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./setup.ps1 eastus`

## ノートブックを使って Delta Lake を調べる

この演習では、ノートブック内のコードを使って、Azure Databricks の Delta Lake を調べます。

1. ワークスペースの Azure Databricks ワークスペース ポータルで、左側のサイドバーで **[ワークスペース]** を選択します。 次に、 **&#8962; [ホーム]** フォルダーを選びます。
1. ページ上部の、ユーザー名の横にある **&#8942;** メニューで **[インポート]** を選びます。 次に、 **[インポート]** ダイアログ ボックスで、 **[URL]** を選び、`https://github.com/MicrosoftLearning/dp-203-azure-data-engineer/raw/master/Allfiles/labs/25/Delta-Lake.ipynb` からノートブックをインポートします
1. ノートブックをクラスターに接続し、含まれている指示に従います。含まれているセルを実行して、Delta Lake 機能を調べます。

## Azure Databricks リソースを削除する

これで Azure Databricks の Delta Lake を調べ終わったので、不要な Azure のコストを避け、サブスクリプションの容量を解放するために、作成したリソースを削除する必要があります。

1. Azure Databricks ワークスペースのブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. **dp203-xxxxxxx** リソース グループ (管理対象リソース グループではありません) を選択し、お使いの Azure Databricks ワークスペースが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名「**dp203-xxxxxxx**」を入力してそれを削除することを確認し、 **[削除]** を選びます。

    数分後に、リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
