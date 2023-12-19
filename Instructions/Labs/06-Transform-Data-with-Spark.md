---
lab:
  title: Synapse Analytics で Spark を使用してデータを変換する
  ilt-use: Lab
---

# Synapse Analytics で Spark を使用してデータを変換する

データ "エンジニア" は、データの形式や構造を別のものに変換する "抽出、変換、読み込み (ETL)" または "抽出、読み込み、変換 (ELT)" アクティビティを実行するための優先ツールの 1 つとして、Spark ノートブックを使うことがよくあります。** ** **

この演習では、Azure Synapse Analytics で Spark ノートブックを使って、ファイル内のデータを変換します。

この演習の所要時間は約 **30** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Synapse Analytics ワークスペースをプロビジョニングする

データ レイク ストレージと Spark プールにアクセスできる Azure Synapse Analytics ワークスペースが必要です。

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
    cd dp-203/Allfiles/labs/06
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 10 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Synapse Analytics ドキュメントの「[Azure Synapse Analytics の Apache Spark の主要な概念](https://learn.microsoft.com/azure/synapse-analytics/spark/apache-spark-concepts)」の記事を確認してください。

## Spark ノートブックを使用してデータを変換する

1. デプロイ スクリプトが完了したら、Azure portal で、作成された **dp203-*xxxxxxx*** リソース グループに移動し、このリソース グループに Synapse ワークスペース、データ レイク用のストレージ アカウント、Apache Spark プールが含まれていることを確認します。
2. Synapse ワークスペースを選び、その **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選んで、新しいブラウザー タブで Synapse Studio を開きます。メッセージが表示されたらサインインします。
3. Synapse Studio の左側にある **&rsaquo;&rsaquo;** アイコンを使用してメニューを展開します。これにより、リソースの管理とデータ分析タスクの実行に使用するさまざまなページが Synapse Studio 内に表示されます。
4. **[管理]** ページで、 **[Apache Spark pools]** タブを選択し、**spark*xxxxxxx*** のような名前の Spark プールがワークスペース内にプロビジョニングされていることに注意してください。
5. **[データ]** ページで **[リンク]** タブを表示して、**synapse*xxxxxxx* (Primary - datalake*xxxxxxx*)** のような名前の Azure Data Lake Storage Gen2 ストレージ アカウントへのリンクがワークスペースに含まれていることを確認します。
6. ストレージ アカウントを展開して、そこに **files (Primary)** という名前のファイル システム コンテナーが含まれていることを確認します。
7. この **files** コンテナーを選択し、そこに **data** と **synapse** という名前のフォルダーが含まれていることを確認します。 synapse フォルダーは Azure Synapse によって使用され、**data** フォルダーにはこれからクエリを実行するデータ ファイルが含まれています。
8. **data** フォルダーを開き、3 年間の売上データの .csv ファイルが含まれていることを確認します。
9. いずれかのファイルを右クリックし、 **[プレビュー]** を選択して、そこに含まれているデータを表示します。 このファイルにはヘッダー行が含まれているため、列ヘッダーを表示するには、オプションを選択できることに注意してください。
10. プレビューを閉じます。 次に **Spark Transform.ipynb** を [https://raw.githubusercontent.com/MicrosoftLearning/dp-203-azure-data-engineer/master/Allfiles/labs/06/notebooks/Spark%20Transform.ipynb](https://raw.githubusercontent.com/MicrosoftLearning/dp-203-azure-data-engineer/master/Allfiles/labs/06/notebooks/Spark%20Transform.ipynb) からダウンロードします

    > **注**: ***Ctrl + A*** キー、***Ctrl + C*** キーの順に押してこのテキストをコピーし、***Ctrl + V*** を使ってメモ帳などのツールに貼り付け、ファイルを使い、ファイルの種類を***すべてのファイル***にして **Spark Transform.ipynb** という名前で保存します。

11. 次に、 **[開発]** ページで **[ノートブック]** を展開し、[+ インポート] オプションをクリックします

    ![Spark ノートブックのインポート](./image/../images/spark-notebook-import.png)
        
12. 先ほどダウンロードし、**Spark Transfrom.ipynb** という名前で保存したファイルを選びます。
13. ノートブックを **spark*xxxxxxx*** Spark プールにアタッチします。
14. ノートブックの注意事項を確認して、コード セルを実行します。

    > **注**: Spark プールを開始する必要があるため、最初のコード セルの実行には数分かかります。 後続のセルは、もっと速く実行されます。

## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースに対して **dp203-*xxxxxxx*** リソース グループ (管理対象リソース グループ以外) を選択し、そこに Synapse ワークスペース、ストレージ アカウント、ワークスペースの Spark プールが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。
