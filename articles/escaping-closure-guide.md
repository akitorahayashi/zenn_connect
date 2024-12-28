---
title: "@escapingのイメージ"
emoji: "🛴"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "swift", "escaping"]
published: true
---
# はじめに
通常、クロージャはその場で使い終わると破棄される。しかし再度使う場合、クロージャをどこかに保存する必要がある。@escapingを付けることで、「このクロージャはスコープ外に逃げる可能性がある」と伝えることができる。これにより、Swiftは適切なメモリ管理（参照カウント）を行うことができる。

# @escapingを使う状況
@escapingは、クロージャが関数のスコープを超えて保持される場合に必要で、これを付けないと、コンパイルエラーが発生する。@escapingを使う状況とは次のような状況である。
1. クロージャをプロパティとして保存。
2. クロージャを他の関数や構造に渡す。
3. 非同期処理など、後で呼び出すために保存する。

# 1. クロージャをプロパティとして保存
クロージャをクラスや構造体のプロパティとして保存する場合、スコープを超えて保持されるため、@escapingが必要。
```swift
class TaskManager {
    var completion: (() -> Void)? // クロージャをプロパティとして保存

    func saveCompletion(_ closure: @escaping () -> Void) {
        self.completion = closure
    }

    func executeTask() {
        print("Task started")
        completion?() // 保存していたクロージャを実行
    }
}

// 使用例
let manager = TaskManager()
manager.saveCompletion {
    print("Task completed!")
}
manager.executeTask()
// 出力: Task started
//       Task completed!
```

# 2. クロージャを他の関数や構造に渡す
クロージャを他の関数や構造体に渡す場合、元の関数スコープを超えて保持される可能性があるため、@escapingが必要。
```swift
func performTask(handler: @escaping () -> Void) {
    DispatchQueue.global().async {
        handler() // 渡されたクロージャを非同期処理の中で実行
    }
}

func startTask() {
    performTask {
        print("Task performed!")
    }
}

// 実行例
startTask()
// 出力: Task performed!
```

# 3. 非同期処理など、後で呼び出すために保存する
非同期処理では、渡されたクロージャを保持して後で実行するため（自動で行われる）、@escapingが必要。
```swift
func fetchData(completion: @escaping (String) -> Void) {
    DispatchQueue.global().async {
        print("Fetching data...")
        sleep(2) // 模擬的な遅延処理
        DispatchQueue.main.async {
            completion("Data fetched!") // クロージャを後で実行
        }
    }
}

fetchData { result in
    print(result) // 非同期処理後に呼び出される
}

// 出力: Fetching data...
//       Data fetched!
```

# まとめ
@escapingは、クロージャが関数のスコープを超えて保存される場合に必要な機能で、プロパティとして保存、他の関数に渡す、非同期処理で後から呼び出すといった状況で利用される。