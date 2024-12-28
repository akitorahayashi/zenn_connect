---
title: "俯瞰して画面遷移を管理する"
emoji: "🗺️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "swift", "navigation", "uikit"]
published: false
---
# はじめに
アプリ開発において、画面ごとに個別の遷移ロジックを書き散らしてしまうと、コードの混乱を招き、プロジェクトがスケールするにつれて手に負えなくなるリスクがある。特に多機能なアプリでは、画面遷移のロジックが複雑になりがちで、適切な管理手法を組み合わせることで保守性や拡張性を向上させる必要がある。

ここでは、画面遷移を管理するための設計について扱う。適切なパターンを選択し、それらを効果的に組み合わせることで、長期的な安定性や柔軟な拡張に対応することを目指す。

# 1. Coordinator Pattern
## 概要
- 画面遷移のロジックをViewControllerから分離し、専用のCoordinatorクラスで管理する。
- アプリの画面構造を俯瞰しやすく、遷移ロジックが一元化される。
## メリット
- 各画面（ViewController）が画面遷移の責務を持たないため、テストが容易。
- 画面遷移を一箇所にまとめることでコードが整理され、保守性が向上。
## 全体のフロー
### 1. ベースとなるプロトコルの作成
すべてのCoordinatorが共通して持つ機能をプロトコルで定義する。
### 2. 画面遷移の単位ごとにCoordinatorを作成
各機能（モジュール）ごとにCoordinatorを分割する。
### ViewControllerとの連携
各ViewControllerにイベントハンドラ（クロージャなど）を用意して、画面遷移のトリガーを設置。
### SceneDelegateやAppDelegateで起点を作成
アプリ起動時に、最初のCoordinatorを呼び出し、アプリ全体の遷移を開始。
## ディレクトリの構造(例)
Project/
├── Coordinators/
│   ├── Base/
│   │   └── Coordinator.swift
│   ├── Home/
│   │   ├── HomeCoordinator.swift
│   │   └── HomeViewController.swift
│   ├── Detail/
│   │   ├── DetailCoordinator.swift
│   │   └── DetailViewController.swift
│   └── Purchase/
│       ├── PurchaseCoordinator.swift
│       └── PurchaseViewController.swift
└── App/
    └── SceneDelegate.swift