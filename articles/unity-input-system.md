---
title: "【Unity6】Input System を用いた入力処理の実装"
emoji: "🎮"
type: "tech"
topics: ["Unity", "Unity6","InputSystem", "csharp", "ゲーム開発"]
published: true
published_at: "2025-01-16 21:00"
---

# はじめに
ゲームエンジンの Unity にてキーボードやコントローラーから取得する方法は幾つかありますが、今回は「Input System」という機能を紹介します。
個人的に「Input System」は便利で使いやすく非常にオススメできる入力系の機能です。
※古いバージョンの記事は沢山ありますが、最新の記事がまだ少ないので自分なりに纏めました。
:::message
今回使用する環境は2024年12月時点で最新になりますので、他のバージョンとはUIが大きく異なる場合があります
:::

# 動作環境
今回は下記の環境で実施します。
- Windows 10
- Unity (6000.0.29f1)
- Input System 1.11.2

# Input System とは
Unity の Input System は、ゲームやアプリケーションで使用する入力デバイス（キーボード、マウス、コントローラー、タッチスクリーンなど）を簡単に扱えるようにするための入力管理フレームワークです。

## Input System を使用するメリット
Input System の個人的に感じる良い点はこちらです。
- 「移動」「ジャンプ」「攻撃」などの操作を設定すれば、キーボードやコントローラーなどの異なるデバイスでも同じように動作するように簡単に作れる。
- 専用の画面（Input Actions Editor）で視覚的に設定を管理できる。

# Input System の導入
自分の環境では Unity6 で新しいプロジェクトを作った時点で最初から導入されていましたが、一応導入方法を記載します。
1. 画面左上の「Window」から「Package Manager」を開く。
![](/images/unity-input-system/open-package-manager.png)
2. 「Unity Registry」から「Input System」を探して「install」をクリック。
検索もできます。
![](/images/unity-input-system/install-input-system.png)

これで Input System の導入が完了です。

# 事前確認
1. Input System をインポートすると `InputSystem_Actions.inputactions` というファイルが作成されている。
![](/images/unity-input-system/input-system-actions.png)
2. 「Edit」>「Project Settings」>「Input System Package」を開き、「Project-wide Actions」 に 1 のファイルがアタッチされている。
![](/images/unity-input-system/project-settings.png)

もし、ファイルが存在しない場合は、Project で右クリックして「Ceate」>「Input Actions」からファイルを作成して、「Project Settings」でアタッチして下さい。
![](/images/unity-input-system/create-input-actions.png)

# Input Action アセットの設定
`InputSystem_Actions` ファイルをダブルクリックして、入力アクションエディターを開きます。
最初からアセットが設定された状態になっているかと思います。ここはお好みで修正してみて下さい。
![](/images/unity-input-system/input-actions-editor.png)
| 名前 | 説明 |
| ---- | ---- |
| Action Maps | グループとしてまとめて有効または無効にできるアクションのコレクション |
| Actions | アクションマップで定義されているすべてのアクションと、各アクションに関連付けられているバインディングを表示 |
| Action Properties | アクションのプロパティを表示 |
| Binding Properties | バインディングのプロパティを表示 |

## アクションプロパティ (Action Properties)
アクション(Action)を選択している時はアクションのプロパティが表示され、アクションを編集することができます。
![](/images/unity-input-system/action-properties.png)

### アクションタイプ (Action Type)
アクションタイプ設定では、ボタン(Button)、値(Value)、パススルー(PassThrough)を選択できます。
- Button: キーボードのキー、マウスクリック、ゲームパッドのボタンなど、オン/オフの入力に適用。
- Value: マウス移動、ジョイスティック、デバイスの傾きなど、連続的な入力に適用。開始・終了情報や競合解決のデータも提供。
- PassThrough: Valueと同様の連続入力向けだが、開始・終了情報や競合解決を行わず、基本的な値のみ提供。

### コントロールタイプ (Control Type)
コントロールタイプ設定では、アクションに適した入力タイプを選択できます。
これにより、バインド設定時に表示されるコントロールが制限され、適切な入力のみが割り当て可能になります。

