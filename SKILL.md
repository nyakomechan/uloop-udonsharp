---
name: uloop-udonsharp
description: "Operate on UdonSharp (VRChat) scripts: attach/detach components, read/write fields, sync proxy↔backing, compile, create program assets, validate scene behaviours. Use ONLY when working with UdonSharp/UdonSharpBehaviour in a VRChat Unity project."
---

# uloop-udonsharp

UdonSharp (VRChat) 向けの操作を実行する。`uloop-execute-dynamic-code` を使って `UdonSharpEditor` API を呼び出す。

**いつ使うか**: UdonSharpBehaviour のアタッチ/デタッチ、フィールド読み書き、コンパイル、ProgramAsset作成、プロキシ↔バッキング同期、シーン検証など、UdonSharp固有の操作が必要な場合。

**いつ使わない**: 通常のUnityコンポーネント操作（Rigidbody追加など）や、UdonSharpと無関係な操作には `uloop-execute-dynamic-code` を直接使う。

## アーキテクチャ概要

UdonSharpは **プロキシ＋バッキングの2コンポーネント構成** を採用：

```
GameObject
├── MyScript : UdonSharpBehaviour  ← プロキシ（インスペクターで操作）
│     _udonSharpBackingUdonBehaviour ──┐
│                                      │
├── UdonBehaviour (HideInInspector)  ←─┘ バッキング（ランタイム実行）
│     programSource → UdonSharpProgramAsset
```

- **プロキシ**: ユーザーが書いたC#クラス。インスペクタに表示。
- **バッキング**: 隠しUdonBehaviour。VRChatランタイムで実行されるUdonプログラムを保持。
- フィールド値はプロキシ↔バッキング間で `CopyProxyToUdon` / `CopyUdonToProxy` で同期する必要がある。
- PlayMode中はバッキングが実データを持つため、読み書き前に同期が必須。

## 新規スクリプト追加ワークフロー

新規UdonSharpスクリプトをプロジェクトに追加してGameObjectにアタッチするまでの完全な手順：

1. **C#スクリプトを作成**（`using UdonSharp;` を忘れないこと。下記テンプレート参照）
2. **Unityコンパイル**: `uloop compile --force-recompile true --wait-for-domain-reload true`
3. **UdonSharpコンパイル**: `uloop execute-dynamic-code --code 'using UdonSharp; using UdonSharp.Compiler; using UnityEngine; UdonSharpCompilerV1.CompileSync(); return string.Format(""Compiled. Errors: {0}"", UdonSharpProgramAsset.AnyUdonSharpScriptHasError());'`
4. **ProgramAsset確認**: 新規スクリプトに対応するUdonSharpProgramAssetが自動生成されたか確認（セクション12で一覧取得）。**UdonSharpCompilerV1.CompileSync()だけでは新規スクリプトのProgramAssetが自動生成されない場合がある**。その場合はセクション10で手動作成する。
5. **ProgramAsset手動作成**（必要な場合）: セクション10のコード例を実行
6. **GameObjectにアタッチ**: セクション1のジェネリックメソッド（`AddUdonSharpComponent<T>()`）を使用
7. **シーン保存**

> **重要**: 手順4でProgramAssetが存在しない状態でアタッチすると `NullReferenceException` が発生する。

## 一般ワークフロー

1. 操作カテゴリに合ったコード例を `references/udonsharp-operations.md` から選ぶ
2. コード内の `MyScript` 等を実際のクラス名に置換する
3. `uloop execute-dynamic-code --code '...'` で実行（PowerShell引用符に注意: 内部 `"` は `""` でエスケープ）
4. 実行結果を確認し、必要に応じてリトライ

## 操作カテゴリ

| # | 操作 | 説明 |
|---|---|---|
| 1 | **コンポーネント追加** | UdonSharpBehaviourサブクラスをアタッチ（バッキング自動生成） |
| 2 | **フィールド値の読み取り** | プロキシのpublicフィールド値を取得（PlayModeではCopyUdonToProxy後に読み取り） |
| 3 | **フィールド値の書き込み** | プロキシのフィールドに値を設定しCopyProxyToUdonで同期 |
| 4 | **ProgramAssetの確認** | UdonSharpProgramAssetの取得・コンパイル状態確認 |
| 5 | **コンパイル実行** | 全UdonSharpスクリプトのUdonアセンブリへのコンパイル |
| 6 | **プロキシ↔バッキング同期** | CopyProxyToUdon / CopyUdonToProxy |
| 7 | **コンポーネント削除** | プロキシとバッキングの両方を安全に削除 |
| 8 | **シーン内一覧** | 全UdonSharpBehaviourとバッキング状態の一覧 |
| 9 | **SyncMode確認** | 同期モードの確認 |
| 10 | **ProgramAsset作成** | C#スクリプトに対応するUdonSharpProgramAssetを生成 |
| 11 | **PlayMode 読み書き** | PlayMode中のフィールド値アクセス（同期付き） |
| 12 | **型一覧取得** | プロジェクト内の全UdonSharp型とProgramAssetの照合 |
| 13 | **シーン検証・修復** | プロキシ/バッキングの整合性検証と自動修復 |
| 14 | **バッキング再同期** | バッキングUdonBehaviourの変数をプロキシから再投入 |
| 15 | **コンパイル状態確認** | エラー有無・スクリプト更新状態の確認 |

