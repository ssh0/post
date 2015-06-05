% markdownの編集環境をいい感じに整えてみた
<div align="right">
<p>2015/05/26</p>
</div>

markdownでちょっとしたメモを残すことが多いのですが、やっぱりそのままで見やすいとは言え、簡単にHTMLにレンダリングして表示してやりたいって気持ちになることも多いでしょう。

ということで、僕がmarkdownエディタとして期待することは以下の項目。

- シンタックスハイライトが効く
- Vimみたいに動く
- リアルタイムプレビュー
- でもしょっちゅう更新して重くなってしまうようなものはナシ
- 当然数式も表示できる
- vim

以上の条件を満たすような環境を整えることができたと思うので、ここで紹介していきたいと思います。

前提条件は

- [vim-quickrun](https://github.com/thinca/vim-quickrun)
- [pandoc](http://pandoc.org/)
- firefox  
    (コマンドから新しいタブを開くことができ、ローカルファイルが変更された時自動で読み込むアドオン([firefox用"Auto Reload"](https://addons.mozilla.org/en-US/firefox/addon/auto-reload/?src=api))があれば何でもよい)

となっています。

## vim + quickrun + pandoc で幸せになれた。

まず、はじめにやったことは先人たちの作り上げたプラグインを入れて試してみることからでした。[previm](https://github.com/kannokanno/previm)、[vim-instant-markdown](https://github.com/suan/vim-instant-markdown)などを試してみて、使いやすいんだけど、自分でも何とか改善できないかなぁといった感じでした。特に数式やCSSでのスタイルの指定など、うまく設定すればなんとかなったのかもしれませんが、うまい解決策を見つけられませんでした。また、できるだけ最小構成でいきたい自分としては、これらは便利ではありましたが、ベストチョイスとは言えませんでした。そこで[thinca](http://d.hatena.ne.jp/thinca/)さんによる[vim-quickrun](https://github.com/thinca/vim-quickrun)を使って、変換と表示を行うようにしてみました。

このプラグインを用いれば、例えば今編集しているファイルのタイプにしたがって:QuickRunを実行することによって、そのコードのテストを走らせたり、自動でインデントや構文の調整を行ったり、LaTeXのコンパイルから表示まで行ったり、そして自分で作った任意の関数やコードを実行することが出来るのです。

自分の環境ではmarkdownからHTMLへの変換ツールとして[pandoc](http://pandoc.org/)が一番使いやすそうだったので、これを使ってmarkdownからHTMLに変換し、作成したHTMLをブラウザで見るようにすることにしました。


### 下準備

vimでのmarkdownの編集をするために、

- [vim-markdown](https://github.com/plasticboy/vim-markdown)  
    シンタックスハイライト、自動折りたたみなど
- vimrcに以下を追記  
```
set syntax=markdown
autocmd BufRead,BufNewFile *.mkd set filetype=markdown
autocmd BufRead,BufNewFile *.md set filetype=markdown
```

これで拡張子".md"もしくは".mkd"のファイルはファイルタイプmarkdownとして認識されるようになります。

### pandocを使ってHTML形式に変換、表示するシェルスクリプト

変換用のスクリプトは、以下のようにシェルスクリプトとして準備しました([mkdpreview](https://github.com/ssh0/bin/blob/master/mkdpreview))。

ここで、オプションは

- "-s"  
    引数がファイル名か文字列かによって、カレントディレクトリか/tmp以下にHTML形式のファイルを保存
- "-o (ファイル名)"  
    指定したファイル名でHTMLファイルを保存(sは自動で有効化)
- "-p"  
    Firefoxの新しいタブでHTMLファイルを表示する

のようになっています。

また、[pandocのDemo](http://pandoc.org/demos.html)を見てもらうと分かるように、自由にヘッダーとフッター、それからCSSファイルを指定することができるので、ここに好きなHTMLファイルやスタイルシートを指定しておきます(相対リンクだと/tmp以下に配置されたときに読み込まれないので、絶対パスにするか、Web上を参照させるのがいいと思います)。

それから、数式もLaTeX形式で書いていれば、pandocへ渡すオプションに`-s --mathjax`などと入れておけば、[MathJax](https://www.mathjax.org/)で数式に自動変換してくれます。MathJaxはオンラインで数式のレンダリングを行うので、オフラインで作業したい人は別の数式レンダリングのオプションを使うといいかもしれません。

```sh
#!/bin/sh
#

template='/home/shotaro/Workspace/blog/styles/stylesheet.css'
# template='/home/shotaro/Workspace/blog/styles/bootstrap-md.css'
# template='http://szk-engineering.com/markdown.css'
header='/home/shotaro/Workspace/blog/html/header.html'
footer='/home/shotaro/Workspace/blog/html/footer.html'

usage() {
    echo "Simply render markdown by pandoc."
    echo "Usage: $0 [OPTION]"
    echo ""
    echo "OPTION:"
    echo "  -s: Save html file in the source file dir."
    echo "  -o: Set the name of html file. ($0 -o bar.html foo.md)"
    echo "  -p: Preview HTML file in firefox."
    echo "  -h: Show this message."
    exit 1
}
save=false
output=""
preview=false

while getopts so:ph OPT
do
  case $OPT in
    "s" ) save=true ;;
    "o" ) save=true; output="${OPTARG}" ;;
    "p" ) preview=true ;;
    "h" ) usage ;;
  esac
done

shift $((OPTIND-1))

if [ -f "$1" ]; then
  dir="`cd $(dirname "$1"); pwd`"
  name=${1%.*}
else
  dir="$(pwd)"
  name="/tmp/$(cat /dev/urandom | tr -cd 'a-f0-9' | head -c 32)"
fi

if $save; then
  if [ "${output}" = "" ]; then
    output="${name}.html"
  fi
  pandoc -f markdown -S -t html5 -c "${template}" -B "${header}" -A "${footer}" \
    -s --mathjax -o "${output}" $@
else
  if [ ! -e "${name}.html" ];then
    exit 0
  else
    output="${name}.html"
    pandoc -f markdown -S -t html5 -c "${template}" -B "${header}" -A "${footer}" \
      -s --mathjax -o "${output}" $@
  fi
fi

if $preview; then
  firefox --new-tab "file://${output}"
fi
```


### 自作スクリプトをquickrunで動かす(もちろんキーマッピングもする)

次にvim-quickrunで作成したスクリプトが実行できるようにします。`~/.vim/ftplugin/markdown_quickrun.vim`に以下のように記述します。説明はのちほど。

- [markdown_quickrun.vim](https://github.com/ssh0/dotfiles/blob/master/vimfiles/ftplugin/markdown_quickrun.vim)


```vim
" markdown
let g:quickrun_config = {
\ 'markdown/normal' : {
\   'outputter' : 'error',
\   'outputter/error/error' : 'message',
\   'command' : 'mkdpreview',
\   'srcfile' : expand("%"),
\   'cmdopt' : '',
\   'exec': '%c %o %s',
\   },
\
\ 'markdown/visual' : {
\   'outputter' : 'error',
\   'outputter/error/error' : 'message',
\   'command' : 'mkdpreview',
\   'cmdopt' : '-p',
\   'exec': '%c %o %s',
\   },
\}

let g:quickrun_config['markdown'] = {
\ 'outputter' : 'error',
\ 'outputter/error/error' : 'message',
\ 'command' : 'mkdpreview',
\ 'cmdopt' : '-s -p',
\ 'exec': '%c %o %s',
\}


" QuickRun and view compile result quickly
nnoremap <F5> :QuickRun markdown/normal<CR>
vnoremap <F5> :QuickRun markdown/visual<CR>

autocmd BufWritePost,FileWritePost *.md QuickRun markdown/normal
```

このように書くと、vim-quickrunがこのファイル内を参照してくれるので、markdownファイルを開いた後、普通にノーマルモードで`:QuickRun`とすると、firefoxの新しいタブで現在のファイルがHTMLに変換されたものが表示されます(同名のHTMLファイルがディレクトリ内に作成されます)。それ以後はマークダウンファイルを上書きするたびにHTMLファイルが更新されます。F5を押せば手動でこの操作を行うことができます。また、ビジュアルモードで範囲を指定してF5キーを押せば、その範囲だけでレンダリングされ、/tmp以下に臨時ファイルが作られてそれをブラウザで表示して見ることができます。

また、もし同ディレクトリ内に同名のHTMLファイルがない場合には、F5キーを押したり、保存を行っても何も起こりません。つまり、自分で`:QuickRun`コマンドを実行しないと、これらのプレビュー表示などは行われないことになります。ここらへんの柔軟性を自分で持たせることができるので、自分でスクリプトを書いてquickrunで動かすというのも、一つの手だと思います。まぁ、自分がVimScriptがなかなか書けない&読めないというのもあるのですが。

それと、はじめの前提条件のところでも少し言及しましたが、この設定自体にはブラウザの操作は入っていないので、Firefoxだったら[Auto Reload](https://addons.mozilla.org/en-US/firefox/addon/auto-reload/?src=api)のようなアドオンが別途必要になります。他のプラグインなどではローカルサーバーをたてて、そこを見るようにしてこの問題を解決しているようですね。

## まとめ

ということで、はじめに示した条件を満たす環境を構築することができたのではないかと思います。quickrunの仕様をしっかりと理解した作りになっているかはわかりませんが、個人でシェルスクリプトを作りためてる人はこのようにしていけば簡単に自分の思った通りの挙動をさせることができて便利だと思います。

LaTeXの編集に関しても、latexmkを使った全体のコンパイルと部分コンパイルの設定を行ったので、そちらもまとめ次第こちらにアップしたいと思います。

お疲れ様でした。
