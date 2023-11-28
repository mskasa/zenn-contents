---
title: "Go言語 インタフェース活用の具体例（多態性と依存性の注入について）"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: false
---

## 概要

Go言語におけるインタフェースの活用方法について、実際のコードを交えながら解説します。

想定読者は「Go言語のインタフェースについて勉強したものの、実際どういう時に使うの？」と疑問に感じている方々です。

そのため、所謂「インタフェースとは」的な説明は省略させていただきます。

## 多態性（ポリモーフィズム）

## 依存性の注入（DI）

```go
type Application struct {
    logger ConsoleLogger // 直接 ConsoleLogger に依存
}

func (app *Application) Run() {
    app.logger.Log("Application is running")
}

func main() {
    app := Application{
        logger: ConsoleLogger{},
    }

    app.Run() // 出力: Log to console: Application is running
}
```

- 単体テストが困難
- 拡張性の欠如

ConsoleLogger の動作が Application のテスト結果に直接影響を与えるため、純粋な単体テストを行うのが難しくなります。
テスト中に ConsoleLogger の振る舞いを制御することができないため、テストの再現性や精度が低下します。また、外部リソース（例えばコンソール出力）に依存するコードのテストは、モックやスタブを使用して内部依存関係を隔離する場合と比べて、一般的に複雑で時間がかかります。依存性注入を使用することで、これらの問題を回避し、より効率的で再現性の高いテストを実現できます。

```go
package main

import (
    "fmt"
)

// Logger インタフェースはログ記録の機能を定義します。
type Logger interface {
    Log(message string)
}

// ConsoleLogger は Logger インタフェースの一つの実装です。
type ConsoleLogger struct{}

func (l ConsoleLogger) Log(message string) {
    fmt.Println("Log to console:", message)
}

// Application は Logger の実装に依存しています。
type Application struct {
    logger Logger
}

func (app *Application) SetLogger(logger Logger) {
    app.logger = logger
}

func (app *Application) Run() {
    app.logger.Log("Application is running")
}

func main() {
    app := &Application{}
    
    // コンソールロガーを注入
    consoleLogger := ConsoleLogger{}
    app.SetLogger(consoleLogger)

    app.Run() // 出力: Log to console: Application is running
}
```

```go
package main

import (
    "testing"
)

// MockLogger はテスト用の Logger 実装です。
type MockLogger struct {
    LoggedMessages []string
}

func (l *MockLogger) Log(message string) {
    l.LoggedMessages = append(l.LoggedMessages, message)
}

func TestApplicationRun(t *testing.T) {
    mockLogger := &MockLogger{}
    app := &Application{
        logger: mockLogger,
    }

    app.Run()

    if len(mockLogger.LoggedMessages) != 1 || mockLogger.LoggedMessages[0] != "Application is running" {
        t.Errorf("Expected 'Application is running' log message, got %v", mockLogger.LoggedMessages)
    }
}
```

インタフェースに依存している場合、そのインタフェースのIN/OUT（つまり、メソッドのシグネチャや振る舞いの契約）が変わらない限り、依存している側のコードの振る舞いは変わらないというメリットがあります。これにより、インタフェースの実装を変更しても、インタフェースを使用するコードには影響が及びません。

一方で、インタフェースのIN/OUT自体が変更される場合、それは依存しているコードに大きな影響を与える可能性があります。このため、インタフェースの変更は慎重に行う必要があり、通常は互換性を保つために既存のインタフェースを維持しながら新しいインタフェースを追加する形を取ることが一般的です。

```go
// 既存のインタフェース
type Logger interface {
    Log(message string)
}

// 新しい機能を追加した拡張インタフェース
type AdvancedLogger interface {
    Logger // 既存のインタフェースを組み込む
    LogWithLevel(message string, level LogLevel)
}

// 既存のインタフェースを実装する型
type SimpleLogger struct{}

func (l SimpleLogger) Log(message string) {
    fmt.Println(message)
}

// 拡張インタフェースを実装する型
type LevelLogger struct{}

func (l LevelLogger) Log(message string) {
    // 基本的なログ記録の実装
    fmt.Println(message)
}

func (l LevelLogger) LogWithLevel(message string, level LogLevel) {
    // レベルを含むログ記録の実装
    fmt.Printf("[%s] %s\n", level, message)
}

// 使用例
func performLogging(l Logger) {
    l.Log("Basic log message")
    if advLogger, ok := l.(AdvancedLogger); ok {
        advLogger.LogWithLevel("Advanced log message", Info)
    }
}
```