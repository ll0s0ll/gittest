# TimeManagerを使って、聴きたいラジオ番組が流れてくる、自分だけのラジオを作る

何時からはこの放送局のこの番組、その後はこっちの放送局のあの番組、、、  
様々なソースの聴きたい番組が、時間になると自動的に流れてくる、そんなラジオがあったらいいですよね。  
TimeManagerを使って、radiko、らじるらじる、NHKラジオの聞き逃しサービスをソースに作ってみます。

## 聴取する開始時刻と終了時刻をコントロールする
[TimeManager(tm)](https://github.com/ll0s0ll/TimeManager)は、プログラムの開始時刻と終了時刻を管理するプログラムです。  
このプログラムを使うと、任意のプログラムを指定時刻に実行して、終了させることができます。

radikoが聴取できるプログラムとしては、[play_radiko.sh](https://gist.github.com/ihsoy-s/5292735#file-play_radiko-sh)が公開されています。
このプログラムと、TimeManagerを組み合わせた、以下のようなコマンドを実行すると、
radikoを2018年3月16日の午前7時から180分だけ聴取することができます。
```
$ sh -c 'echo "1521151200:10800:THE GUY PERRYMAN SHOW" | tm set && play_radiko.sh INT'
```

また、TimeManagerでは、プログラムの実行時刻等が、スケジュールとして管理されます。  
スケジュールは他のプロセスからも参照できますので、なにもプログラムが実行されていない、空き時間を取得することもできます。  
空き時間に任意のプログラムを実行することで、番組の間を埋めることができますので、継続的に音声を流すことも出来るようになります。

らじるらじるを聴取できるプログラムとしては、[play_radiru.sh](https://github.com/ll0s0ll/play_radiru)があります。
下記の例では、NHKラジオ第一を空き時間に流します。
```
$ sh -c 'echo "0:0:NHKラジオ第一" | tm unoccupied | tm set && play_radiru.sh tokyo r1'
```
このように、プログラムの実行時間をコントロールすることで、好きな番組を聴取できるようになります。

## スケジューリングを自動化する
聴きたい番組を聴取できるようになりましたが、放送毎にプログラムを実行するのはなかなか大変です。  
そこで、スケジューリングしやすくしたプログラムが以下です。  
これらのプログラムは、繰り返し実行されますので、現在の放送が終わると、次の放送が自動的にスケジュールされます。

- [radiko_for_timemanager](https://github.com/ll0s0ll/radiko_for_timemanager)
play_radiko.shを、TimeManagerに登録しやすくしたプログラムです。
例えば、毎週月曜から金曜日の午前7時から10時に放送されるinterFMの番組を聴取したい場合は、以下のようなコマンドを実行します。  
```$ radiko_for_timemanager.py INT "0 7 * * 1-5" 10800 "THE GUY PERRYMAN SHOW"```

- [radiru_for_timemanager](https://github.com/ll0s0ll/radiru_for_timemanager)
play_radiru.shを、TimeManagerに登録しやすくしたプログラムです。
以下の例は、毎週土曜日の午前9時から午前11時に放送されるNHKFMの番組を聴取する場合です。
```$ radiru_for_timemanager.py tokyo fm "0 9 * * 6" 7200 "世界の快適音楽セレクション"```

- [nhk_radio_ondemand_for_timemanager](https://github.com/ll0s0ll/nhk_radio_ondemand_for_timemanager)
スケジュールの空き時間に、NHKラジオの聞き逃し番組を再生するプログラムです。  
以下の例では、空き時間に収まる番組をランダムに選んで再生します。
```$ nhk_radio_ondemand_for_timemanager.py -r```

これらのプログラムを聴取したい番組ごと実行しておけば、好きな番組が自然にRaspberry Piから流れてくるようになります。

## さらに自動化する
聴取したい番組の数が増えてくると、これらのプログラムをブートごとに実行するのは、なかなか大変です。
そこで、[Supervisor](http://supervisord.org)というプログラムを使います。

Supervisorは、プロセスの管理と監視をしてくれるプログラムです。
もし、エラーが起きた場合、自動で復旧してくれますし、ログもまとめてくれます。
プログラムの実行、停止が簡単にできる点も良いと思います。

以下は、supervisord.confの、プログラムの設定の一例です。データベースは5番を使用しています。
```
; THE GUY PERRYMAN SHOW
; Sun 8:10-10:00
[program:gps]
command=radiko_for_timemanager.py INT "10 8 * * 1-5" 6600 "THE GUY PERRYMAN SHOW"
autostart=true
environment=TM_DB_NUM="5"
stopasgroup=true
killasgroup=true
priority=500
```
ここまですれば、Raspberry Piをブートすれば、番組が流れてくるようになります。

## どの方法で放送するか?
音声はオーディオジャックやHDMIから出力するのが一般的ですが、  
GPIO経由の小型アンプとスピーカーを、Raspberry Piとともにオリジナルのケースに入れれば、全く新しいラジオができあがるでしょう。  
また、以下のようなソースを使うと、もっと魅力的なものができあがるかもしれません。

- インターネットストリーミング
私はこの方法で使っています。古いスマートフォンなどからでも聴取できるので、便利です。
ストリーミングするには、icecastとdarkiceを使います。  
ネットでは、Raspberry Piへの導入方法も公開されていますので、検索してみてください。  
音声はALSAを通して出力されますので、ALSAをストリーミングできるようになれば、OKです。

- FM電波
以前、[ALSAから出力された音声をpifmで再生する](https://ll0s0ll.wordpress.com/raspberrypi/alsa_to_pifm/)で、Raspberry PiをFMトランスミッターにする方法を紹介しました。  
今回の場合も同じ要領でFM電波に音声をのせることができます。  
いつもラジオで聴いている人にとっては、自分専用の放送局ができたように感じられるかもしれません。

以上

Last update 2018/03/20