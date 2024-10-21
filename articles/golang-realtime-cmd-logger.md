---
title: "ログ出力のリアルタイム化とタイムアウト処理を改善するGo言語コードの実装"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, sre]
published: false
---

## はじめに
システムの信頼性やトラブルシューティングの観点から、ログ出力は非常に重要な要素です。

特に、リアルタイムにログを出力する仕組みは、長時間のバッチ処理や複雑なスクリプト実行時における信頼性向上に役立ちます。

しかし、単純な `exec.Command` を用いた実装では、出力が全て蓄積され、一度にまとめて表示されることがあり、開発者や運用担当者がリアルタイムで処理状況を把握できず、作業効率が低下してしまいます。

本記事では、以下の2つのコードを比較し、ログ出力のリアルタイム化と、出力が途絶えた際のタイムアウト機能について解説します。

:::message
Go Playgroundでは、シェルコマンドの実行や外部コマンドの呼び出しをサポートしていないため、本記事のサンプルコードを試す場合はローカル環境で実行してください。
https://github.com/mskasa/go-realtime-cmd-logger
:::

## 改善前のコード：一度にログが出力される
まずは改善前のコードを見てみましょう。

```go:改善前のコード
func main() {
	ctx := context.Background()
	cmdName := "bash"
	args := []string{"-c", "for i in {1..5}; do echo \"output $i\"; sleep 3; done"}
	cmd := exec.CommandContext(ctx, cmdName, args...)
	cmd.Dir = "."

	slog.Info(fmt.Sprintf("cmd: %s %v, path: %s", cmdName, args, cmd.Dir))
	out, err := cmd.CombinedOutput()
	if err != nil {
		slog.Error(fmt.Sprintf("Error: %v", err))
	}
	slog.Info(string(out))
}
```

このコードでは、`exec.CommandContext` を利用してシェルコマンドを実行し、`CombinedOutput()` によって標準出力と標準エラー出力を取得しています。

しかし、問題点として、コマンド実行が完了するまでログは蓄積され、最後にまとめて出力されるため、リアルタイム性が損なわれます。

例えば、コマンドの実行が遅い場合や長時間の処理が発生した際、ログ出力が遅延し、開発者や運用担当者は処理状況をリアルタイムで把握できなくなります。

また、エラーが発生した場合も、ログが一度にまとめて出力されるため、原因の特定が遅れる可能性があります。

このコードを実行すると、以下のように15秒の実行後、蓄積されたログが一度に出力されます。

```text:実行結果の例
2024/10/18 11:09:32 INFO cmd: bash [-c for i in {1..5}; do echo "output $i"; sleep 3; done], path: .
2024/10/18 11:09:47 INFO output 1
output 2
output 3
output 4
output 5
```

![](/images/golang-realtime-cmd-logger/before.gif)

## 改善後のコード：リアルタイムログ出力とタイムアウト機能
次に、改善後のコードを見ていきます。

このコードでは、出力がリアルタイムに表示され、さらに出力が途絶えた場合には自動的にタイムアウトする機能を追加しています。

