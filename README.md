# Hatohol Arm Plugin Interface 2.0

## ToDO

Initiation
serverTypeのURL
エラーコード
getMonitoringServerの項目，本当にMandatroyか
情報の情報？
reqTermはサーバー->HAPオンリー？

## 概要

Hatohol Arm Plugin InterfaceはHatoholサーバーとプラグイン間でのデータの送受信を定義するフレームワークです。
通信規格としてAMQP（rabbitmq），データプロトコルとしてJSON-RPC 2.0を採用しています。JSON-RPCの仕様については[公式リファレンス](http://www.jsonrpc.org/specification)をお読みください。

このドキュメントではHatohol Arm Plugin Interface規格に則り,Hatoholサーバーとデータの送受信を行うHatohol Arm Plugin(以下，HAP)の仕様について記載しています。

不明点についてはHatoholコミュニティにお問い合わせください。[hatohol-users@sourceforge.net]

![overview](hapi_overview.png)

## HAP動作全容

```
Hatoholサーバー                                   HAP
    |                                               
    |                                        Turn on HAP         
    |                                              |
    |<----------------Initiation------------------>|
    |                                              |
    |                  (データ同期)                |
    |<-----------getMonitoringServerInfo-----------|
    |           ポーリングタイムなどを取得         |
    |<------------------updateHosts----------------|
    |<------------------updateHostGroups-----------|
    |<------------------updateHostGroupElements----|
    |<------------------updateTriggers-------------|
    |<------------------updateEvents---------------|
    |     接続以前に発生したイベントの同期の有無は |
    |     Hatoholサーバーで設定                    |
    |                                              |
    |<------------------updateVirtualRelations-----|
    |<------------------updateArmInfo--------------|
    |                 (データ同期終了)         |   |
    |                                          |   |
    |                               ポーリング間隔 |
    |            (通常運用，定期更新)          |   |
    |<-----------getMonitoringServerInfo-----------|
    |                                              |
    |<--------------getLastInfo--------------------|
    |<------------------updateHosts----------------|
    |                                              |
    |<--------------getLastInfo--------------------|
    |<------------------updateHostGroups-----------|
    |                                              |
    |<--------------getLastInfo--------------------|
    |<------------------updateHostGroupElements----|
    |                                              |
    |<--------------getLastInfo--------------------|
    |<------------------updateTriggers-------------|
    |                                              |
    |<--------------getLastInfo--------------------|
    |<------------------updateEvents---------------|
    |                                              |
    |<--------------getLastInfo--------------------|
    |<------------------updateVirtualRelations-----|
    |                                              |
    |<------------------updateArmInfo--------------|
    |            (ここまで定期更新)            |   |
    |                               ポーリング間隔 |
    |                                          |   |
 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    |                                              |
    |-----------fetchItems------------------------>|
    |<------------------updateItems----------------|
    |                                              |
    |-----------fetchHistory---------------------->|
    |<------------------sendHistory----------------|
    |                                              |
    |-----------fetchTriggers--------------------->|
    |<------------------updateTriggers-------------|
    |"ALL"オプションを使用し全てのトリガーを送信する|
    |                                              |
    |                                              |
    |--------------reqTerminate------------------->|
    |   HatoholとHAPの接続を終了する               |
    |                                         Turn off HAP
```

## 注意事項

 - 現在，HatoholはUTF-8を標準的な文字コードとして，作成されています。Hatoholサーバーに送信するデータの文字コードにはUTF-8を使用することを推奨します。
 - リクエスト・レスポンスで使用するIDオブジェクトの値には，十分なランダム性を必要とします。
 - JSON-RPCにはバッチリクエストという，複数のリクエストを同時に送信する文法が存在しますが，Hatoholはこの文法を用いたリクエストには対応していません。

## データ型解説

このセクションは，JSON-RPC形式ではなくHatoholが独自に定義しているデータ型について解説します。これらは内部的にはJSON-RPC形式が定義しているデータ型を使用しています。

|名前|JSON型|解説|
|:---|:---------|:---|
|timestamp|string|時刻フォーマットはyyyyMMDDHHmmss.nnnnnnnnnです。小数点以下の時刻については省くことが出来ます。また，小数点以下には9桁までしか値を挿入することはできません。小数点以下を省いた場合，または小数点以下が9桁未満の場合には余った桁部に0が挿入されます。(Ex.100 -> 100.000000000, 100.1234 -> 100.123400000)。|
|boolean|true, false|true or falseを指定し，その値の真偽を示します|

##Initiationについて

応相談

## プロシージャ解説

|プロシージャ名|解説|実装箇所  |タイプ|M/O|
|:-------------|:---|:---------|:-----|:-:|
|[getMonitoringServerInfo](#user-content-getMonitoringServerInfo)|Hatoholサーバーから監視対象の接続情報やポーリング間隔等を取得します|サーバー|method|M|
|[getLastInfo](#user-content-getLastInfo)|Hatoholサーバーから指定した要素の最新情報を取得します|サーバー|method|O|
|[updateItems](#user-content-updateItems)|監視サーバーが監視しているアイテム一覧をHatoholサーバーに送信します|サーバー|method|O|
|[sendHistory](#user-content-sendHistory)|各アイテムが所持しているアイテムのヒストリーをHatoholサーバーに送信します|サーバー|notification|O|
|[updateHosts](#user-content-updateHosts)|監視サーバーが監視しているホスト一覧をHatoholサーバーに送信します|サーバー|method|O|
|[updateHostGroups](#user-content-updateHostGroups)|ホストグループの情報をHatoholサーバーに送信します|サーバー|method|O|
|[updateHostGroupElements](#user-content-updateHostGroupElements)|ホストのホストグループ所属情報をHatoholサーバーに送信します|サーバー|method|O|
|[updateTriggers](#user-content-updateTrigges)|トリガーをHatoholサーバーに送信します<br>送信するトリガーはオプションで指定することが出来ます|サーバー|method|O|
|[updateEvents](#user-content-updateEvents)|アップデートされたイベントをHatoholサーバーに送信します|サーバー|method|O|
|[updateVirtualRelations](#user-content-updateVirtualRelations)|ホスト同士のVM親子関係をHatoholサーバーに送信します|サーバー|method|O|
|[updateArmInfo](#user-content-updateArmInfo)|HAPの接続情報をHatoholサーバーに送信します|サーバー|method|O|
|[fetchItems](#user-content-fetchItems)|Hatoholサーバーがアイテムを要求しているときにHAPに送信されます|プラグイン|method|O|
|[fetchHistory](#user-content-fetchHistory)|Hatoholサーバーがヒストリーを要求しているときにHAPに送信されます|プラグイン|method|O|
|[fetchTriggers](#user-content-fetchTriggers)|Hatoholサーバーが全てのトリガーを要求しているときにHAPに送信されます|プラグイン|method|O|
|[reqTerminate](#user-content-reqTerminate)|HAPとHatoholサーバーとの接続を終了させます|プラグイン|notification|M|

 - 「実装箇所」は各プロシージャを実装する箇所を示しています。
 - 「実装箇所」がサーバーとなっているプロシージャが使用するnumber型オブジェクトの値範囲は0~2147483647です。
 - 各プロシージャ解説にはM/O(Mandatory/Optional)カラムがあります。このカラムがMのプロシージャは実装を省略することができません。
 - M/OがOとなっているプロシージャは実装を省略することが可能です。しかしfetch~~~プロシージャのようにHatoholサーバーからリクエストを受けるプロシージャの実装を省略している場合は，errorオブジェクトで要求を受けたプロシージャを実装していないことを伝えるエラーメッセージを返すよう実装してください。
 - update~~~プロシージャは，送信したデータのデータベース書き込み成否をresultオブジェクトとして受け取ります。結果値については[[一覧](#user-content-updateResult)]をご覧ください。
 - fetch~~~プロシージャで受けたリクエスト受け入れの成否をresultオブジェクトでHatoholサーバーに返します。結果値については[[一覧](#user-content-fetchResult)]をご覧ください。
 - fetch~~~プロシージャはHatoholサーバーからのリクエスト頻度が高い場合は受け入れを省略することができます。

### getMonitoringServerInfo(method)

ポーリング時間毎にHatoholサーバーに自身の監視サーバー情報を問い合わせることを標準的な動作としますが，任意のタイミングで問い合わせることもできます。

***params***

getMonitoringServerInfoメソッドには引数が存在しません。nullオブジェクトとしてparamsを送信してください。

```
{"jsonrpc":"2.0", "method":"getMonitoringServerInfo", "params":"", "id":1}
```

***result***

 - M/OがMとなっているが，データベースにデータが入っていない場合がある。その時そのオブジェクトにはnullとしてresultが返ってくる。

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:---|:--|:----------:|:---------:|:------:|:---|
|serverId          |number|M|-|正の整数|監視サーバーのサーバーID|
|url               |string|M|-|-|監視サーバーのURL [[解説](#user-content-serverType)]|
|type              |string|M|-|-|監視サーバーの種類 [[一覧](#user-content-serverType)]|
|nickName          |string|M|-|255文字以内|監視サーバーのニックネーム|
|userName          |string|M|-|255文字以内|監視サーバーのユーザーネーム|
|password          |string|M|-|255文字以内|監視サーバーのパスワード|
|dbName            |string|M|-|255文字以内|監視対象のデータベースのパスワード|
|pollingIntervalSec|number|M|-|正の整数|ポーリングを行う間隔|
|retryIntervalSec  |number|M|-|正の整数|ポーリングが失敗した場合，リトライを行うまでの間隔|
|extra             |string|M|-|255文字以内|プラグイン固有の情報を格納することができる|

```
{"jsonrpc":"2.0", "result":{"hostName":"exampleHost", "type":0, "ipAddress":"127.0.0.1", "nickName":"exampleName", "userName":"Admin", "password":"examplePass", "dbName":"", "port":80, "pollingIntervalSec":30, "retryIntervalSec":10}, "id":1}
```

### getLastInfo(method)

 - update~~~プロシージャでHatoholサーバーに送信，保存されたlastInfo情報を要求します。
 - 初回起動時など，HatoholサーバーにlastInfoが保存されていない場合，resultオブジェクトはnullとして返ってきます。
 - update~~~プロシージャは，getLastInfoプロシージャで取得したlastInfoを用いて，前回送信したデータと現在所持しているデータの差分をHatoholサーバーに送信できます。

***params***

```
{"jsonrpc":"2.0", "method":"getLastInfo", "params":"trigger", "id":1}
```

|paramsの値|
|:---------|
|"host"|
|"hostGroup"|
|"hostGroupElement"|
|"trigger"|
|"event"  |
|"relation"|

***result***

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|result|string|M|-|255文字以内|Hatoholサーバーに保存されている指定した要素の最新情報|

```
{"jsonrpc":"2.0", "result":"201504011349", "id":1}

この例ではlastInfoとしてtimestampが返ってきています
```

### updateItems(method)

 - Hatoholサーバーとの接続完了時，または[fetchItems](#user-content-fetchItems)プロシージャをHatoholサーバーから受け取った時に全てのアイテム情報を送信することを標準動作とします。Hatoholサーバーの負荷が高くなることが危惧されるため，任意のタイミングで使用することはできません。
 - fetchItemsプロシージャのparamsオブジェクト内のfetchIdオブジェクトの値をupdateItemsプロシージャのparamsのfetchIdオブジェクトに入れてください。

***params***

名前：items, fetchId

各オブジェクトの値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:--|:-----------|:-------|:--|
|items|object|M|-|-|アイテム情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|fetchId|string|M|-|-|Hatoholサーバーから送られたどのリクエストに対するレスポンスであるかを示すIDです。fetchItemsプロシージャのparams内のfetchIdオブジェクトの値をここに入れてください。|

***itemsオブジェクト***

オブジェクトの名前：アイテムID

オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:---|:--|:----------:|:---------:|:------:|:---|
|hostId       |number|M|-|正の整数     |アイテムが所属するホストのID|
|brief        |string|M|-|65535byte以内|アイテムの概要|
|lastValueTime|string|M|-|65535byte以内|アイテムが最後に更新された時間|
|lastValue    |string|M|-|65535byte以内|アイテムが最後に更新された際の値|
|prevValue    |string|M|-|65535byte以内|アイテムが最後に更新される前の値|
|itemGroupName|string|M|-|65535byte以内|アイテムをグループ分けしたもの<br>任意のグループ名をご使用ください|
|unit         |string|M|-|65535byte以内|valueの単位|

```
{"jsonrpc":"2.0","method":"updateItems", "params":{"items":{"1":{"hostId":"1", "brief":"example brief", "lastValueTime":"201504101755", "lastValue":"example value", "prevValue":"example previous value", "itemGroupName":"example name", "unit":"example unit"}}, "fetchId":"1"}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### sendHistory(notification)

 - [fetchHistory](#user-content-fetchhistory)プロシージャをHatoholサーバーから受け取った際に，条件にマッチするヒストリーをHatoholサーバーに送信するします。Hatoholサーバーの負荷が高くなることが危惧されるため，任意のタイミングで使用することはできません。
 - fetchHistoryプロシージャのparamsオブジェクト内の"fetchId"オブジェクトの値をsendプロシージャのparamsの"fetchId"オブジェクトに入れてください。

***params***
名前：history, fetchId

各オブジェクトの値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:--|:-----------|:-------|:--|
|history|object|M|-|-|ヒストリー情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|fetchId|string|M|-|-|Hatoholサーバーから送られたどのリクエストに対するレスポンスであるかを示すIDです。fetchHistoryのparams内のfetchIdオブジェクトの値をここに入れてください|

***historyオブジェクト***

オブジェクトの名前:アイテムID

オブジェクトの値: 配列

配列の値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|value |string|M|-|65535byte以内|clock時点でのアイテムの値|
|clock |string|M|-|65535byte以内|このヒストリーの値が記録された時刻|

```
{"jsonrpc":"2.0", "method":"sendHistory", "params":{"history":{"1":[{"value":"exampleValue","clock":"201503231130"},{"value":"exampleValue2","clock":"201503231130"}]}, "fetchId":"1"}, "id":1}
```

### updateHosts(method)

 - Hatoholサーバーとの接続完了時，またはHAPが内部的に保存している登録ホスト情報が変更された際は"ALL"オプションを用い，全てのホスト情報をHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getLastInfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに追加されたホストをHatoholサーバーに送信します。

***params***

名前：hosts, lastInfo, option

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|hosts|object|M|-|-|ホスト情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|No|null|65535byte以内|最後に送信したホストの情報を送信する。この情報が[getLastInfo](#user-content-getLastInfo)の返り値になる|

***hostsオブジェクト***

オブジェクトの名前：ホストID

オブジェクトの値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|hostName|string|M|-|65535byte以内|監視サーバーが監視しているホスト名|

```
{"jsonrpc":"2.0","method":"updateHosts", "params":{"hosts":{1":"exampleHostName1"},"option":"UPDATE","lastInfo":"201504091052"}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateHostGroups(method)

 - Hatoholサーバーとの接続完了時，またはHAPが内部的に保存している登録ホスト情報が変更された際は"ALL"オプションを用い，全てのホストグループ情報をHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getLastInfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに追加されたホストグループをHatoholサーバーに送信します。

***params***

名前：hostGroups, lastInfo, option

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|hostGroups|object|M|-|-|ホストグループ情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|No|null|65535byte以内|最後に送信したホストグループの情報を送信する。この情報が[getLastInfo](#user-content-getLastInfo)の返り値になる|

***hostGroupsオブジェクト***

オブジェクトの名前：グループID

オブジェクトの値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|groupName|string|M|-|65535byte以内|グループIDに対応したホストグループの名前|

```
{"jsonrpc":"2.0","method":"updateHostGroups", "params":{"hostGroups":{"1":"Group1", "2":"Group2"},"option":"ALL", "lastInfo":"201504091049"}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateHostGroupElements(method)

 - Hatoholサーバーとの接続完了時，またはHAPが内部的に保存している登録ホスト情報が変更された際は"ALL"オプションを用い，全てのホストグループ所属情報をHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getLastInfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに追加されたホストグループ所属情報をHatoholサーバーに送信します。

***params***

名前：hostGroupElements, lastInfo, option

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|hostGroupElements|object|M|-|-|ホストグループ所属情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|No|null|65535byte以内|最後に送信したホストグループ所属情報の情報を送信する。この情報が[getLastInfo](#user-content-getLastInfo)の返り値になる|

***hostGroupElementsオブジェクト***

オブジェクトの名前：ホストID

オブジェクトの値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|groupId|number|M|- |正の整数|監視ホストが所属しているグループのID|

```
{"jsonrpc":"2.0","method":"updateHostGroupElements", "params":{"hostGroupsElements":{"1":1, "2":2},"lastInfo":"201504091056","option":"ALL"}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateTriggers(method)

[getLastInfo](#user-content-getLastInfo)を用いて取得，またはHAPプロセス自身が保管している最新トリガー情報を基に，そのトリガーから現時点までに更新されたトリガーをHatoholサーバーに送信するか，全てのトリガーを送信します。
 - Hatoholサーバーとの接続完了時，fetchTriggersプロシージャによる要求があった際は"ALL"オプションを用い，全てのトリガーをHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getLastInfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに更新，追加されたトリガーをHatoholサーバーに送信します。

***params***

名前：triggers, lastInfo, option, fetchId

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|triggers|object|M|-|-|トリガー情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|O|-|65535byte以内|最新トリガーの情報を送信する。この情報が[getLastInfo](#user-content-getLastInfo)の返り値になる|
|fetchId|string|M|-|-|Hatoholサーバーから送られたどのリクエストに対するレスポンスであるかを示すIDです。fetchTriggersnoのparams内のfetchIdオブジェクトの値をここに入れてください|

***triggersオブジェクト***

HAP自身のトリガーを送信する場合は，トリガーIDとホストIDを"_SELF_"と記述することで送信したトリガーをSELFトリガー扱いにできます。SELFトリガーはHAPがHAPをプロセス自身を監視し，異常が起きていないかどうかを検知します。


|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|status        |string|M|-|-            |トリガーのステータス [[一覧](#user-content-triggerStatus)]|
|severity      |string|M|-|-            |トリガーの種別 [[一覧](#user-content-triggerSeverity)]|
|lastChangeTime|string|M|-|65535byte以内|トリガーが最後に更新された時間|
|hostId        |string|M|-|正の整数     |トリガーが所属するホストID|
|hostName      |string|M|-|65535byte以内|トリガーが所属するサーバーのホスト名|
|brief         |string|M|-|65535byte以内|トリガーの概要|
|extendedInfo  |string|M|-|65535byte以内|上記の情報以外の必要な情報。主にWebUI上にデータを表示する際に用いられる|

```
{"jsonrpc":"2.0", "method":"updateTriggers", "params":{"option":"UPDATED", "lastInfo":"201504061606", "Triggers":{"1":{"status":"OK", "severity":"INFO","lastChangeTime":"201503231758", "hostId":"1", "hostName":"exampleName", "brief":"example brief", "extendedInfo": "sample extended info"}, "_SELF_":{"status":"OK", "severity":"INFO","lastChangeTime":"201503231760", "hostId":"_SELF_ "hostName":"exampleName", "brief":"example brief", "extendedInfo": "sample extended info"}}},"id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateEvents(method)

 - Hatoholサーバーとの接続完了時にHatoholサーバーが過去イベントを同期設定になっていた場合は"ALL"オプションを用い，全てのイベントをHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getLastInfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに更新，発生したイベントをHatoholサーバーに送信します。

***params***

オブジェクトの名前：events, option, lastInfo

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|events|object|M|-|-|イベント情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください。|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|O|null|65535byte以内|最新イベントの情報を送信する。この情報が[getLastInfo](#user-content-getLastInfo)の返り値になる|

***eventsオブジェクト***

オブジェクトの名前：イベントID

オブジェクトの値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|time        |string|M|-|65535byte以内|イベントが発生した時刻|
|type        |string|M|-|-|イベントのタイプ [[一覧](#user-content-eventType)]|
|triggerId   |number|M|-|正の整数     |このイベントを発火させたトリガーID|
|status      |string|M|-|-|トリガーのステータス [[一覧](#user-content-triggerStatus)]|
|severity    |string|M|-|-|トリガーの種別 [[一覧](#user-content-triggerSeverity)]|
|hostId      |string|M|-|65535byte以内|イベントが発生したホストのID|
|hostName    |string|M|-|65535byte以内|イベントが発生したホストの名前|
|brief       |string|M|-|65535byte以内|イベントの説明。Web上に表示される情報|
|extendedInfo|string|M|-|65535byte以内|briefには書いていない追加の情報を記述できます|

```
{"jsonrpc":"2.0", "method":"updateEvents", "params":{"events":{"1":{"time":"201503231513", "type":"GOOD", "triggerId":2, "status": "OK","severity":"INFO":, "hostId":3, "hostName":"exampleName", "brief":"example brief", "extendedInfo": "sampel extended info"}}, "option":"ALL", "lastInfo":"201504011759"},"id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateVirtualRelations(method)

 - Hatoholサーバーとの接続完了時は"ALL"オプションを用い，全てのVM親子関係をHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getLastInfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに追加されたVM親子関係をHatoholサーバーに送信します。

***params***

名前：virtualRelations, lastInfo, option

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|virtualRelations|object|M|-|-|VMの親子関係を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|No|null|65535byte以内|最後に送信したホストグループ所属情報の情報を送信する。この情報が[getLastInfo](#user-content-getLastInfo)の返り値になる|

***virtualRelationsオブジェクト***

オブジェクトの名前：子ホストID
オブジェクトの値：親ホストID

```
{"jsonrpc":"2.0", "method":"updateVirtualRelations", "params":{"12":"10", "11":"10"},"option":"ALL", "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateArmInfo(method)

HostやTrigger，Event情報の送信処理が行われるたびにHatoholサーバーに送信することを標準的な動作としますが，任意に送信してもかまいません。最小間隔は１秒（MUST），最大間隔はgetMonitoringServerInfoで取得したポーリング時間の2倍（SHOULD）とします。

***params***

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:---|:--|:---------:|:----------:|:------:|:---|
|lastStatus         |string   |M|-|-|最新のポーリング結果 [[一覧](#user-content-armInfoStatus)]|
|failureReason      |string   |M|-|65535byte以内|情報取得が失敗した理由|
|lastSuccessTime    |timestamp|M|-|-|最後に情報取得が成功した時刻|
|lastFailureTime    |timestamp|M|-|-|最後に情報取得が失敗した時刻|
|numSuccess         |number   |M|-|正の整数|HAPが起動してから情報取得に成功した回数|
|numFailure         |number   |M|-|正の整数|HAPが起動してから情報取得に失敗した回数|

```
{"jsonrpc":"2.0", "method":"updateArmInfo", "params":{"lastStatus":"INIT", "failureReason":"Example reason", "lastSuccessTime":"201503131611", "lastFailureTime":"201503131615", "numSuccess":165, "numFailure":10}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### fetchItems(method)

このプロシージャは、Hatoholサーバーがアイテムを要求しているときにHAPに送信されます。このプロシージャを受け取った時，resultとしてリクエスト受け入れの成否を返す必要があります。その後，全てのアイテムをupdateItemsプロシージャ[updateItems](#user-content-updateitems)を用いてHatoholサーバーに送信してください。また，paramsのfetchIdオブジェクトの値を[updateItems](#user-contents-updateItems)に渡す必要があります。

***params***

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|fetchId|string|M|-|255文字以内|updateItemsプロシージャで使用します。そのupdateItemsプロシージャがどのfetchItemsプロシージャによる要求に対応したものかをHatoholサーバーが識別するために必要です|

```
{"jsonrpc":"2.0", "method":"fetchItems", "params":{"fetchId":"1"}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### fetchHistory(method)

 - このプロシージャは、Hatoholサーバーがヒストリーを要求しているときにHAPに送信されます。HAPはレスポンスとしてリクエスト受け入れの成否を返す必要があります。その後，指定条件に当てはまるヒストリーをupdateHistoryプロシージャ[updateHistory](#user-content-sendhistory)を用いてHatoholサーバーに送信してください。その際，fetchHistoryプロシージャのparams内にあるfetchIdの値をsendHistoryプロシージャに渡す必要があります。

***params***

名前：reqHistory, fetchId

各オブジェクトの値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:--|:-----------|:-------|:--|
|reqHistory|object|M|-|-|ヒストリーの要求情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|fetchId|string|M|-|255文字以内|sendHistoryプロシージャで使用します。そのsendHistoryプロシージャがどのfetchHistoryプロシージャによる要求に対応したものかをHatoholサーバーが識別するために必要です|

***reqHistoryオブジェクト***

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|hostId   |string|M|-|255文字以内|ヒストリーのアイテムが所属しているホストID|
|itemId   |number|M|-|正の整数   |ヒストリーのアイテムID|
|valueType|string|M|-|-|取得するヒストリーの値の型 [[一覧](#user-content-itemValueType)]|
|beginTime|string|M|-|255文字以内|ヒストリー取得域の始点時間を指定します|
|endTime  |string|M|-|255文字以内|ヒストリー取得域の終点時間を指定します|

```
{"jsonrpc":"2.0", "method":"fetchHistory", "params":{"hostId":"1", "itemId":1, "valueType":"INTERGER", "beginTime":"201503231513, "beginTime":"201503231513"}},"id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### fetchTriggers(method)

 - このプロシージャは、Hatoholサーバーが全てのトリガーを要求しているときにHAPに送信されます。HAPはレスポンスとしてリクエスト受け入れの成否を返す必要があります。その後，[updateTriggersプロシージャ](#user-content-updatetriggers)の"ALL"オプションを用いて監視している全てのトリガーを送信してください。その際，fetchTriggersプロシージャのparams内にあるfetchIdの値をupdateTrriggersプロシージャに渡す必要があります。

***params***

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|fetchId|string|M|-|255文字以内|updateItemsプロシージャで使用します。そのupdateItemsプロシージャがどのfetchItemsプロシージャによる要求に対応したものかをHatoholサーバーが識別するために必要です|

```
{"jsonrpc":"2.0", "method":"fetchItems", "params":{"fetchId":"1"}, "id":1}
```

***result***

 - fetchTriggersメソッドには引数が存在しません。paramsをnullオブジェクトにして送信してください。

***result***

|名前         |型|M/O|デフォルト値|値の範囲|解説|
|:------------|:----|:----:|:----------:|:------:|:---|
|status        |number|M|-|正の整数     |トリガーのステータス|
|severity      |string|M|-|-|トリガーの種別 [[一覧](#user-content-triggerSeverity)]|
|lastChangeTime|string|M|-|65535byte以内|トリガーが最後に更新された時間|
|hostId        |number|M|-|正の整数     |監視サーバー内で設定されているホストID|
|hostName      |string|M|-|65535byte以内|トリガーが所属するサーバーのホスト名|
|brief         |string|M|-|65535byte以内|トリガーの概要|
|extendedInfo  |string|M|-|65535byte以内|上記の情報以外の必要な情報。主にWebUI上にデータを表示する際に用いられる|

```
{"jsonrpc":"2.0", "result":{"1":{"option":"UPDATED", "status":"OK", "severity":"INFO","lastChangeTime":"201503231758", "hostId":"1", "hostName":"exampleName", "brief":"example brief", "extendedInfo": "sample extended info"}},"id":1}
```

### reqTerminate(notification)

 - このプロシージャはHatoholサーバーがHAPを設置している監視対象サーバーの登録を削除した場合にHAPに送信されます。HAPはこのプロシージャを受け取った場合，HAPプロセス自身を終了させてください。

***params***

 - reqTerminateプロシージャには引数が存在しません。paramsをnullオブジェクトにして送信してください。

***result***

 - reqTerminateプロシージャにはresultが存在しません。


## 表

### errorCode

リクエストに成功した場合，送信したリクエストに応じたresultオブジェクトが返されます。
リクエストに失敗した場合，resultオブジェクトではなくerrorオブジェクトを返す必要があります。
このセクションではエラーオブジェクトとして返すエラーコードとエラーメッセージについて解説します。
状況に応じて適したエラーコードを使用してください。

|code|message|meaning|
|:--|:-------|:---|
|1  |Unknown error|原因不明のエラーが起こっています|

### armInfoStatus

|ステータス|解説|
|:---------|:---|
|"INIT"   |初期状態。まだ通信を行っていない|
|"OK"     |通信に成功している|
|"FAILURE"|通信に失敗している|

### serverType

以下が既存のHAPを識別する際に使用されている各サーバータイプのURLとUUIDです。あなたがHAPを作成する場合，サーバータイプを新しく定義する必要があります。以下のUUIDと被っていない任意のUUIDを生成し，以下のドキュメント参考にして新しいサーバータイプを定義してください。[[ほげほげ](#http://なんちゃらかんちゃら)]

|名前|UUID|URL|
|:---|:---|:--|
|Zabbix      |8e632c14-d1f7-11e4-8350-d43d7e3146fb||
|Nagios      |902d955c-d1f7-11e4-80f9-d43d7e3146fb||
|Ceilometer  |aa25a332-d1f7-11e4-80b4-d43d7e3146fb||

### triggerSeverity

トリガーの種別です。任意のステータスを各トリガーに設定してください。

|種別|
|:---|
|"ALL"      |
|"UNKNOWN"  |
|"INFO"     |
|"WARNING"  |
|"ERROR"    |
|"CRITICAL" |
|"EMERGENCY"|

### triggerStatus

|ステータス|解説|
|:---------|:---|
|"ALL"    |初期状態。まだ通信を行っていない|
|"OK""    |通信に成功している|
|"PROBLEM"|通信に失敗している|
|"UNKNOWN"|状態不明|

### option

|種類|解説|
|:---------|:---|
|"ALL"    |各データ全てを送信します。Hatoholサーバー内の古いデータを削除し，その後送信した全てのデータを登録します|
|"UPDATED"|アップデートされたデータのみをHatoholサーバーに送信し，同一IDのデータは上書き，初出のデータは新規登録します|

### eventType

イベントのタイプです。任意のタイプを各イベントに設定して下さい。

|タイプ|解説|
|:-----|:--:|
|"GOOD"   |正常|
|"BAD"    |異常|
|"UNKNOWN"|不明|

### itemValueType

アイテムのタイプです。任意のタイプを各アイテムに設定して下さい。

|タイプ|解説|
|:-----|:---|
|"UNKNOWN"|未定義  |
|"FLOAT"  |float型 |
|"INTEGER"|int型   |
|"STRING" |string型|

### updateResult

update~~~プロシージャを送信した際におけるDB書き込みの成否です。
書き込みに失敗した際は，再度送信するといった動作が標準的です。


|ステータス|解説|
|:---------|:---|
|"SUCCESS"|DBへの書き込みが成功しました|
|"FAILED" |DBへの書き込みに失敗しました|

### fetchResult

fetch~~~プロシージャを受信した際に，そのリクエストを受け入れたかどうかの成否です。書き込みに失敗した際は，再度送信するといった動作が標準的です。

|ステータス|解説|
|:---------|:---|
|"SUCCESS"|リクエストが受け入れられました|
|"ADDREV"|リクエストの受け入れが省略されました<br>リクエストの間隔が近い，といった理由でリクエストを受け入れない場合に使用します|
|"FAILED"|リクエストの受け入れに失敗しました|
