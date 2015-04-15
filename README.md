# Hatohol Arm Plugin Interface 2.0

## ToDO

serverTypeのURL

## 概要

Hatohol Arm Plugin InterfaceはHatoholサーバーとプラグイン間でのデータの送受信を定義するフレームワークです。
通信規格としてAMQP（RabbitMQ），データプロトコルとしてJSON-RPC 2.0を採用しています。JSON-RPC，AMQP(RabbitMQ)の各仕様についてはそれぞれの公式リファレンス([JSON-RPC](http://www.jsonrpc.org/specification)/[RabbitMQ](https://www.rabbitmq.com/documentation.html))をご覧ください。

このドキュメントではHatohol Arm Plugin Interface規格に則り,Hatoholサーバーとデータの送受信を行うHatohol Arm Plugin(以下，HAP)の仕様について記載しています。

不明点についてはHatoholコミュニティにお問い合わせください。[hatohol-users@sourceforge.net]

![overview](hapi_overview.png)

## HAP動作概要

```
Hatoholサーバー                                   HAP
    |                                               
    |                                        Turn on HAP
    |                                              |
    |<----------exchangeProfile(リクエスト)------->|
    |<----------exchangeProfile(レスポンス)------->|
    |                                          |   |
    |                               ポーリング間隔 |
    |                                          |   |
    |<-----getMonitoringServerInfo(リクエスト)-----|
    |------getMonitoringServerInfo(レスポンス)---->|
    |                                              |
    |<-----------getLastInfo(リクエスト)-----------|
    |------------getLastInfo(レスポンス)---------->|
    |<-----------updateHosts(リクエスト)-----------|
    |------------updateHosts(レスポンス)---------->|
    |                                              |
    |<-----------getLastInfo(リクエスト)-----------|
    |------------getLastInfo(レスポンス)---------->|
    |<--------updateHostGroups(リクエスト)---------|
    |---------updateHostGroups(レスポンス)-------->|
    |                                              |
    |<-----------getLastInfo(リクエスト)-----------|
    |------------getLastInfo(レスポンス)---------->|
    |<----updateHostGroupMembership(リクエスト)----|
    |-----updateHostGroupMembership(レスポンス)--->|
    |                                              |
    |<-----------getLastInfo(リクエスト)-----------|
    |------------getLastInfo(レスポンス)---------->|
    |<-----------updateTriggers(リクエスト)--------|
    |------------updateTriggers(レスポンス)------->|
    |                                              |
    |<-----------getLastInfo(リクエスト)-----------|
    |------------getLastInfo(レスポンス)---------->|
    |<-----------updateEvents(リクエスト)----------|
    |------------updateEvents(レスポンス)--------->|
    |                                              |
    |<-----------getLastInfo(リクエスト)-----------|
    |------------getLastInfo(レスポンス)---------->|
    |<-----------updateHostParent(リクエスト)------|
    |------------updateHostParent(レスポンス)----->|
    |                                              |
    |<-----------updateArmInfo(リクエスト)---------|
    |------------updateArmInfo(レスポンス)-------->|
    |                                          |   |
    |                               ポーリング間隔 |
    |                                          |   |
 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    |                                              |
    |-----------fetchItems(リクエスト)------------>|
    |<----------fetchItems(レスポンス)-------------|
    |<----------updateItems(リクエスト)------------|
    |-----------updateItems(レスポンス)----------->|
    |                                              |
    |-----------fetchHistory(リクエスト)---------->|
    |<----------fetchHistory(レスポンス)-----------|
    |<-----sendHistory(ノーティフィケイション)-----|
    |                                              |
    |-----------fetchTriggers(リクエスト)--------->|
    |<----------fetchTriggers(レスポンス)----------|
    |<---------updateTriggers(リクエスト)----------|
    |"ALL"オプションを使用し全てのトリガーを送信する|
    |----------updateTriggers(レスポンス)--------->|
    |                                              |
    |-------------fetchEvents(リクエスト)--------->|
    |<------------fetchEvents(レスポンス)----------|
    |<-----------updateEvents(リクエスト)----------|
    |    "SOME"オプションを使用し                  |
    |         指定された件数のイベントを送信する   |
    |------------updateEvents(レスポンス)--------->|
    |                                              |

```

## 注意事項

 - リクエスト・レスポンスで使用するIDオブジェクトの値には，十分なランダム性が必要です。
 - JSON-RPCにはバッチリクエストという，複数のリクエストを同時に送信する文法が存在しますが，Hatoholはこの文法を用いたリクエストには対応していません。
 - HAPからHatoholサーバーに送信されるプロシージャで使用されるnumber型オブジェクトの値範囲は0~2147483647です。

## データ型解説

 - このセクションは，JSON-RPCではなくHatoholが独自に定義しているデータ型について解説します。これらは内部的にはJSON-RPCが定義しているデータ型を使用しています。

|名前|JSON型|解説|
|:---|:---------|:---|
|timestamp|string|時刻フォーマットはyyyyMMDDHHmmss.nnnnnnnnnです。小数点以下の時刻については省略できます。また，小数点以下には9桁までしか値を挿入できません。小数点以下を省略した場合，または小数点以下が9桁未満の場合には余った桁部に0が挿入されます。(Ex.100 -> 100.000000000, 100.1234 -> 100.123400000)。|
|boolean|true, false|true or falseを指定し，その値の真偽を示します|

## exchangeProfile

 - exchangeProfileプロシージャは実装を省略することは出来ません。
 - HAP，またはHatoholサーバーの初回起動時や，なんらかの理由で再起動した際に，自身と接続相手が使用可能なプロシージャの一覧をexchangeProfileプロシージャを用いて教え合います。この情報を基にお互いに向けて送信するプロシージャの最適化を行います。
 - HAP，またはHatoholサーバーを再起動した際，AMQPのキューに再起動以前のリクエストが残留している可能性があります。しかし再起動したHAPやHatoholサーバーのプロセスはそれらのリクエストIDを覚えていないため，接続相手に処理されて返ってきたレスポンスを受け入れることはありません。
 - exchangeProfileプロシージャのリクエストを送信し，そのレスポンスが返ってくるまでに再度exchangeProfileのリクエストを送信した際は後に発行されたリクエストが優先されます。先に送信したリクエスト情報は破棄され，古いリクエストに対するレスポンスを受け入れることはありません。

以下にJSON-RPC形式のexchangeProfileプロシージャのリクエスト，リザルトを示します。

***リクエスト***

HAPがHatoholサーバーとの接続に成功した際，paramsに入れた値がHAPの名前としてログに出力されます。
```
{"jsonrpc":"2.0", "method":"exchangeProfile", "params":{"procedure":["getMonitoringServerInfo", "getLastInfo", "updateItems", "updateArmInfo", "fetchItems"], "AgentName":"exampleName"} "id":1}
```

***リザルト***

```
{"jsonrpc":"2.0", "result":["getMonitoringServerInfo", "getLastInfo", "updateItems", "updateArmInfo", "fetchItems"], "id":1}
```

## プロシージャ解説

 - Hatoholサーバーに実装されているプロシージャ

|プロシージャ名|解説|タイプ|M/O|
|:-------------|:---|:-----|:-:|
|[getMonitoringServerInfo](#user-content-getmonitoringserversnfo)|接続情報やポーリング間隔等をHatoholサーバーから取得します|method|M|
|[getLastInfo](#user-content-getlastinfo)|指定した要素の最新情報をHatoholサーバーから取得します|method|M
|[updateItems](#user-content-updateitems)|監視しているアイテム一覧をHatoholサーバーに送信します|method|O|
|[sendHistory](#user-content-sendhistory)|各アイテムが所持しているアイテムのヒストリーをHatoholサーバーに送信します|notification|O|
|[updateHosts](#user-content-updatehosts)|監視しているホスト一覧をHatoholサーバーに送信します|method|O|
|[updateHostGroups](#user-content-updatehostgroups)|ホストグループの情報をHatoholサーバーに送信します|method|O|
|[updateHostGroupMembership](#user-content-updatehostgroupmembership)|ホストのホストグループ所属情報をHatoholサーバーに送信します|method|O|
|[updateTriggers](#user-content-updatetrigges)|監視しているトリガーをHatoholサーバーに送信します|method|O|
|[updateEvents](#user-content-updateevents)|アップデートされたイベントをHatoholサーバーに送信します|method|O|
|[updateHostParent](#user-content-updatehostparent)|ホスト同士のVM親子関係をHatoholサーバーに送信します|method|O|
|[updateArmInfo](#user-content-updatearminfo)|HAPの接続ステータスをHatoholサーバーに送信します|method|M|

 - HAPに実装するプロシージャ

|プロシージャ名|解説|タイプ|M/O|
|:-------------|:---|:-----|:-:|
|[fetchItems](#user-content-fetchitems)|Hatoholサーバーがアイテムを要求しているときにHAPに送信されます|method|O|
|[fetchHistory](#user-content-fetchhistory)|Hatoholサーバーがヒストリーを要求しているときにHAPに送信されます|method|O|
|[fetchTriggers](#user-content-fetchtriggers)|Hatoholサーバーが全てのトリガーを要求しているときにHAPに送信されます|method|O|
|[fetchEvents](#user-content-fetchevents)|HatoholサーバーがHatoholDBに登録されている最古イベント以前のイベントを要求しているときにHAPに送信されます|method|O|

 - 「M/O」はそのプロシージャがMandatory(必須)かOptional(任意)であるかを表します。これがMのプロシージャは実装を省略できません。
 - M/OがOとなっているプロシージャは実装を省略可能です。しかし，fetch~~~プロシージャのようにHatoholサーバーからリクエストを受けるプロシージャの実装を省略している場合は，呼び出されたプロシージャが実装されていないことをエラーとして返す必要があります。エラーメッセージをerrorオブジェクトに入れてHatoholサーバーにレスポンスを返してください。[エラーメッセージ一覧](#user-content-errorcode)
 - update~~~プロシージャは，送信したデータのデータベース書き込み成否をresultオブジェクトで受け取ります。受け取る値については[[一覧](#user-content-updateresult)]をご覧ください。
 - fetch~~~プロシージャで受けたリクエスト受け入れの成否をresultオブジェクトとしてHatoholサーバーに返す必要があります。返す値については[[一覧](#user-content-fetchresult)]をご覧ください。
 - fetch~~~プロシージャはHatoholサーバーからのリクエスト頻度が高い場合は受け入れを拒否することができます。

### getMonitoringServerInfo(method)

 - ポーリング時間毎にHatoholサーバーに自身の接続情報やポーリング間隔等を問い合わせることを標準的な動作としますが，任意のタイミングで問い合わせることもできます。

***params***

 - getMonitoringServerInfoメソッドには引数が存在しません。paramsオブジェクトの値をnullにしてリクエストを送信してください。

```
{"jsonrpc":"2.0", "method":"getMonitoringServerInfo", "params":null, id":1}
```

***result***

|オブジェクトの名前|型|M/O|デフォルト値|値の範囲|解説|
|:---|:--|:----------:|:---------:|:------:|:---|
|serverId          |number|M|-|正の整数|監視サーバーのサーバーID|
|url               |string|M|-|-|監視サーバーのURL [[解説](#user-content-servertype)]|
|type              |string|M|-|-|監視サーバーの種類 [[一覧](#user-content-servertype)]|
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
 - 初回起動時など，HatoholサーバーにlastInfoが保存されていない場合，resultオブジェクトの値はnullとして返ってきます。
 - update~~~プロシージャは，getLastInfoプロシージャで取得したlastInfoを用いて，前回までに送信したデータと現在所持しているデータの差分をHatoholサーバーに送信できます。

***params***

 - どの種類のlastInfoが必要かを指定する必要があります。以下の表は指定できる値の一覧です。

|paramsオブジェクトの値一覧(string)|解説|
|:---------|:---|
|"host"|ホストの最新情報を指定します。|
|"hostGroup"|ホストグループの最新情報を指定します。|
|"hostGroupMembership"|ホストグループの所属情報の最新情報を指定します。|
|"trigger"|トリガーの最新情報を指定します。|
|"event"|イベントの最新情報を指定します。|
|"hostParent"|VMの親子関係の最新情報を指定します。|

```
{"jsonrpc":"2.0", "method":"getLastInfo", "params":"trigger", "id":1}
```

***result***

|resultオブジェクトの値|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|最新情報|string|M|-|255文字以内|Hatoholサーバーに保存されている指定した要素の最新情報|

```
{"jsonrpc":"2.0", "result":"201504011349", "id":1}

この例ではlastInfoとしてタイムスタンプが返ってきています
```

### updateItems(method)

 - Hatoholサーバーとの接続完了時，または[fetchItems](#user-content-fetchitems)プロシージャをHatoholサーバーから受け取った時に全てのアイテム情報を送信することを標準動作とします。Hatoholサーバーの負荷が高くなることが危惧されるため，任意のタイミングで使用することはできません。

***params***

|オブジェクトの名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:--|:-----------|:-------|:--|
|items|object|M|-|-|アイテム情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|fetchId|string|M|-|-|Hatoholサーバーから送られたどのfetchItemsプロシージャによるリクエストに対するレスポンスであるかを示すIDです。fetchItemsプロシージャのparams内のfetchIdオブジェクトの値をここに入れてください。|

***itemsオブジェクト***

オブジェクトの名前：アイテムID

オブジェクトの値：オブジェクト

|オブジェクトの名前|型|M/O|デフォルト値|値の範囲|解説|
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

***params***

|オブジェクトの名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:--|:-----------|:-------|:--|
|history|object|M|-|-|ヒストリー情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|fetchId|string|M|-|-|Hatoholサーバーから送られたどのリクエストに対するレスポンスであるかを示すIDです。fetchHistoryのparams内のfetchIdオブジェクトの値をここに入れてください|

***historyオブジェクト***

オブジェクトの名前:アイテムID

オブジェクトの値: 配列

配列の値：オブジェクト

|オブジェクトの名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|value |string|M|-|65535byte以内|clock時点でのアイテムの値|
|clock |string|M|-|65535byte以内|このヒストリーの値が記録された時刻|

```
{"jsonrpc":"2.0", "method":"sendHistory", "params":{"history":{"1":[{"value":"exampleValue","clock":"201503231130"},{"value":"exampleValue2","clock":"201503231130"}]}, "fetchId":"1"}, "id":1}
```

### updateHosts(method)

 - Hatoholサーバーとの接続完了時，またはHAPが内部的に保存している登録ホスト情報が変更された際は"ALL"オプションを用い，全てのホスト情報をHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getlastinfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに追加されたホストをHatoholサーバーに送信します。

***params***

|オブジェクトの名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|hosts|object|M|-|-|ホスト情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|O|-|65535byte以内|最後に送信したホストの情報を送信する。この情報が[getLastInfo](#user-content-getlastinfo)の返り値になる|

***hostsオブジェクト***

オブジェクトの名前：ホストID

|オブジェクトの値|型 |M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:----------:|:------:|:---|
|ホスト名|string|M|-|65535byte以内|監視サーバーが監視しているホスト名|

```
{"jsonrpc":"2.0","method":"updateHosts", "params":{"hosts":{1":"exampleHostName1"},"option":"UPDATE","lastInfo":"201504091052"}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateHostGroups(method)

 - Hatoholサーバーとの接続完了時，またはHAPが内部的に保存している登録ホスト情報が変更された際は"ALL"オプションを用い，全てのホストグループ情報をHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getlastinfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに追加されたホストグループをHatoholサーバーに送信します。

***params***

|オブジェクトの名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|hostGroups|object|M|-|-|ホストグループ情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|O|-|65535byte以内|最後に送信したホストグループの情報を送信する。この情報が[getLastInfo](#user-content-getlastinfo)の返り値になる|

***hostGroupsオブジェクト***

オブジェクトの名前：グループID

|オブジェクトの値|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|グループ名|string|M|-|65535byte以内|グループIDに対応したホストグループの名前|

```
{"jsonrpc":"2.0","method":"updateHostGroups", "params":{"hostGroups":{"1":"Group1", "2":"Group2"},"option":"ALL", "lastInfo":"201504091049"}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateHostGroupMembership(method)

 - Hatoholサーバーとの接続完了時，またはHAPが内部的に保存している登録ホスト情報が変更された際は"ALL"オプションを用い，全てのホストグループ所属情報をHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getlastinfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに追加されたホストグループ所属情報をHatoholサーバーに送信します。

***params***

|オブジェクトの名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|hostGroupMembership|object|M|-|-|ホストグループ所属情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|O|-|65535byte以内|最後に送信したホストグループ所属情報の情報を送信する。この情報が[getLastInfo](#user-content-getlastinfo)の返り値になる|

***hostGroupMembershipオブジェクト***

オブジェクトの名前：ホストID

オブジェクトの値：グループIDのString配列

```
{"jsonrpc":"2.0","method":"updateHostGroupMembership", "params":{"hostGroupsMembership":{"1":["1", "2", "5"], "2":["3", "4", "6"]},"lastInfo":"201504091056","option":"ALL"}, "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateTriggers(method)

[getLastInfo](#user-content-getlastinfo)を用いて取得，またはHAPプロセス自身が保管している最新トリガー情報を基に，そのトリガーから現時点までに更新されたトリガーをHatoholサーバーに送信するか，全てのトリガーを送信します。
 - Hatoholサーバーとの接続完了時，fetchTriggersプロシージャによる要求があった際は"ALL"オプションを用い，全てのトリガーをHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getlastinfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに更新，追加されたトリガーをHatoholサーバーに送信します。

***params***

名前：triggers, lastInfo, option, fetchId

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|triggers|object|M|-|-|トリガー情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|O|-|65535byte以内|最新トリガーの情報を送信する。この情報が[getLastInfo](#user-content-getlastinfo)の返り値になる|
|fetchId|string|O|-|-|Hatoholサーバーから送られたどのリクエストに対するレスポンスであるかを示すIDです。fetchTriggersのparams内のfetchIdオブジェクトの値をここに入れてください|

***triggersオブジェクト***

HAP自身のトリガーを送信する場合は，トリガーIDとホストIDを"_SELF_"と記述することで送信したトリガーをSELFトリガー扱いにできます。SELFトリガーはHAPがHAPをプロセス自身を監視し，異常が起きていないかどうかを検知します。


|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|status        |string|M|-|-            |トリガーのステータス [[一覧](#user-content-triggerstatus)]|
|severity      |string|M|-|-            |トリガーの種別 [[一覧](#user-content-triggerseverity)]|
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
updateEventsはmaxかいとく

 - Hatoholサーバーとの接続完了時にHatoholサーバーが過去イベントを同期設定になっていた場合は"ALL"オプションを用い，全てのイベントをHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getlastinfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに更新，発生したイベントをHatoholサーバーに送信します。

***params***

オブジェクトの名前：events, lastInfo

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|events|object|M|-|-|イベント情報を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください。|
|lastInfo|string|O|-|65535byte以内|最新イベントの情報を送信する。この情報が[getLastInfo](#user-content-getlastinfo)の返り値になる|

***eventsオブジェクト***

オブジェクトの名前：イベントID

オブジェクトの値：

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|time        |string|M|-|65535byte以内|イベントが発生した時刻|
|type        |string|M|-|-|イベントのタイプ [[一覧](#user-content-eventtype)]|
|triggerId   |number|M|-|正の整数     |このイベントを発火させたトリガーID|
|status      |string|M|-|-|トリガーのステータス [[一覧](#user-content-triggerstatus)]|
|severity    |string|M|-|-|トリガーの種別 [[一覧](#user-content-triggerseverity)]|
|hostId      |string|M|-|65535byte以内|イベントが発生したホストのID|
|hostName    |string|M|-|65535byte以内|イベントが発生したホストの名前|
|brief       |string|M|-|65535byte以内|イベントの説明。Web上に表示される情報|
|extendedInfo|string|M|-|65535byte以内|briefには書いていない追加の情報を記述できます|

```
{"jsonrpc":"2.0", "method":"updateEvents", "params":{"events":{"1":{"time":"201503231513", "type":"GOOD", "triggerId":2, "status": "OK","severity":"INFO":, "hostId":3, "hostName":"exampleName", "brief":"example brief", "extendedInfo": "sampel extended info"}}, "lastInfo":"201504011759"},"id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateHostParent(method)

 - Hatoholサーバーとの接続完了時は"ALL"オプションを用い，全てのVM親子関係をHatoholサーバーに送信します。
 - "UPDATE"オプションを用いた場合は[getLastInfo](#user-content-getlastinfo)プロシージャ，またはHAPプロセス自身から呼び出したlastInfoを基に，その時点から現時点までに追加されたVM親子関係をHatoholサーバーに送信します。

***params***

名前：hostParent, lastInfo, option

各オブジェクトの値：

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:--|:--|:--|:--|:--|:--|
|hostParent|object|M|-|-|VMの親子関係を格納するオブジェクトを配置します。詳細は次のテーブルを確認してください|
|option|string|M|-|-|送信オプション[[一覧]("user-content-option")]の中から状況に応じた送信オプションを選択してください|
|lastInfo|string|O|-|65535byte以内|最後に送信したホストグループ所属情報の情報を送信する。この情報が[getLastInfo](#user-content-getlastinfo)の返り値になる|

***hostParentオブジェクト***

 - VMの親子関係を削除する場合は親ホストIDの値をnullにすることで，送信した子ホストIDの親子関係をHatoholサーバーから削除することができます。

オブジェクトの名前：子ホストID(String)
オブジェクトの値：親ホストID(String or null)

```
{"jsonrpc":"2.0", "method":"updateHostParent", "params":{"12":"10", "11":"10"},"option":"ALL", "id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### updateArmInfo(method)

HostやTrigger，Event情報の送信処理が行われるたびにHatoholサーバーに送信することを標準的な動作としますが，任意に送信してもかまいません。最小間隔は１秒（MUST），最大間隔は[getMonitoringServerInfo](#user-content-getmonitoringserverinfo)で取得したポーリング時間の2倍（SHOULD）とします。

***params***

|名前|型|M/O|デフォルト値|値の範囲|解説|
|:---|:--|:---------:|:----------:|:------:|:---|
|lastStatus         |string   |M|-|-|最新のポーリング結果 [[一覧](#user-content-arminfostatus)]|
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

このプロシージャは、Hatoholサーバーがアイテムを要求しているときにHAPに送信されます。このプロシージャを受け取った時，resultとしてリクエスト受け入れの成否を返す必要があります。その後，全てのアイテムをupdateItemsプロシージャ[updateItems](#user-content-updateitems)を用いてHatoholサーバーに送信してください。また，paramsのfetchIdオブジェクトの値を[updateItems](#user-contents-updateitems)に渡す必要があります。

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
|reqHistory|array|M|-|-|ヒストリーの要求情報を格納する配列を配置します。この配列の中に詳細は次のテーブルを確認してください|
|fetchId|string|M|-|255文字以内|sendHistoryプロシージャで使用します。そのsendHistoryプロシージャがどのfetchHistoryプロシージャによる要求に対応したものかをHatoholサーバーが識別するために必要です|

***reqHistoryオブジェクト***

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|hostId   |string|M|-|255文字以内|ヒストリーのアイテムが所属しているホストID|
|itemId   |number|M|-|正の整数   |ヒストリーのアイテムID|
|valueType|string|M|-|-|取得するヒストリーの値の型 [[一覧](#user-content-itemvaluetype)]|
|beginTime|string|M|-|255文字以内|ヒストリー取得域の始点時間を指定します|
|endTime  |string|M|-|255文字以内|ヒストリー取得域の終点時間を指定します|

```
{"jsonrpc":"2.0", "method":"fetchHistory", "params":{"reqHistory":[{"hostId":"1", "itemId":1, "valueType":"INTERGER", "beginTime":"201503231513", "beginTime":"201503231513"}], "fetchId":1 },"id":1}
```

***result***

```
{"jsonrpc":"2.0", "result":"SUCCESS", "id":1}
```

### fetchTriggers(method)

 - このプロシージャは、Hatoholサーバーが全てのトリガーを要求しているときにHAPに送信されます。HAPはレスポンスとしてリクエスト受け入れの成否を返す必要があります。その後，[updateTriggersプロシージャ](#user-content-updatetriggers)の"ALL"オプションを用いて監視している全てのトリガーを送信してください。その際，fetchTriggersプロシージャのparams内にあるfetchIdの値をupdateTriggersプロシージャに渡す必要があります。

***params***

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|fetchId|string|M|-|255文字以内|updateTriggersプロシージャで使用します。そのupdateTriggersプロシージャがどのfetchTriggersプロシージャによる要求に対応したものかをHatoholサーバーが識別するために必要です|

```
{"jsonrpc":"2.0", "method":"fetchTriggers", "params":{"fetchId":"1"}, "id":1}
```

***result***

 - fetchTriggersメソッドには引数が存在しません。paramsオブジェクトの値をnullにしてリクエストを送信してください。

***result***

|名前         |型|M/O|デフォルト値|値の範囲|解説|
|:------------|:----|:----:|:----------:|:------:|:---|
|status        |number|M|-|正の整数     |トリガーのステータス|
|severity      |string|M|-|-|トリガーの種別 [[一覧](#user-content-triggerseverity)]|
|lastChangeTime|string|M|-|65535byte以内|トリガーが最後に更新された時間|
|hostId        |number|M|-|正の整数     |監視サーバー内で設定されているホストID|
|hostName      |string|M|-|65535byte以内|トリガーが所属するサーバーのホスト名|
|brief         |string|M|-|65535byte以内|トリガーの概要|
|extendedInfo  |string|M|-|65535byte以内|上記の情報以外の必要な情報。主にWebUI上にデータを表示する際に用いられる|

```
{"jsonrpc":"2.0", "result":{"1":{"option":"UPDATED", "status":"OK", "severity":"INFO","lastChangeTime":"201503231758", "hostId":"1", "hostName":"exampleName", "brief":"example brief", "extendedInfo": "sample extended info"}},"id":1}
```

### fetchEvents(method)

 - このプロシージャは、HatoholサーバーがHatoholDBに保存されている最古のイベント以前のイベントを要求しているときにHAPに送信されます。HAPはレスポンスとしてを返す必要があります。その後，[updateEventsプロシージャ](#user-content-updateevents)の"SOME"オプションを用いて指定されたIDのイベントから，指定された件数の過去イベントを送信してください。その際，fetchEventsプロシージャのparams内にあるfetchIdの値をupdateEventsプロシージャに渡す必要があります。
 - 取得時できる件数は一度のリクエストで1000件までです。それ以上の件数を取得したい場合は複数回のリクエストに分けて取得してください。
 - リクエストしたID以前のイベントが存在しない場合はレスポンスとしてresultオブジェクトの値をnullにしてください。

***params***

|名前|型 |M/O|デフォルト値|値の範囲|解説|
|:---|:--|:-------:|:----------:|:------:|:---|
|reqBeforeEvents|object|M|-|-|過去のイベントを要求する際に必要な情報をまとめています|
|fetchId|string|M|-|255文字以内|updateEventsプロシージャで使用します。そのupdateEventsプロシージャがどのfetchEventsプロシージャによる要求に対応したものかをHatoholサーバーが識別するために必要です|

```
{"jsonrpc":"2.0", "method":"fetchEvents", "params":{"fetchId":"1", "baseEventId":"10", "number": "1000"}, "id":1}
```

***result***

 - fetchEventsメソッドには引数が存在しません。paramsオブジェクトの値をnullにしてリクエストを送信してください。

***result***

|名前         |型|M/O|デフォルト値|値の範囲|解説|
|:------------|:----|:----:|:----------:|:------:|:---|
|status        |number|M|-|正の整数     |トリガーのステータス|
|severity      |string|M|-|-|トリガーの種別 [[一覧](#user-content-triggerseverity)]|
|lastChangeTime|string|M|-|65535byte以内|トリガーが最後に更新された時間|
|hostId        |number|M|-|正の整数     |監視サーバー内で設定されているホストID|
|hostName      |string|M|-|65535byte以内|トリガーが所属するサーバーのホスト名|
|brief         |string|M|-|65535byte以内|トリガーの概要|
|extendedInfo  |string|M|-|65535byte以内|上記の情報以外の必要な情報。主にWebUI上にデータを表示する際に用いられる|

```
{"jsonrpc":"2.0", "result":{"1":{"option":"UPDATED", "status":"OK", "severity":"INFO","lastChangeTime":"201503231758", "hostId":"1", "hostName":"exampleName", "brief":"example brief", "extendedInfo": "sample extended info"}},"id":1}
```

## 表

### errorCode

リクエストに成功した場合，送信したリクエストに応じたresultオブジェクトが返されます。
リクエストに失敗した場合，resultオブジェクトではなくerrorオブジェクトを返す必要があります。
このセクションではHatoholが独自に定義するJSON-RPCでのエラーコードについて解説します。
JSON-RPCのデフォルトエラーコードについては[[公式リファレンス](http://www.jsonrpc.org/specification#error_object)]をご覧ください。

|code|message|meaning|
|:--|:-------|:---|
|1  |NotGetAPIVersion|接続相手のAPIバージョンを取得できていません。|

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
|"SUCCESS"|リクエストの受け入れに成功しました|
|"ADDREV"|リクエストの受け入れが省略されました<br>リクエストの間隔が近い，といった理由でリクエストを受け入れない場合に使用します|
|"FAILED"|リクエストの受け入れに失敗しました|