```go
func ShellExecWithArgs(ctx context.Context, cmdName string, args []string, dir string, timeout time.Duration) error {
	cmd := exec.CommandContext(ctx, cmdName, args...)
	cmd.Dir = dir

	slog.Info(fmt.Sprintf("cmd: %s %v, path: %s", cmdName, args, cmd.Dir))

	return executeCommand(ctx, cmd, timeout)
}

func executeCommand(ctx context.Context, cmd *exec.Cmd, timeout time.Duration) error {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	timer := time.AfterFunc(timeout, cancel)

	stdout, err := cmd.StdoutPipe()
	if err != nil {
		return fmt.Errorf("StdoutPipe: %w", err)
	}
	stderr, err := cmd.StderrPipe()
	if err != nil {
		return fmt.Errorf("StderrPipe: %w", err)
	}

	if err := cmd.Start(); err != nil {
		return fmt.Errorf("cmd.Start: %w", err)
	}

	stdoutScanner := bufio.NewScanner(stdout)
	stdoutChan := make(chan string)
	stdoutDone := make(chan bool)
	stderrScanner := bufio.NewScanner(stderr)
	stderrChan := make(chan string)
	stderrDone := make(chan bool)

	go streamReader(stdoutScanner, stdoutChan, stdoutDone, timer, timeout)
	go streamReader(stderrScanner, stderrChan, stderrDone, timer, timeout)

	isRunning := true
	for isRunning {
		select {
		case <-stdoutDone:
			isRunning = false
		case line := <-stdoutChan:
			slog.Info(line)
		case line := <-stderrChan:
			slog.Error(line)
		case <-ctx.Done():
			return fmt.Errorf("command cancelled due to timeout or context cancellation")
		}
	}

	if err := cmd.Wait(); err != nil {
		return fmt.Errorf("failed to execute command: %w", err)
	}

	return nil
}

func streamReader(scanner *bufio.Scanner, outputChan chan string, doneChan chan bool, timer *time.Timer, timeout time.Duration) {
	buf := make([]byte, 4096)
	scanner.Buffer(buf, 65536)

	scanner.Split(splitFunc)
	defer close(outputChan)
	defer close(doneChan)
	for scanner.Scan() {
		outputChan <- scanner.Text()
		timer.Reset(timeout)
	}
	doneChan <- true
}

func splitFunc(data []byte, atEOF bool) (advance int, token []byte, err error) {
	if atEOF && len(data) == 0 {
		return 0, nil, nil
	}

	for i := 0; i < len(data); i++ {
		switch data[i] {
		case '\n':
			if i > 0 && data[i-1] == '\r' {
				return i + 1, data[:i-1], nil // CRLF
			}
			return i + 1, data[:i], nil // LF
		case '\r':
			if i == len(data)-1 || data[i+1] != '\n' {
				return i + 1, data[:i], nil // CR
			}
		}
	}

	if atEOF {
		return len(data), data, nil
	}
	return 0, nil, nil
}

func main() {
	ctx := context.Background()
	timeout := 5 * time.Second

	err := ShellExecWithArgs(ctx, "bash", []string{"-c", "for i in {1..5}; do echo \"output $i\"; sleep 3; done"}, ".", timeout)
	if err != nil {
		slog.Error(fmt.Sprintf("Error: %v", err))
	}
}
```

### 改善点1：リアルタイムなログ出力
改善後のコードでは、標準出力および標準エラー出力をリアルタイムに逐次処理しています。

`bufio.Scanner` を用いて標準出力・標準エラーを読み取り、各行を即座に表示します。

これにより、ログが蓄積されず、リアルタイムにユーザーにフィードバックを返すことが可能となります。

### 改善点2：タイムアウト機能
リアルタイムログ出力は、正確なタイムアウト機能にも寄与します。

「エラーは報告されないものの、処理が停止している」という状況を、「ログが出力されていない」ということで検知が可能になります。

具体的には、`context.WithCancel` を使用してコマンドのキャンセル機能を実装し、`time.AfterFunc` で指定した時間内に出力がない場合、タイマーがキャンセルをトリガーします。

また、出力があるたびにタイマーがリセットされるため、正常な処理が続いている限り、タイムアウトは発生しません。

### 結果
このアプローチにより、改善前のコードで発生していた「ログが一度にまとめて出力される」という問題が解消され、さらにタイムアウト機能を追加することで、プロセスの安定性とレスポンス性が大幅に向上しました。

```text:実行結果の例
2024/10/18 11:14:56 INFO cmd: bash [-c for i in {1..5}; do echo "output $i"; sleep 3; done], path: .
2024/10/18 11:14:56 INFO output 1
2024/10/18 11:14:59 INFO output 2
2024/10/18 11:15:02 INFO output 3
2024/10/18 11:15:05 INFO output 4
2024/10/18 11:15:08 INFO output 5
```

![](/images/golang-realtime-cmd-logger/after_info.gif)

## 改善後のコード詳細解説
ここでは、改善後のコードに関する詳細な解説を行います。

リアルタイムなログ出力機能と、出力が途絶えた場合のタイムアウト処理の仕組みについて、具体的なコードの動作を説明します。

### 1. ShellExecWithArgs 関数
この関数は、シェルコマンドを実行するエントリーポイントです。

```go
func ShellExecWithArgs(ctx context.Context, cmdName string, args []string, dir string, timeout time.Duration) error {
	cmd := exec.CommandContext(ctx, cmdName, args...)
	cmd.Dir = dir

	slog.Info(fmt.Sprintf("cmd: %s %v, path: %s", cmdName, args, cmd.Dir))

	return executeCommand(ctx, cmd, timeout)
}
```

