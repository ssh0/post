# スクリーンショットを撮って日付のディレクトリに連番で保存するスクリプト

スクリーンショットを撮る機会はなかなか多いと思うのですが、皆さんはどのようにされているでしょうか。Linuxではshutterというアプリケーションがスクリーンショットを撮るのには使いやすいかなぁと思うんですが、保存する際には毎回どこに保存するか指定しなければならず、僕の場合日付ごとにファイルを分けたいので、少し不便でした。それで、今回シェルスクリプトで簡単に書いてみました。必要なパッケージはimagemagickです。




下のスクリプトを

```
sh ~/bin/screenshot.sh
```

のようにオプション無しで実行すると、`rootdir`に指定したディレクトリ(下の例の場合`$HOME/Workspace/blog`)の中に、今日の日付のディレクトリがない場合には今日の日付を名前にしたディレクトリを作成し、その中に"screen_001.jpg"のように名前をつけて画面全体のスクリーンショットを保存します。ディレクトリの中に、同じ形式で名前が付けられたファイルがあった際には、その数字を読み取って、一番数の大きいものよりさらに一つ大きい数字をつけて保存します。保存が終わったら`notify-send`で保存したことを通知します。Macでは他の通知スクリプトがあると思うので、適宜そちらを使えば良いかと思います。

また、基本的にキーボード・ショートカットにして使うことを考えているので、コマンドラインから使う場面はあまり考えていないのですが、一応オプションを与えることもできます。"d"で(日付のディレクトリのかわりに)保存するディレクトリを絶対パスで指定することができます。"o"で保存する画像ファイルの名前を指定することができます。

ちなみに僕の場合はキーボードのPrintScreenを押した時は画面全体、Shiftを押しながらPrintScreenを押した時には範囲指定でスクリーンショットを撮るようにしています。

``` sh: screenshot.sh
#!/bin/sh
#

get_dir=false
get_name=false
rootdir=$HOME/Workspace/blog

if [ $# -le 0 ]; then
    while getopts d:o: OPT
    do
      case $OPT in
        "d" ) get_dir=true
              dir="$OPTARG" ;;
        "o" ) get_name=true
              name="$OPTARG" ;;
          * ) exit 1 ;;
      esac
    done
fi

# スクリーンショットを保存するディレクトリを設定
if ! $get_dir; then
    if [ ! -e $rootdir ]; then
        echo "There is no directory named: $rootdir"
        exit 1
    fi
    daydir=`date +%Y-%m-%d`
    dir=$rootdir/$daydir
fi

# ファイル名を設定
if ! $get_name; then
    if [ ! -e $dir ]; then
        mkdir $dir
        i=1
    else
        i=`expr $(ls $dir | sed -n 's/screen_\([0-9]\{3\}\).jpg/\1/p' | tail -n 1) + 1`
    fi
    name=$(printf screen_%03d.jpg $i)
fi

import -window root -quality 0 $dir/$name
# 範囲選択 (別ファイルに分けてもよい)
# import -quality 0 $dir/$name

notify-send "Screenshot has been made" "saved: $dir/$name"

exit 0
```

ディレクトリ構造はこんな感じになっていきます。


