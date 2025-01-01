---
title: "Swift Concurrencyの概要"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "swift", "concurrency", "async"]
published: false
---
# はじめに
Swift Concurrencyは、Swift 5.5以降で、Swiftにおける非同期処理や並列処理を簡単かつ安全に記述するための仕組みである。従来の非同期処理（例えば、コールバックやDispatchQueue）と比較して、コードの可読性・安全性を向上させることができる。

# サンプルアプリ
https://github.com/akitorahayashi/techtrain_book_reviewer

# 従来の非同期処理の書き方の問題点
```swift
func fetchBookReview(
        id: String,
        token: String
    ) {
        let headers = ["Authorization": "Bearer \(token)"]
        let endpoint = "/books/\(id)"
        
        TechTrainAPIClient.shared.makeRequest(to: endpoint, method: "GET", body: nil, headers: headers) { result in
            switch result {
            case .success(let data):
                do {
                    let decoder = JSONDecoder()
                    let bookReview = try decoder.decode(BookReview.self, from: data)
                    completion(.success(bookReview))
                } catch {
                    completion(.failure(.underlyingError(.decodingError)))
                }
            case .failure(let error):
                completion(.failure(error.toServiceError()))
            }
        }
    }
```
## 問題 1: ネストが深くなりやすい
非同期処理をコールバックで書くと、ネストが深くなり、可読性が下がる。
## 問題 2: エラーハンドリングが分散
従来のCompletion Handlerでは、エラー処理が各コールバックに分散する。
## 問題 3: データの競合が起きる可能性がある
競合は、複数の処理が同時に同じデータを使い、「読み取り」や「書き込み」が重なるときに起こる。例えば、カウンターの値を1ずつ増やす処理で、2つの処理が同時に現在の値「0」を読み取った場合、それぞれが「0 + 1 = 1」と計算し、結果を上書きする。この結果、「2」が期待されているが「1」となる、という問題が発生する。
# Swift Concurrency
## 1. async/await
- async: 非同期処理を扱う関数を定義する
- await: 非同期処理の結果を待つ

この仕組みで、従来のコールバックを使った複雑なコードがシンプルになる。

```swift: BookReviewService.swift
func fetchBookReview(
        id: String,
        token: String
    ) async throws(TechTrainAPIError.ServiceError) -> BookReview {
        let endpoint = "/books/\(id)"
        let headers = ["Authorization": "Bearer \(token)"]
        
        do {
            let bookReviewData = try await TechTrainAPIClient.shared.makeRequestAsync(to: endpoint, method: "GET", headers: headers, body: nil)
            let decodedBookReview = try BookReview.decodeBookReview(bookReviewData)
            return decodedBookReview
        } catch {
            throw error.toServiceError()
        }
    }
```
## 2. Task
TaskはSwift Concurrencyにおける非同期処理を実行する基本単位である。同期メソッド内で非同期処理を実行する必要があるときに使用し、非同期関数を呼び出すための文脈を作るために使う。
### 処理の効率化
Taskを使うことで、非同期処理が待機中でもスレッドがブロックされないため、効率的に処理を行うことができる。例えば、ネットワーク通信やファイルの読み書きのような時間のかかる処理中に、他のタスクを並行して実行できる。
```swift
Task {
    try? await Task.sleep(nanoseconds: 3 * 1_000_000_000) // 3秒待機
    print("ここは非同期処理の結果を待ってから実行される部分")
}

print("ここは先に実行される部分")
```
### 並行処理の実行
```swift
func fetchNumber1() async -> Int {
    try? await Task.sleep(nanoseconds: 1 * 1_000_000_000) // 1秒待機
    return 42
}

func fetchNumber2() async -> Int {
    try? await Task.sleep(nanoseconds: 2 * 1_000_000_000) // 2秒待機
    return 58
}

Task {
    async let number1 = fetchNumber1()
    async let number2 = fetchNumber2()
    let result1 = await number1
    let result2 = await number2
    let total = result1 + result2
    print("The total is \(total)") // 結果: The total is 100
}
```
#### ~処理の流れ~
1. タスクの開始
`async let number1 = fetchNumber1()`で`fetchNumber1`（2秒待機）が実行開始。
`async let number2 = fetchNumber2()`で`fetchNumber2`（1秒待機）が並行して実行開始。
2. number1の結果を待機
`let result1 = await number1`で、`fetchNumber1`が完了するまで待機（約2秒）。
3. number2の結果を待機
`let result2 = await number2`で、`fetchNumber2`が完了するまで待機。
ただし、`fetchNumber2`は並行実行されており、1秒で終了しているため、ここでの待機時間はゼロ。
4. 結果の合計を計算
`result1`（42）と`result2`（58）を加算し、`total`に代入。
5. 結果を出力
`print("The total is \(total)")`で100を出力。

