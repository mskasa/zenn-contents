---
title: "vimコマンドでパックマンを操作するvim学習ゲーム「PacVim」をGoで作ってみた"
emoji: "😶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go,vim,ゲーム]
published: false
---

## 概要

元ネタの C++版は最後に参考として載せております。

https://github.com/masahiro-kasatani/pacvim

## 遊び方

### オブジェクト
| オブジェクト| 文字 | 説明 |
|:-|:-:|:-|
| りんご | o  | いくらでも食べられるおいしいやつ |
| 毒 | X  | 間違って食べちゃうと大変なやつ |
| 障害物 | -, &#124; | 通り抜けられない邪魔なやつ |
| プレイヤー | P  | りんごを食べたい欲張りなやつ |
| 敵（ハンター） | H  | とにかく追いかけてくるウザいやつ |
| 敵（ゴースト） | G  | 障害物をすり抜けてくる怖いやつ |

:::message
- プレイ時にはプレイヤーおよび敵はカーソルで表示されます
- 視覚的に分かりやすいのと、**敵は文字ではない**ことを強調するためです（詳細は後述）
:::

### 操作方法
下記の vimコマンドに対応しています。限りなく本来の動きに寄せたつもりです。

| key | what it does |
|:-:|:-:|
| h  | 左へ1マス移動する  |
| j  | 下へ1マス移動する  |
| k  | move up  |
| l  | move right  |
| w  | move forward to next word beginning  |
| e  | move forward to next word ending  |
| b  | move backward to next word beginning  |
| $  | move to the end of the line  |
| 0  | move to the beginning of the line  |
| gg/1G  | move to the beginning of the first line  |
| NG  | move to the beginning of the line given by N  |
| G  | move to the beginning of the last line  |
| ^  | move to the first word at the current line  |
| q  | ゲームをやめる  |

## カスタマイズ方法

### ステージを追加したい

### 敵の種類を追加したい

### 敵の動きを追加したい

## 実装 Tips

ここからは pacvim の実装自体に興味のある方向けになります。

## 参考

https://github.com/jmoon018/PacVim

C++版の PacVim です。
実装の参考にさせていただきました。