#### 引数の説明
- `ctx`: コマンド実行時にタイムアウトやキャンセルを管理するためのコンテキスト
- `cmdName`: 実行するコマンドの名前
- `args`: コマンドに渡す引数のリスト
- `dir`: コマンドを実行するディレクトリ
- `timeout`: 出力が途絶えた場合にキャンセルされるまでの時間

### 2. executeCommand 関数
この関数では、コマンドの実行とリアルタイムのログ出力、タイムアウト機能の両方を管理します。

```go
func executeCommand(ctx context.Context, cmd *exec.Cmd, timeout time.Duration) error {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	timer := time.AfterFunc(timeout, cancel)
```

#### コンテキストとキャンセルの設定
`context.WithCancel` により、新しいコンテキストが生成され、`cancel` 関数でコマンド実行を中断できるようにしています。

`time.AfterFunc` は、指定された `timeout` 時間後に `cancel` 関数を自動的に呼び出すタイマーを設定します。

このタイマーが、出力が途絶えた際のタイムアウト処理を実現しています。
```go
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		return fmt.Errorf("StdoutPipe: %w", err)
	}
	stderr, err := cmd.StderrPipe()
	if err != nil {
		return fmt.Errorf("StderrPipe: %w", err)
	}
```

#### 標準出力と標準エラーのパイプ設定
`StdoutPipe` と `StderrPipe` を使用して、コマンドの標準出力と標準エラー出力を取得します。これにより、コマンド実行中の出力をリアルタイムで処理できるようになります。
```go
	if err := cmd.Start(); err != nil {
		return fmt.Errorf("cmd.Start: %w", err)
	}
```

#### コマンドの開始
`cmd.Start` でコマンドを非同期で実行します。これにより、標準出力・エラーの監視を同時に行いながら、コマンドがバックグラウンドで進行するようにしています。

### 3. streamReader 関数
この関数は、標準出力や標準エラー出力を逐次的に読み取ってログに表示し、タイムアウト機能を管理します。

```go
func streamReader(scanner *bufio.Scanner, outputChan chan string, doneChan chan bool, timer *time.Timer, timeout time.Duration) {
	buf := make([]byte, 4096)
	scanner.Buffer(buf, 65536)

	scanner.Split(splitFunc)
	defer close(outputChan)
	defer close(doneChan)
	for scanner.Scan() {
		outputChan <- scanner.Text()
		timer.Reset(timeout)
	}
	doneChan <- true
}
```

#### リアルタイムの出力読み取り

`bufio.Scanner` を使用してコマンドの出力を行単位で逐次的に読み取ります。出力があるたびに `outputChan` チャネルに送信され、それがメインの `executeCommand` 関数で処理されます。

#### タイムアウトのリセット

新しい出力があるたびに、`timer.Reset(timeout)` でタイマーをリセットします。これにより、出力が続く限りタイムアウトが発生しないようにしています。

#### 終了時のチャネルクローズ

出力の読み取りが終了したら、`doneChan` チャネルに `true` を送信し、ストリームの処理が完了したことをメインループに通知します。

### 4. splitFunc 関数
この関数は、カスタムの行区切りとして `\r` と `\n` 、 `\r\n`を使用するための関数です。
標準の `Scanner` の動作を上書きするカスタムスプリッタです。
これにより、改行コードが異なる環境でも正確に行単位で処理できます。

```go
func splitFunc(data []byte, atEOF bool) (advance int, token []byte, err error) {
	if atEOF && len(data) == 0 {
		return 0, nil, nil
	}

	for i := 0; i < len(data); i++ {
		switch data[i] {
		case '\n':
			if i > 0 && data[i-1] == '\r' {
				return i + 1, data[:i-1], nil // CRLF
			}
			return i + 1, data[:i], nil // LF
		case '\r':
			if i == len(data)-1 || data[i+1] != '\n' {
				return i + 1, data[:i], nil // CR
			}
		}
	}

	if atEOF {
		return len(data), data, nil
	}
	return 0, nil, nil
}
```

### 5. コマンドの監視とログ出力
メインの `for select` ループで、標準出力やエラーをリアルタイムにログに出力しています。