### 非同期処理をまとめて実行して、結果がまとまってから別の処理を実行する
withTaskGroupは、Swift Concurrencyで複数の非同期タスクをグループ化して管理するための仕組みである。一度に複数の非同期処理を実行し、その結果をまとめて収集して、活用したい場合に使用する。
withTaskGroupを使用する場合、すべてのタスクの結果の型は同じである必要がある。そして、タスクの完了順序は保証されない。完了した順に結果を収集する。
```swift
func fetchNumber1() async -> Int {
    try? await Task.sleep(nanoseconds: 1 * 1_000_000_000) // 1秒待機
    return 42
}

func fetchNumber2() async -> Int {
    try? await Task.sleep(nanoseconds: 2 * 1_000_000_000) // 2秒待機
    return 58
}

Task {
    let total = await withTaskGroup(of: Int.self) { group -> Int in
        // fetchNumber1()はこのaddTaskの中で非同期タスクとして実行を開始
        group.addTask {
            await fetchNumber1()
        }

        // fetchNumber2()もこのaddTaskの中で非同期タスクとして実行を開始
        group.addTask {
            await fetchNumber2()
        }

        var sum = 0

        // タスクが完了するごとに結果を受け取り、合計を計算
        for await result in group {
            sum += result
        }

        // 全てのタスクが完了した後に合計値を返す
        return sum
    }

    // 全タスクが終了し合計値が計算された後に出力
    print("The total is \(total)") // 結果: The total is 100
}
```

### Taskを中断する
ユーザーの操作や特定の条件に応じて、Task.cancel()を使うことで処理を停止できる。
```swift
import Foundation
func fetchData() async throws -> String {
    try await Task.sleep(nanoseconds: 10 * 1_000_000_000) // 10秒待機
    return "データ取得完了"
}

let task = Task {
    do {
        let result = try await fetchData()
        print(result)
    } catch {
        print("タスクが中断されました")
    }
}

Task {
    try await Task.sleep(nanoseconds: 3 * 1_000_000_000) // 3秒待機
    task.cancel() // タスクをキャンセル
}
```
このコードが動作するのは、Task.sleepが内部的にキャンセルをサポートしているためである。もし他の非同期処理を使っている場合は、キャンセルの仕組みを明示的に追加する必要がある。
```swift
import Foundation

func downloadLargeFile() async throws {
    var downloaded = 0
    let totalSize = Int.random(in: 500...1000) // ファイルサイズ（MB）
    print("ファイルサイズは \(totalSize) MB です")
    
    while downloaded < totalSize {
        try Task.checkCancellation() // キャンセル確認
        try await Task.sleep(nanoseconds: 1 * 1_000_000_000) // 1秒待機
        
        let progress = Int.random(in: 50...100) // ダウンロード量（MB）
        downloaded = min(downloaded + progress, totalSize)
        print("進捗: \(downloaded) / \(totalSize) MB")
    }
    
    print("ダウンロードが完了しました！")
}

let task = Task {
    do {
        try await downloadLargeFile()
    } catch is CancellationError {
        print("タスクがキャンセルされました")
    } catch {
        print("エラーが発生しました: \(error)")
    }
}

// 5秒後にタスクをキャンセル
Task {
    try await Task.sleep(nanoseconds: 5 * 1_000_000_000)
    task.cancel()
}
```
## 3. Actor
Actor内のプロパティやメソッドへのアクセスは直列化され、複数のスレッドから同時に操作される際のデータの破損を防ぐ。
- データの競合を防ぐため、actorのプロパティを読み込んだり、メソッドを使うときは await をつける必要がある。
- actorのプロパティに変更を加える場合は、actor 内でプロパティを操作する専用のメソッドを定義し、そのメソッドを呼び出す必要がある（意図しない変更に対する安全性）。
```swift
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }

    func getValue() -> Int {
        return value
    }
}

let counter = Counter()

Task {
    // 複数のタスクが同時にincrementを呼び出しても、Actorがアクセスを直列化
    await withTaskGroup(of: Void.self) { group in
        for _ in 0..<100 {
            group.addTask {
                await counter.increment()
            }
        }
    }

    let finalValue = await counter.getValue()
    print("最終値: \(finalValue)") // 常に100
}
```
## 4. Actor Isolation
Actor Isolationは、並行処理の安全性を確保するために、あるactorが管理する状態を他のコードから安全に扱えるようにする概念である。具体的には、状態の一貫性を保ちながら、複数のタスクから同時にアクセスされることや、データ競合や不正な状態の発生を防ぐ役割を果たす。
### 基本的なポイント
#### 隔離（Isolation）:
actorの内部状態は、デフォルトで外部から直接アクセスできない。
#### 直列化（Serialization）:
actor内の処理は、直列化され、1つずつ順番に実行される。同時に複数のタスクがactorの内部状態を操作することはできない。
#### 非同期（Async）:
他のスレッドやタスクからactorにアクセスする際は、非同期の文脈でawaitを使用する必要がある。非同期的なアクセスは、他の処理が終了するまで待機するため、安全性が保たれる。

@MainActor と MainActor.run はActor Isolation（隔離）の一環として、Actorの概念を基盤とした並行処理を扱う方法であり、メインスレッド上の特定の実行を保証するために設計されている。
### @MainActor
@MainActorを使用すると、プロパティやメソッドがメインスレッドで実行されることを保証できる。これにより、UIの更新など、メインスレッドで実行する必要がある処理を安全に実行できるようになる。
```swift
import UIKit

@MainActor
class ViewController: UIViewController {
    ...
    override func viewDidLoad() {
        super.viewDidLoad()
    }
    ...
}
```
### MainActor.run
特定の箇所でUI更新をメインスレッドに限定する必要があるとき、MainActor.runを使う。
```swift
private func fetchAndUpdateUI() async {
        let data = await fetchData()
        await MainActor.run {
            label.text = "Data: \(data)"
        }
    }
```
# まとめ
Swift Concurrencyは、Swift 5.5以降で、async/awaitによるシンプルな非同期処理や、Actorによるデータ競合の防止によって、従来のコールバックやDispatchQueueを使用した複雑なコードを簡潔かつ安全に書き換えるためのSwift言語の仕組みである。