---
title: "Vimコマンドでパックマンを遊ぶ！Vim学習ゲームをGoで作ってみた👍"
emoji: "😶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, vim, ゲーム, 個人開発]
published: true
published_at: 2023-08-28 07:00
---

![pacvim](https://github.com/masahiro-kasatani/pacvim/blob/readme-images/files/readme.png?raw=true)
_The Go gopher was designed by Renée French._

https://github.com/masahiro-kasatani/pacvim

## 概要

個人開発の宣伝記事になります。
タイトルの通り、パックマンを Vim コマンドで操作するゲームを Go で作ってみました。

下記が本記事の対象読者です。

- Vim 操作を楽しく練習したい方
- Go で何かを作ってみたい方

## PacVim を遊びたい方へ

### PacVim の起動方法

- ダウンロード

```sh
git clone https://github.com/masahiro-kasatani/pacvim.git
```

- 起動
  - ダウンロードした中に以下のファイルがあるので、ダブルクリックで起動してください
    - Windows の場合：[pacvim/bin/win/pacvim.exe](https://github.com/masahiro-kasatani/pacvim/tree/master/bin/win)
    - Mac の場合：[pacvim/bin/mac/pacvim](https://github.com/masahiro-kasatani/pacvim/tree/master/bin/mac)

### PacVim のルール

PacVim はパックマンのルールを踏襲しています。

#### ゲーム画面

![ゲーム画面](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/screen.png)

:::message
目がチカチカするので、ターミナルやコマンドプロンプトを拡大表示して遊ぶことをオススメします。
:::

#### オブジェクトについて

| オブジェクト名 |                                                                                                                                                     表示　　　　                                                                                                                                                     | 補足説明                     |
| :------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: | :--------------------------- |
| りんご         |                                               ![りんご（未）](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/apple_1.png) ![りんご（済）](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/apple_2.png)                                                | 食べると緑色になります       |
| 毒             |                                                                                                           ![毒](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/poison.png)                                                                                                           | -                            |
| 障害物         | ![障害物１](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/wall_1.png) ![障害物２](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/wall_2.png) ![障害物３](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/wall_3.png) | -                            |
| プレイヤー     |                                                                                                       ![プレイヤー](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/player.png)                                                                                                       | -                            |
| 敵（ハンター） |                                                                                                        ![ハンター](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/hunter.png)                                                                                                        | -                            |
| 敵（ゴースト） |                                                                                                        ![ゴースト](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/ghost.png)                                                                                                         | 障害物をすり抜けられる敵です |

#### ゲームの状態について

| 状態           | 遷移条件                            |
| :------------- | :---------------------------------- |
| ステージクリア | すべてのりんごを食べる              |
| ステージ失敗   | 敵に捕まる or 毒を食べる            |
| ゲームクリア   | すべてのステージをクリアする        |
| ゲームオーバー | ライフが 0 の状態でステージ失敗する |

### プレイヤーの操作方法

|   キー    | 動作種別 | 動作                                                   |
| :-------: | :------- | :----------------------------------------------------- |
| `h`, `Nh` | `walk`   | 左へ 1 マス移動する（`Nh` の場合は N 回繰り返す）      |
| `j`, `Nj` | `walk`   | 下へ 1 マス移動する（`Nj` の場合は N 回繰り返す）      |
| `k`, `Nk` | `walk`   | 上へ 1 マス移動する（`Nk` の場合は N 回繰り返す）      |
| `l`, `Nl` | `walk`   | 右へ 1 マス移動する（`Nl` の場合は N 回繰り返す）      |
| `w`, `Nw` | `walk`   | 次の単語の先頭に移動する（`Nw` の場合は N 回繰り返す） |
| `e`, `Ne` | `walk`   | 次の単語の末尾に移動する（`Ne` の場合は N 回繰り返す） |
| `b`, `Nb` | `walk`   | 前の単語の先頭に移動する（`Nb` の場合は N 回繰り返す） |
|    `0`    | `jump`   | 現在の行の先頭に移動する                               |
|    `$`    | `jump`   | 現在の行の末尾に移動する                               |
|    `^`    | `jump`   | 現在の行の最初の単語の先頭に移動する                   |
|   `gg`    | `jump`   | 最初の行の最初の単語の先頭に移動する                   |
|    `G`    | `jump`   | 最後の行の最初の単語の先頭に移動する                   |
|   `NG`    | `jump`   | N 行目の行の最初の単語の先頭に移動する                 |
|    `q`    | -        | ゲームをやめる                                         |

#### 動作種別について

- `walk`

  - `walk` は目的地まで、 1 マスずつ一瞬で移動するイメージです。そのため、敵や障害物、りんごとの当たり判定が適用されます。一気にりんごを食べたいときに使いましょう。

    - 例： `w` を入力した場合

      ![walkの例](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/readme-w.gif)

- `jump`

  - `jump` は目的地まで、間を飛び越えて一瞬で到達するイメージです。そのため、敵や障害物、りんごとの当たり判定が適用されません。敵や障害物を避けて移動したいときに使いましょう。

    - 例： `$` を入力した場合

      ![jumpの例](https://raw.githubusercontent.com/masahiro-kasatani/pacvim/readme-images/files/readme-doller.gif)

## PacVim を開発したい方へ

### 開発用コマンド

```sh
make help
Usage:
    make <command>

Commands:
    fmt
        go fmt
    lint
        golangci-lint run
    deps
        go mod tidy
    test
        go test
    cover
        create cover.html
    build
        Make a macOS executable binary
    build-win
        Make a Windows executable binary
    clean
        Remove binary files
```

:::message
make をインストールしていない場合、[Makefile](https://github.com/masahiro-kasatani/pacvim/blob/master/Makefile) を参考にコマンドを実行してください。

- 例：MacOS でビルドをする場合
  - `go build -o bin/mac/pacvim .`

:::

### 実行用オプション

```sh
./pacvim -h
Usage of ./pacvim:
  -level int
    	Level at the start of the game. (default 1)
  -life int
    	Remaining lives. (default 2)
```

- 例：残機 5 でレベル 3 からスタートしたい場合
  - `go run . -level 3 -life 5`

### PacVim のカスタマイズ方法

それぞれコミット例を作成したので、参考にしてください。

#### ステージマップの追加方法

https://github.com/masahiro-kasatani/pacvim/commit/ab3afdd377e3ac83e0b05b279096f3bcbdd5a26f

#### 敵の種類の追加方法

https://github.com/masahiro-kasatani/pacvim/commit/6c5f88a32b7ffe73bd640717f0470407578c65d0

#### 敵の戦略の追加方法

https://github.com/masahiro-kasatani/pacvim/commit/b0f405ff0be4dc3143579536f89aa30c83c608b6

## PacVim 実装 Tips

ここからは、PacVim を実装する上で工夫した点を紹介します。

### 高階関数

Go では、関数は第一級関数であるため、高階関数（関数が引数として関数を取ったり、関数を返したりすること）を実装できます。このテクニックを今回は主に switch 文の可読性の向上や、重複処理の共通化のために利用しました。
以下に示す通り、キー入力に対する挙動が一行で簡潔に書けています。おかげさまで [gocyclo](https://goreportcard.com/report/github.com/masahiro-kasatani/pacvim#gocyclo) も 100%を維持できました。

- 抜粋

```go:player.go
func (p *player) action(ch rune, s stage) {
	switch ch {
	case 'w':
		p.moveByWord(p.toBeginningOfNextWord)
	case 'b':
		p.moveByWord(p.toBeginningPrevWord)
	case 'e':
		p.moveByWord(p.toEndOfCurrentWord)
	case '0':
		p.jumpOnCurrentLine(p.toLeftEdge)
	case '$':
		p.jumpOnCurrentLine(p.toRightEdge)
	case '^':
		p.jumpOnCurrentLine(p.toBeginningOfFirstWord)
	case 'g':
		p.jumpAcrossLine(p.toFirstLine, s, ch)
	case 'G':
		p.jumpAcrossLine(p.toLastLine, s, ch)
	}
}

func (p *player) moveByWord(fn func() bool) {
	if p.inputNum != 0 && p.inputG {
		p.initInput()
		fn()
	} else if p.inputNum != 0 {
		for i := 0; i < p.inputNum; i++ {
			if !fn() {
				break
			}
		}
	} else {
		fn()
	}
	p.initInput()
}

func (p *player) jumpOnCurrentLine(fn func()) {
	fn()
	p.judgeMoveResult()
	p.initInput()
}

func (p *player) jumpAcrossLine(fn func(stage), s stage, ch rune) {
	if ch == 'g' && !p.inputG {
		p.inputG = true
	} else if ch == 'G' || (ch == 'g' && p.inputG) {
		if p.inputNum == 0 {
			fn(s)
		} else {
			p.toSelectedLine(s)
		}
		p.judgeMoveResult()
		p.initInput()
	}
}
```

- `player.go`全文

https://github.com/masahiro-kasatani/pacvim/blob/master/player.go

### ビルダーパターン

敵の生成にはビルダーパターンを採用しました。構造体`enemy`のフィールド数が多く、コンストラクタ関数（ファクトリ関数）では可読性と拡張性に問題があると判断したためです。

デフォルトの敵を作りたい場合は以下のような形です。非常に分かりやすいですね。

```go
hunter := newEnemyBuilder().defaultHunter().build()
ghost := newEnemyBuilder().defaultGhost().build()
```

設定値をイジりたい場合は以下のような形です。これも直感的ですね。

```go
hunter := newEnemyBuilder().defaultHunter().speed(2).build()
ghost := newEnemyBuilder().defaultGhost().strategize(&assault{}).build()
```

提供側（`enemy.go`）はコード量が増えて少々複雑になってしまいますが、利用側は上記のように可読性に優れたシンプルなコードになります。ユーザーがゲームをカスタマイズすることも想定し、こちらの方が良いと判断しました。

### ストラテジーパターン

敵の戦略にはその名の通り、ストラテジーパターン（インタフェースの埋め込み）を採用しました。ストラテジーパターンはアルゴリズムの切り替えを簡単にします。要はポリモーフィズム（多態性）の実装ですね。
同じ`enemy`オブジェクトでも、種類（`hunter`か`ghost`か）や戦略（`assault`か`tricky`か）で振る舞いが異なります。
今回、種類による振る舞いの違いを関数`canMove`の埋め込み、戦略による振る舞いの違いをインタフェース`strategy`の埋め込みで実装しました。

- 抜粋

```go:enemy.go
type enemy struct {
	x            int
	y            int
	char         rune
	color        termbox.Attribute
	waitingTime  int
	oneActionInN int
	canMove      func(int, int) bool
	strategy
	underRune
}

type strategy interface {
	eval(p *player, x, y int) float64
}
type assault struct{}
type tricky struct{}

func (e *enemy) eval(p *player, x, y int) float64 {
	if !e.canMove(x, y) {
		// Returns a large enough value if it can't move
		return 1000
	}
	return e.strategy.eval(p, x, y)
}
func (s *assault) eval(p *player, x, y int) float64 {
	// Distance between two points
	return math.Sqrt(math.Pow(float64(p.y-y), 2) + math.Pow(float64(p.x-x), 2))
}
func (s *tricky) eval(p *player, x, y int) float64 {
	if random(0, 5) == 0 {
		return float64(random(0, 30))
	} else {
		return math.Sqrt(math.Pow(float64(p.y-y), 2) + math.Pow(float64(p.x-x), 2))
	}
}
```

- `enemy.go`全文

https://github.com/masahiro-kasatani/pacvim/blob/master/enemy.go

## 所感

ゴルーチンの勉強のために作り始めましたが、気づけば中々のボリュームになったので公開 + 宣伝させていただきました。敵の思考ロジックとかは専門外なので突き詰められませんでしたが、取り敢えず動くものが出来たので満足しています。

以上です。

## 参考

- C++版 PacVim

https://github.com/jmoon018/PacVim