```go
	isRunning := true
	for isRunning {
		select {
		case <-stdoutDone:
			isRunning = false
		case line := <-stdoutChan:
			slog.Info(line)
		case line := <-stderrChan:
			slog.Error(line)
		case <-ctx.Done():
			return fmt.Errorf("command cancelled due to timeout or context cancellation")
		}
	}
```
#### 標準出力とエラー出力のリアルタイム表示

`stdoutChan` や `stderrChan` からのデータがログに表示され、ユーザーにリアルタイムでフィードバックが行われます。

#### キャンセル処理

`ctx.Done()` が呼び出されると、コマンドがキャンセルされた旨のエラーメッセージが出力され、処理が終了します。

## 動作検証の方法と結果
### 検証方法
#### シンプルなシェルコマンド実行の確認
改善後のコードにおいて、まずはシンプルなコマンドとして以下のバッシュスクリプトを実行します。

```bash
for i in {1..5}; do echo "output $i"; sleep 3; done
```
このコマンドは、5回出力を行い、それぞれ3秒間隔で実行されます。ログが逐次的に出力されるかを確認します。

#### 標準エラー出力の確認
標準エラーが正しく取得されているか確認するために、エラーを出力する以下のバッシュスクリプトを実行します。

```bash
for i in {1..3}; do echo "error $i" 1>&2; sleep 2; done
```
標準エラーが逐次的にリアルタイムでログに出力されることを確認します。

#### 出力が途絶える場合のタイムアウト確認
出力が途絶えた場合の挙動を確認するため、タイムアウト設定を 5秒 にした状態で以下のバッシュコマンドを実行します。

```bash
echo "start"; sleep 10; echo "end"
```
このコマンドは、最初に出力を行い、10秒間何も出力せずに最後に再度出力を行います。タイムアウトが5秒に設定されているため、途中でキャンセルが発生することを期待します。

### 結果
#### シンプルなシェルコマンド実行の結果
コマンドが実行され、output 1 から output 5 までの出力が3秒間隔でリアルタイムにログに表示されました。

```text:実行結果の例
2024/10/18 11:14:56 INFO cmd: bash [-c for i in {1..5}; do echo "output $i"; sleep 3; done], path: .
2024/10/18 11:14:56 INFO output 1
2024/10/18 11:14:59 INFO output 2
2024/10/18 11:15:02 INFO output 3
2024/10/18 11:15:05 INFO output 4
2024/10/18 11:15:08 INFO output 5
```

![](/images/golang-realtime-cmd-logger/after_info.gif)

#### 標準エラー出力の結果
標準エラー出力として error 1 から error 3 までが2秒間隔でリアルタイムにログに表示されました。

```text:実行結果の例
2024/10/18 17:37:09 INFO cmd: bash [-c for i in {1..3}; do echo "error $i" 1>&2; sleep 2; done], path: .
2024/10/18 17:37:09 ERROR error 1
2024/10/18 17:37:11 ERROR error 2
2024/10/18 17:37:13 ERROR error 3
```

![](/images/golang-realtime-cmd-logger/after_error.gif)

#### タイムアウト検証の結果
5秒 のタイムアウト設定を行った場合、start の出力が表示された後、10秒間出力が途絶えたため、タイムアウトによりコマンドがキャンセルされました。

ログにはキャンセルされた旨のエラーメッセージが出力されました。

```text:実行結果の例
2024/10/18 17:38:30 INFO cmd: bash [-c echo "start"; sleep 10; echo "end"], path: .
2024/10/18 17:38:30 INFO start
2024/10/18 17:38:35 ERROR Error: command cancelled due to timeout or context cancellation
```

![](/images/golang-realtime-cmd-logger/after_timeout.gif)

## まとめ
今回の改善は、私が現在のプロジェクトで実際に直面した問題に基づいています。

あるバッチ処理が終了せず、さらにその処理中にコマンド実行のログも表示されていないという事象が発生しました。これにより、問題の原因を特定できず、デバッグが非常に困難な状況となっていました。

そのため、ログをリアルタイムに表示するとともに、出力が途絶えた際にプロセスを自動的にタイムアウトさせる仕組みを導入しました。この対応により、バッチ処理の監視が強化され、トラブルシューティングが容易になりました。

以上です。

本記事の内容が、皆様のシステム運用に役立つことを願っています。

https://github.com/mskasa/go-realtime-cmd-logger