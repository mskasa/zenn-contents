---
title: "vimコマンドでパックマンを操作するvim学習ゲーム「PacVim」をGoで作ってみた"
emoji: "😶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go,vim,ゲーム]
published: false
---

https://github.com/masahiro-kasatani/pacvim

## 遊び方

### Objects
| char | what it does |
|:-:|:-:|
| o  | food  |
| X  | poison  |
| G  | ghost  |

### Operation
| key | what it does |
|:-:|:-:|
| q  | quit the game  |
| h  | move left  |
| j  | move down  |
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

## 改良

## 実装