## バインディングプロパティ (Binding Properties)
バンディングを選択している時はバインディングのプロパティが表示され、バンディングを編集することができます。
![](/images/unity-input-system/binding-properties.png)

# 入力処理の為の準備
ここまで出来たら実際に Input System を使用して入力情報を取得します。
手始めに Unity 内で適当なオブジェクトを作成します。
`Controller.cs`というスクリプトファイルも作成し、オブジェクトにコンポーネント追加します。
今回は入力情報を取得してこの四角オブジェクトを操作します。
![](/images/unity-input-system/create-object.png)

# Input System のワークフロー

## アクションの使用
![](https://docs.unity3d.com/Packages/com.unity.inputsystem@1.11/manual/images/Workflow-Actions.png)
コードからアクションにアクセスする方法です。
`Controller.cs`のソースコードを下記のようにして下さい。
```cs
using UnityEngine;
using UnityEngine.InputSystem;

public class Controller : MonoBehaviour
{
    private float speed = 10f;
    private InputAction moveAction;
    private InputAction jumpAction;

    private void Start()
    {
        // "Move" と "Jump" のリファレンスを探す
        moveAction = InputSystem.actions.FindAction("Move");
        jumpAction = InputSystem.actions.FindAction("Jump");
    }

    private void Update()
    {
        // 移動処理
        var moveValue = context.ReadValue<Vector2>();
        var move = new Vector3(moveValue.x, 0f, moveValue.y) * speed * Time.deltaTime;
        transform.Translate(move);

        // ジャンプ処理
        if (jumpAction.IsPressed())
        {
            Debug.Log("Jumping");
        }
    }
}
```

### メリットとデメリット
- メリット
  - Action Maps や Bindings などの機能を活用可能
  - Actions Editor で簡単に設定 できる
- デメリット
  - ローカルマルチプレイ向けの機能がない

## アクションと PlayerInput コンポーネントの使用
![](https://docs.unity3d.com/Packages/com.unity.inputsystem@1.11/manual/images/Workflow-PlayerInput.png)
アクションと PlayerInput コンポーネントを一緒に使用する方法です。
`Controller.cs`のソースコードを下記のようにして下さい。
```cs
using UnityEngine;
using UnityEngine.InputSystem;

public class Controller : MonoBehaviour
{
    private float speed = 10f;
    private Vector2 moveInput = Vector2.zero;

    private void Update()
    {
        var move = new Vector3(moveInput.x, 0f, moveInput.y) * speed * Time.deltaTime;
        transform.Translate(move);
    }

    public void OnMove(InputAction.CallbackContext context)
    {
        moveInput = context.ReadValue<Vector2>();
    }

    public void OnJump(InputAction.CallbackContext context)
    {
        Debug.Log("Jumping");
    }
}
```

その後、オブジェクトに PlayerInput コンポーネントを追加し、インスペクター上で下記の設定を行って下さい。
1. Behavior を `Invoke Unity Events` に変更
2. Events で該当のアクションに対して、先ほど作成した関数をセット
![](/images/unity-input-system/player-input.png)

### メリットとデメリット
- メリット
  - Action Maps や Bindings などの機能を活用可能
  - エディタのインスペクターでコールバック設定可能（コード量を減らせる）
  - ローカルマルチプレイ向けのデバイス管理や画面分割を簡単に実装可能
- デメリット
  - デバッグが難しくなる可能性（アクションとコードの関係が明示されない）
  - 仕組みがブラックボックス化 されており、カスタマイズしにくい

# 実行して挙動確認
どちらかの方法で実装が完了したら、実際に挙動を確認してみましょう。
どの方法で実装しても処理内容は変わらずこちらのようになっているはずです。
Input Action アセットの設定によりますが、キーボードでもコントローラーでも同じように操作できるかと思います。
![](/images/unity-input-system/play-input-system.gif)
※↑ジャンプはログだけの実装ですが、ちゃんと出来ています（笑）

# 最後に
以上で Input System の説明は終わりになります。
非常に便利な機能なので是非使用してみて下さい。

# 公式ドキュメント
https://docs.unity3d.com/Packages/com.unity.inputsystem@1.11/manual/index.html
