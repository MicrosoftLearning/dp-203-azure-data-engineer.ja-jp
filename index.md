---
title: オンラインでホスティングされている手順
permalink: index.html
layout: home
---

# Azure Data Engineering の演習

次の演習では、[Microsoft Certified: Azure Data Engineer Associate](https://learn.microsoft.com/certifications/azure-data-engineer/) 認定をサポートする Microsoft Learn のトレーニング モジュールをサポートしています。

これらの演習を完了するには、管理用のアクセスが付与される [Microsoft Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。 一部の演習では、[Microsoft Power BI テナント](https://learn.microsoft.com/power-bi/fundamentals/service-self-service-signup-for-power-bi)へのアクセスが必要な場合もあります。

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %}
| 演習 | ILT では、これは... |
| --- | --- |
{% for activity in labs %}| [{{ activity.lab.title }}{% if activity.lab.type %} - {{ activity.lab.type }}{% endif %}]({{ site.github.url }}{{ activity.url }}) | {{ activity.lab.ilt-use }} |
{% endfor %}