## PowerShell引用符ルール

`uloop execute-dynamic-code --code` をPowerShellで使う場合、コード全体を `'...'` 単一引用符で囲む。コード内の文字列リテラルは `""` でエスケープ:

```powershell
uloop execute-dynamic-code --code 'using UdonSharp; using UnityEngine; return GameObject.Find(""MyObject"") != null;'
```

**C# 文字列補間の制限**: `$"..."` はPowerShell単一引用符内でもパース対象にならないため機能するが、可読性とデバッグの観点から `string.Format()` の使用を推奨する:

```powershell
# 推奨: string.Format
return string.Format(""Count: {0}"", count);

# 回避: $"..." も動作するが見づらい
return $""Count: {count}"";
```

## セキュリティ制限

`uloop execute-dynamic-code` のセキュリティにより以下のAPIがブロックされる:

| ブロックされるAPI | 代替手段 |
|---|---|
| `System.Type.GetType()` | ジェネリックメソッド（`AddUdonSharpComponent<T>()` 等）または `MonoScript.GetClass()` |
| `System.IO.*` | ターミナルコマンドで代用 |

セクション1のコンポーネント追加では **ジェネリックメソッド** を使用すること。`Type.GetType` を使うコード例は `DangerousApiCall` エラーになる。

## 前提条件

- Unity Editor が起動していること
- プロジェクトにVRChat Worlds SDK (UdonSharp含む) がインストールされていること
- EditMode操作が多いが、PlayMode中の操作もProxySerializationPolicy付きで可能
- **新規スクリプトのアタッチ前に**、対応するUdonSharpProgramAssetが存在することを確認（セクション10・12参照）

## PlayModeでの制限

- バッキングUdonBehaviourがプログラムを読み込めていない場合（"Could not load the program"警告）、`CopyUdonToProxy` / `CopyProxyToUdon` が `NullReferenceException` になる。
  - この状態ではプロキシから直接フィールドを読むことは可能だが、初期値が返される（ランタイム値ではない）。
  - `CopyProxyToUdon` はメソッド自体は成功するが、バッキングが実行中でないためランタイム値は反映されない。
- VRChatクライアント外のPlayModeではUdonプログラムがロードされない場合がある。

## 新規スクリプト作成テンプレート

UdonSharpBehaviourの最小テンプレート。`using UdonSharp;` を忘れないこと:

```csharp
using UnityEngine;
using UdonSharp;

namespace MyNamespace
{
    [AddComponentMenu("MyNamespace/MyScript")]
    [UdonBehaviourSyncMode(BehaviourSyncMode.NoVariableSync)]
    public class MyScript : UdonSharpBehaviour
    {
        public int myPublicField;

        void Start()
        {
            Debug.Log(string.Format("[MyScript] Start. myPublicField={0}", myPublicField));
        }
    }
}
```

## 主要APIリファレンス

| API | 用途 |
|---|---|
| `UdonSharpEditorUtility.RunBehaviourSetup(proxy)` | プロキシにバッキングを紐付け |
| `UdonSharpEditorUtility.RunBehaviourSetupWithUndo(proxy)` | Undo付きセットアップ |
| `UdonSharpEditorUtility.GetBackingUdonBehaviour(proxy)` | プロキシからバッキング取得 |
| `UdonSharpEditorUtility.GetProxyBehaviour(udonBehaviour)` | バッキングからプロキシ取得 |
| `UdonSharpEditorUtility.IsProxyBehaviour(behaviour)` | プロキシ判定 |
| `UdonSharpEditorUtility.CopyProxyToUdon(proxy, policy)` | プロキシ→バッキング同期 |
| `UdonSharpEditorUtility.CopyUdonToProxy(proxy, policy)` | バッキング→プロキシ同期 |
| `UdonSharpEditorUtility.DestroyImmediate(proxy)` | プロキシ+バッキング削除 |
| `UdonSharpEditorUtility.GetUdonSharpProgramAsset(behaviour)` | ProgramAsset取得 |
| `UdonSharpProgramAsset.CompileAllCsPrograms()` | 全スクリプトコンパイル(非同期) |
| `UdonSharpCompilerV1.CompileSync()` | 同期コンパイル |
| `UdonSharpProgramAsset.AnyUdonSharpScriptHasError()` | エラー有無確認 |

## ProxySerializationPolicy

| 値 | 用途 |
|---|---|
| `ProxySerializationPolicy.Default` | デフォルトの深さ制限付き |
| `ProxySerializationPolicy.All` | 全フィールドを含む（PlayMode推奨） |
| `ProxySerializationPolicy.RootOnly` | ルートオブジェクトのみ |
| `ProxySerializationPolicy.PreBuildSerialize` | ビルド前の事前シリアライズ |
| `ProxySerializationPolicy.CollectRootDependencies` | 依存関係収集（内部用） |