# はじめに

> 「そろそろ頃合い」セイウチは言った
> 「あれやこれやの積もる話」
>
> ルイス・キャロル
> 「不思議の国のアリス」

今回は盛りだくさんです！ まずは身近な例を使って，OpenFlowの動作モデルを説明します。これが理解できれば，OpenFlowの基本概念はバッチリです。次に，「トラフィック集計付きスイッチ」を実現するコントローラを実際に作ります。これはOpenFlowの重要な処理をすべて含んでいるので，応用するだけでさまざまなタイプのコントローラが作れるようになります。最後に，作成したコントローラをTremaの仮想ネットワーク上で実行します。すばらしいことに，Tremaを使えば開発から動作テストまでを開発マシン1台だけで完結できます！

では前置きはこのぐらいにして，まずはOpenFlowでスイッチを制御するしくみを理解しましょう。

# OpenFlowの動作モデル

OpenFlowの動作を現実世界にたとえると，製品の電話サポートサービスに似ています。

## 電話サポートの業務手順

友太郎（ゆうたろう）君は，エアコンが故障したので修理に出そうと考えました（図1）。電話サポートに問い合わせると，サポート係の葵（あおい）さんはエアコンの症状を聞き，手元のマニュアルに対処方法が載っている場合にはこれをすぐに教えてくれます。問題は，マニュアルに対処法が載っていない場合です。このようなときは少し時間がかかりますが，上司の宮坂主任にどうしたらよいか聞きます。そして，宮坂主任からの回答が得られたら，葵さんは友太郎君に折り返し電話をします。また，次からの同じ問い合わせにはすばやく答えられるようにするため，葵さんは教わった対処法を手元のマニュアルに追加しておきます。

簡単ですね？ 信じられないかもしれませんが，あなたはすでにOpenFlowの95%を理解したも同然なのです。

![電話サポートの業務手順](https://github.com/trema/Programming-Trema/raw/master/images/2_001.png)

図1　電話サポートの業務手順

## OpenFlowに置き換えると……

OpenFlowでは，お客さんがパケットを発生させるホスト，電話サポート係がスイッチ，上司がコントローラ，マニュアルがスイッチのフローテーブル（後述）に対応します（図2）。

![OpenFlowの動作モデル](https://github.com/trema/Programming-Trema/raw/master/images/2_002.png)
図2　OpenFlowの動作モデル

スイッチはホストからのパケットを受信すると，最初はその処理方法がわかりません。そこで，上司にあたるコントローラに問い合わせます。この問い合わせをpacket_inメッセージと呼びます。コントローラはこれを受け取ると，同様のパケットが届いた場合にスイッチでどう処理すべきか（パケットを転送する，書き換えるなど）を決めます。これをアクションと呼びます。そして「スイッチで処理すべきパケットの特徴」＋「アクション」の組（フローと呼びます）をスイッチのマニュアルに追加します。この命令をflow_modメッセージと呼び，スイッチのマニュアルをフローテーブルと呼びます。処理すべきパケットの特徴とアクションをフローテーブルに書いておくことで，以後，これに当てはまるパケットはスイッチ側だけですばやく処理できます。忘れてはならないのが，packet_inメッセージで上がってきた最初のパケットです。これはコントローラに上がってきて処理待ちの状態になっているので，packet_outメッセージで適切な宛先に転送してあげます。

電話サポートとの大きな違いは，フローテーブルに書かれたフローには期限があり，これを過ぎると消えてしまうということです。これは，「マニュアルに書かれた内容は徐々に古くなるので，古くなった項目は消す必要がある」と考えるとわかりやすいかもしれません。フローが消えるタイミングでコントローラにはflow_removedメッセージが送信されます。これには，あるフローに従ってパケットがどれだけ転送されたか──電話サポートの例で言うと，マニュアルのある項目が何回参照されたか──つまり，トラフィックの集計情報が記録されています。

それではしくみの話はこのぐらいにして，早速実践に移りましょう。もし途中でわからなくなったら，この節の頭から読み直してください。