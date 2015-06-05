[xmonadとxmobarの設定](http://qiita.com/ssh0/items/dedeebe249d0d6ba4975)と言うのを書いていたのですが、いろいろ改善点を直しているうちに、追記では収まらなくなていたので、新たにエントリーを書くことにしました。

xmoadを含むタイリングウィンドウマネージャの魅力については、他の人も多く語っているので、少しググってみると、その魅力に惹かれる人はいるんじゃないかなと思います。今回はxmonadとはなんぞや、といった話はせずに、今xmonadを使っている人向けに設定ファイルを晒す程度のものです。

また、当人はhaskellの知識はなく、stackoverflowやメーリングリストの情報から設定ファイルを書いているのであしからず。

#xmonadの設定


```haskell
import qualified Data.Map as M

import XMonad
import qualified XMonad.StackSet as W  -- myManageHookShift

import Control.Monad (liftM2)          -- myManageHookShift
import System.IO                       -- for xmobar

import XMonad.Actions.WindowGo
import XMonad.Actions.FloatKeys
import XMonad.Actions.CycleWS
import qualified XMonad.Actions.FlexibleResize as Flex  -- Resize floating windows from any corner
import XMonad.Hooks.DynamicLog         -- for xmobar
import XMonad.Hooks.EwmhDesktops
import XMonad.Hooks.FadeWindows
import XMonad.Hooks.ManageDocks        -- avoid xmobar area
import XMonad.Hooks.Place
import XMonad.Layout
import XMonad.Layout.DecorationMadness
import XMonad.Layout.DragPane          -- see only two window
import XMonad.Layout.Gaps
import XMonad.Layout.Grid
import XMonad.Layout.IM
import qualified XMonad.Layout.Magnifier as Mag -- this makes window bigger
import XMonad.Layout.MultiToggle
import XMonad.Layout.MultiToggle.Instances
import XMonad.Layout.Named
import XMonad.Layout.NoBorders         -- In Full mode, border is no use
import XMonad.Layout.PerWorkspace      -- Configure layouts on a per-workspace
import XMonad.Layout.ResizableTile     -- Resizable Horizontal border
import XMonad.Layout.SimplestFloat
import XMonad.Layout.Spacing           -- this makes smart space around windows
import XMonad.Layout.Tabbed
import XMonad.Layout.ThreeColumns      -- for many windows
import XMonad.Layout.ToggleLayouts     -- Full window at any time
import XMonad.Util.EZConfig            -- removeKeys, additionalKeys
import XMonad.Util.Run(spawnPipe)      -- spawnPipe, hPutStrLn
import XMonad.Util.Run

import Graphics.X11.ExtraTypes.XF86

myWorkspaces = ["  main  ", "  browser  ", "  float  ", "  work  ", "  tray  "]
modm = mod4Mask

-- Color Setting
colorBlue      = "#90caf9"
colorGreen     = "#a5d6a7"
colorRed       = "#ef9a9a"
colorGray      = "#9e9e9e"
colorWhite     = "#ffffff"
colorGrayAlt   = "#eceff1"
colorNormalbg  = "#212121"
colorfg        = "#9fa8b1"

-- Border width
borderwidth = 0

-- Define keys to remove
keysToRemove x =
    [
        -- Unused gmrun binding
          (modm .|. shiftMask, xK_p)
        -- Unused close window binding
        , (modm .|. shiftMask, xK_c)
        , (modm .|. shiftMask, xK_Return)
    ]

-- Delete the keys combinations we want to remove.
strippedKeys x = foldr M.delete (keys defaultConfig x) (keysToRemove x)

main :: IO ()

main = do
    wsbar <- spawnPipe myWsBar
    xmonad $ defaultConfig
       { borderWidth        = borderwidth
       , terminal           = "urxvt"
       , focusFollowsMouse  = False
       , normalBorderColor  = "#212121"
       , focusedBorderColor = colorWhite
       , startupHook        = myStartupHook
       , manageHook         = placeHook myPlacement <+>
                              myManageHookShift <+>
                              myManageHookFloat <+>
                              manageDocks
        -- any time Full mode, avoid xmobar area
       , layoutHook         = toggleLayouts (noBorders Full) $
                              avoidStruts $
                              onWorkspace "  float  " simplestFloat $
                              myLayout
        -- xmobar setting
       , logHook            = myLogHook wsbar
       , handleEventHook    = fadeWindowsEventHook
       , workspaces         = myWorkspaces
       , modMask            = modm
       , mouseBindings      = newMouse
       }

       `additionalKeys`
       [ ((modm                , xK_h      ), sendMessage Shrink)
       , ((modm                , xK_l      ), sendMessage Expand)
       , ((modm                , xK_f      ), sendMessage ToggleLayout)
       , ((modm .|. shiftMask  , xK_f      ), withFocused (keysMoveWindow (-borderwidth, -borderwidth)))
       , ((modm                , xK_z      ), sendMessage MirrorShrink)
       , ((modm                , xK_a      ), sendMessage MirrorExpand)
       , ((modm                , xK_Right  ), nextWS ) -- go to next workspace
       , ((modm                , xK_Left   ), prevWS ) -- go to prev workspace
       , ((modm .|. shiftMask  , xK_Right  ), shiftToNext)
       , ((modm .|. shiftMask  , xK_Left   ), shiftToPrev)
       , ((modm .|. controlMask, xK_Right  ), withFocused (keysMoveWindow (30, 0)))
       , ((modm .|. controlMask, xK_Left   ), withFocused (keysMoveWindow (-30, 0)))
       , ((modm .|. controlMask, xK_Up     ), withFocused (keysMoveWindow (0, -30)))
       , ((modm .|. controlMask, xK_Down   ), withFocused (keysMoveWindow (0, 30)))
       , ((modm                , xK_comma  ), sendMessage Mag.MagnifyLess) -- smaller window
       , ((modm                , xK_period ), sendMessage Mag.MagnifyMore) -- bigger window
       , ((modm                , xK_j      ), windows W.focusDown)
       , ((modm                , xK_k      ), windows W.focusUp)
       , ((modm .|. shiftMask  , xK_j      ), windows W.swapDown)
       , ((modm .|. shiftMask  , xK_k      ), windows W.swapUp)
       , ((modm .|. shiftMask  , xK_m      ), windows W.shiftMaster)
       , ((modm                , xK_w      ), nextScreen) ]

       `additionalKeys`
       [ ((modm .|. m, k), windows $ f i)
         | (i, k) <- zip myWorkspaces
                     [ xK_exclam, xK_at, xK_numbersign
                     , xK_dollar, xK_percent, xK_asciicircum
                     , xK_ampersand, xK_asterisk, xK_parenleft
                     , xK_parenright
                     ]
         , (f, m) <- [(W.greedyView, 0), (W.shift, shiftMask)]
       ]

       `additionalKeys`
       [ ((mod1Mask .|. controlMask, xK_l      ), spawn "xscreensaver-command -lock")
       , ((mod1Mask .|. controlMask, xK_t      ), spawn "bash /home/shotaro/bin/toggle_compton.sh")
       , ((modm                    , xK_Return ), spawn "urxvt")
       , ((modm                    , xK_c      ), kill) -- %! Close the focused window
       , ((modm                    , xK_p      ), spawn "exe=`dmenu_run -b -nb '#009688' -nf '#ffffff' -sb '#ffffff' -sf '#000000'` && exec $exe")
       , ((mod1Mask .|. controlMask, xK_f      ), spawn "python /home/shotaro/Workspace/python/web_search/websearch.py")
       , ((0                       , 0x1008ff14), spawn "sh /home/shotaro/bin/cplay.sh")
       , ((0                       , 0x1008ff13), spawn "amixer -D pulse set Master 1%+ && paplay /usr/share/sounds/freedesktop/stereo/audio-volume-change.oga")
       , ((0                       , 0x1008ff11), spawn "amixer -D pulse set Master 1%- && paplay /usr/share/sounds/freedesktop/stereo/audio-volume-change.oga")
       , ((0                       , 0x1008ff12), spawn "amixer -D pulse set Master toggle")
        -- Brightness Keys
       , ((0                       , 0x1008FF02), spawn "xbacklight + 10 -time 100 -steps 5")
       , ((0                       , 0x1008FF03), spawn "xbacklight - 10 -time 100 -steps 5")
       , ((0                       , 0xff61    ), spawn "sh /home/shotaro/bin/screenshot.sh")
       , ((shiftMask               , 0xff61    ), spawn "sh /home/shotaro/bin/screenshot_select.sh")
       , ((modm     .|. controlMask, xK_h      ), spawn "sh /home/shotaro/bin/xte-left.sh")
       , ((modm     .|. controlMask, xK_l      ), spawn "sh /home/shotaro/bin/xte-right.sh")
       , ((modm     .|. controlMask, xK_j      ), spawn "sh /home/shotaro/bin/xte-down.sh")
       , ((modm     .|. controlMask, xK_k      ), spawn "sh /home/shotaro/bin/xte-up.sh")
       , ((modm     .|. controlMask, xK_Return ), spawn "sh /home/shotaro/bin/xte-click.sh")

       ]

-- Handle Window behaveior
myLayout = (spacing 16 $ ResizableTall 1 (1/100) (1/2) [])
             |||  (spacing 16 $ ThreeCol 1 (1/100) (16/35))
             |||  (spacing 16 $ ResizableTall 2 (1/100) (1/2) [])
--             |||  Mag.magnifiercz 1.1 (spacing 6 $ GridRatio (4/3))

-- Start up (at xmonad beggining), like "wallpaper" or so on
myStartupHook = do
        spawn "gnome-settings-daemon"
        spawn "nm-applet"
        spawn "gnome-sound-applet"
        spawn "xscreensaver -no-splash"
        spawn "/home/shotaro/.dropbox-dist/dropboxd"
        spawn "feh --bg-fill '/home/shotaro/Downloads/20 Free Modern Backgrounds/20 Low-Poly Backgrounds/Low-Poly Backgrounds (2).JPG'"
        spawn "bash /home/shotaro/bin/toggle_compton.sh"
        -- spawn "compton -b --config /home/shotaro/.config/compton/compton.conf"

-- some window must created there
myManageHookShift = composeAll
            -- if you want to know className, type "$ xprop|grep CLASS" on shell
            [ className =? "Firefox"       --> mydoShift "  browser  "
            , className =? "Google-chrome" --> mydoShift "  work  "

            ]
             where mydoShift = doF . liftM2 (.) W.greedyView W.shift

-- new window will created in Float mode
myManageHookFloat = composeAll
            [ className =? "Gimp"             --> doFloat,
              className =? "mplayer2"         --> doFloat,
              className =? "Tk"               --> doFloat,
              className =? "Display.im6"      --> doFloat,
              className =? "Shutter"          --> doFloat,
              className =? "Websearch.py"     --> doFloat,
              className =? "Plugin-container" --> doFloat,
              title     =? "Speedbar"         --> doFloat]


myLogHook h = dynamicLogWithPP $ wsPP { ppOutput = hPutStrLn h }

myWsBar = "xmobar /home/shotaro/.xmonad/xmobarrc"

wsPP = xmobarPP { ppOrder           = \(ws:l:t:_)  -> [ws,t]
                , ppCurrent         = xmobarColor  colorGreen    colorNormalbg
                , ppUrgent          = xmobarColor  colorWhite    colorNormalbg
                , ppVisible         = xmobarColor  colorWhite    colorNormalbg
                , ppHidden          = xmobarColor  colorWhite    colorNormalbg
                , ppHiddenNoWindows = xmobarColor  colorfg       colorNormalbg
                , ppTitle           = xmobarColor  colorGreen    colorNormalbg
                , ppOutput          = putStrLn
                , ppWsSep           = ""
                , ppSep             = " : "
                }

-- Right click is used for resizing window
myMouse x = [ ((modm, button3), (\w -> focus w >> Flex.mouseResizeWindow w)) ]
newMouse x = M.union (mouseBindings defaultConfig x) (M.fromList (myMouse x))

myPlacement = fixed (0.5, 0.5)

```


設定ファイルに書いているキーバインドは以下

<table class="table table-condensed table-bordered">
  <tr><td><Strong>キー</Strong></td><td><Strong>やっていること</Strong></td></tr>
  <tr><td>Ctrl + Alt + l</td><td>スクリーンをロック</td></tr>
  <tr><td>Super + Return</td><td>端末urxvtを起動</td></tr>
  <tr><td>Super + c</td><td>フォーカスのあるウィンドウを削除</td></tr>
  <tr><td>Super + p</td><td>アプリケーションランチャーdmenuを起動。オプションは色の設定</td></tr>
  <tr><td>Ctrl + Alt + f</td><td> [以前に作成した簡易検索バーwebsearch.py](http://qiita.com/ssh0/items/7e70c8f72a0fa03df4b0)を起動</td></tr>
  <tr><td>音量アップボタン(0x1008ff13)</td><td>音量を上げてunity環境と同じ音でフィードバック</td></tr>
  <tr><td>音量ダウンボタン(0x1008ff11)</td><td>音量を下げてunity環境と同じ音でフィードバック</td></tr>
  <tr><td>ミュートボタン(0x1008ff12)</td><td>音量をミュート</td></tr>
  <tr><td>輝度アップボタン(0x1008FF02)</td><td>xbacklightを使って画面の明るさを上げる</td></tr>
  <tr><td>輝度ダウンボタン(0x1008FF03)</td><td>xbacklightを使って画面の明るさを下げる</td></tr>
  <tr><td>Super + f</td><td>全画面表示にするかの切り替え</td></tr>
  <tr><td>Super + l</td><td>マスター領域の横幅を大きくする</td></tr>
  <tr><td>Super + a</td><td>レイアウトがResizableTallのとき、領域の縦幅を小さくする</td></tr>
  <tr><td>Super + z</td><td>レイアウトがResizableTallのとき、領域の縦幅を大きくする</td></tr>
  <tr><td>Super + h</td><td>マスター領域の横幅を小さくする</td></tr>
  <tr><td>Super + Ctrl + 矢印キー</td><td>フォーカスのあるウィンドウを動かす</td></tr>
  <tr><td>Super + Ctrl + h</td><td>マウスポインタを左に動かす[xte-left.sh](https://github.com/ssh0/bin/blob/master/xte-left.sh)</td></tr>
  <tr><td>Super + Ctrl + j</td><td>マウスポインタを下に動かす[xte-down.sh](https://github.com/ssh0/bin/blob/master/xte-down.sh)</td></tr>
  <tr><td>Super + Ctrl + k</td><td>マウスポインタを上に動かす[xte-up.sh](https://github.com/ssh0/bin/blob/master/xte-up.sh)</td></tr>
  <tr><td>Super + Ctrl + l</td><td>マウスポインタを右に動かす[xte-right.sh](https://github.com/ssh0/bin/blob/master/xte-right.sh)</td></tr>
  <tr><td>Super + Ctrl + Return</td><td>マウスの左クリックをエミュレート[xte-click.sh](https://github.com/ssh0/bin/blob/master/xte-click.sh)</td></tr>
  <tr><td>Super + Right</td><td>次(右)のワークスペースに移動する</td></tr>
  <tr><td>Super + Left</td><td>前(左)のワークスペースに移動する</td></tr>
  <tr><td>Super + Shift + Right</td><td>フォーカスのあるウィンドウを次のワークスペースに移動</td></tr>
  <tr><td>Super + Shift + Left</td><td>フォーカスのあるウィンドウを前にワークスペースに移動</td></tr>
  <tr><td>Super + j</td><td>フォーカスを次のウィンドウに移す</td></tr>
  <tr><td>Super + k</td><td>フォーカスを前のウィンドウに移す</td></tr>
  <tr><td>Super + Shift + j</td><td>フォーカスのあるウィンドウを次のタイル配置に移す</td></tr>
  <tr><td>Super + Shift + k</td><td>フォーカスのあるウィンドウを前のタイル配置に移す</td></tr>
  <tr><td>Super + m</td><td>フォーカスをマスター領域のウィンドウに移す</td></tr>
  <tr><td>Super + Shift + m</td><td>フォーカスのあるウィンドウをマスター領域に移す</td></tr>
  <tr><td>Super + w</td><td>次のスクリーンにフォーカスを移す</td></tr>
  <tr><td>Ctrl + Alt + t</td><td>コンポジットマネージャ[compton](https://github.com/chjj/compton)の有効化、無効化の切り替え[toggle_compton.sh](https://github.com/ssh0/bin/blob/master/toggle_compton.sh)</td></tr>
  <tr><td>PrintScreen(0xff61)</td><td>画面全体のスクリーンショットを撮る[screenshot.sh](https://github.com/ssh0/bin/blob/master/screenshot.sh)</td></tr>
  <tr><td>Shift + PrintScreen(0xff61)</td><td>領域を選択してスクリーンショットを撮る[screenshot_select.sh](https://github.com/ssh0/bin/blob/master/screenshot_select.sh)</td></tr>
  <tr><td>再生・ポーズボタン(0x1008ff14)</td><td>音楽再生アプリケーション[cmus](https://cmus.github.io/)の起動or再生／ポーズ[cplay.sh](https://github.com/ssh0/bin/blob/master/cplay.sh)</td></tr>
</table>


myLayoutで指定した3つのレイアウトをSuper+Spaceで切り替えることができます。

myStartupHookで、xmonadの起動時に呼び出すプログラムを設定しています。

myManageHookShiftで、特定のウィンドウは指定したワークスペースで開くように設定します。

myManageHookFloatで、特定のウィンドウはフロート表示するように設定します。


##スクリーンショット

![](/home/shotaro/Workspace/blog/2015-03-10/screen_002.jpg)

#xmobarの設定

~/.xmonad/xmobarrcの中身は以下のようになっています。
アイコンはxbm形式であるものを使う必要がありますが、僕は[sm4tik-icon-pack](https://github.com/gmmeyer/awesome-dangerzone/tree/master/icons/sm4tik-icon-pack)を使わせてもらってます。


```haskell
-- -*- mode:haskell -*-
Config { font = "xft:TakaoGothic:size=10"
       , bgColor = "#212121"
       , fgColor = "#757575"
       , position = TopSize C 100 26
       , lowerOnStart = False
       , overrideRedirect = False
       , border = NoBorder
       , borderColor = "#26a69a"
       , commands = [ Run Network "wlan0" [ "-t"       , "<icon=/home/shotaro/.icons/sm4tik-icon-pack/xbm/net_down_03.xbm/><rx>  <icon=/home/shotaro/.icons/sm4tik-icon-pack/xbm/net_up_03.xbm/><tx>   "
                                          , "-L"       , "40"
                                          , "-H"       , "200"
                                          , "-m"       , "3"
                                          , "--normal" , "#ffffff"
                                          , "--high"   , "#a5d6a7"
                                          ] 10
                    , Run Network "usb0"  [ "-t"       , "<icon=/home/shotaro/.icons/sm4tik-icon-pack/xbm/net_down_03.xbm/><rx>  <icon=/home/shotaro/.icons/sm4tik-icon-pack/xbm/net_up_03.xbm/><tx>   "
                                          , "-L"       , "40"
                                          , "-H"       , "200"
                                          , "-m"       , "3"
                                          , "--normal" , "#ffffff"
                                          , "--high"   , "#a5d6a7"
                                          ] 10
                    , Run MultiCpu        [ "-t"       , "<icon=/home/shotaro/.icons/sm4tik-icon-pack/xbm/cpu.xbm/> <total0>.<total1>.<total2>.<total3>   "
                                          , "-L"       , "40"
                                          , "-H"       , "85"
                                          , "-m"       , "2"
                                          , "--normal" , "#ffffff"
                                          , "--high"   , "#f44336"
                                          ] 10

                    , Run Memory          [ "-t"       , "<icon=/home/shotaro/.icons/sm4tik-icon-pack/xbm/mem.xbm/> <usedratio>%   "
                                          , "-L"       , "40"
                                          , "-H"       , "90"
                                          , "-m"       , "2"
                                          , "--normal" , "#ffffff"
                                          , "--high"   , "#f44336"
                                          ] 10
                    , Run BatteryP        ["CMB1"]
                                          [ "-t"       , "<icon=/home/shotaro/.icons/sm4tik-icon-pack/xbm/bat_full_02.xbm/> <acstatus>   "
                                          , "-L"       , "20"
                                          , "-H"       , "80"
                                          , "--low"    , "#f443c6"
                                          , "--normal" , "#ffffff"
                                          , "--"
                                                , "-o" , "<left>% (<timeleft>)"
                                                , "-O" , "Charging <left>%"
                                                , "-i" , "<left>%"
                                          ] 50
                    , Run Date "%a %m/%d %H:%M" "date" 10
                    , Run StdinReader
                    ]
       , sepChar = "%"
       , alignSep = "}{"
       , template = " %StdinReader% }{ %multicpu%%memory%%wlan0%%battery%<fc=#fff59d>%date%</fc>   "
       }

```

## スクリーンショット



![](/home/shotaro/Workspace/blog/2015-03-10/screen_003.jpg)

![](/home/shotaro/Workspace/blog/2015-03-10/screen_004.jpg)


