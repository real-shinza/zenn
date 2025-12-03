---
title: "Unity 3D でカメラ方向に合わせた移動の実装"
emoji: "🏃‍♀️"
type: "tech"
topics: ["unity", "unity3d", "Unity6", "cinemachine", "ゲーム開発"]
published: true
published_at: "2025-11-30 12:00"
---

# はじめに

### 今回やる事
この記事では、Unity 上で3Dキャラクターをカメラの向きに合わせて移動させる方法を解説します。
三人称視点でカメラを回転させても、キャラクターが常にカメラの前方向に移動できるように実装していきます。
:::message
本記事ではアニメーションの詳細な設定には踏み込みませんが、移動と連動させるための基本的な制御のみ行います。
:::

### 動作環境
今回使用する動作環境は以下の通りです。
- Windows 10
- Unity 6000.0.62f1
- Input System 1.14.2
- Cinemachine 3.1.5

### ソースコード
今回の内容の Unity プロジェクトは以下の Git リポジトリに公開しています。
こちらも併せてご確認ください。
https://github.com/real-shinza/unitychan-movement-3d

# 事前準備

### Unity プロジェクトを作成
Unity のバージョンは特に何でもよいので、お好きなバージョンで新規プロジェクトを作成してください。
今回の例ではテンプレートに `3D (Built-In Render Pipeline)` を使用しています。

### Input System
今回、入力処理には Unity のパッケージの `Input System` を使用します。
Input Actions に関してはデフォルトの設定のままにしておきます。
![](/images/unitychan-movement-3d/input-system-actions.png)

以前、`Input System` に関する記事を公開していますので、詳細は以下の記事をご参照ください。
https://zenn.dev/kainari/articles/unity-input-system

### 地面を作成
キャラクターを歩かせるための地面を作ります。
`Hierarchy (ヒエラルキー)` ウィンドウ左上の「＋」ボタンをクリックし、`3D Object (3D オブジェクト)` > `Plane (平面)` を選択して地面を作成します。
デフォルトで `Collider (コライダー)` が設定されているため、特に追加設定は不要です。
![](/images/unitychan-movement-3d/create-plane.png)

### 3D モデルデータの用意
この記事で使用するのは「ユニティちゃん 3Dモデルデータ v.1.4.0」です。
以下の公式ウェブサイトからダウンロードできます。
また、この記事用に作成した GitHub リポジトリ内にも同データを含めています。

https://unity-chan.com/
![](/images/unitychan-movement-3d/unitychan-license-logo.png)
この作品は[ユニティちゃんライセンス条項](https://unity-chan.com/contents/license_jp/)の元に提供されています
© Unity Technologies Japan/UCL

ユニティちゃんのアセットデータには、シーンファイルやスクリプトなど、様々なデータが含まれています。
本記事では使用しないデータはインポートせず、必要なファイルのみを利用します。
アニメーションとモデルデータとはそれぞれ以下のものを使用します。
このアニメーションには不要な設定も多く含まれていますが、本記事ではそれらを無視して、設定の変更は行いません。
- `UnityChan` > `Animators` > `UnityChanLocomotions.controller`
![](/images/unitychan-movement-3d/unitychan-animator.png)
- `UnityChan` > `Models` > `unitychan.fbx`
![](/images/unitychan-movement-3d/unitychan-model.png)
:::message
このアセットは Unity 6 に正式対応していないため、幾つか警告が表示されますが、今回は無視します。
この記事の内容は特定のモデルデータに依存していないため、他の 3D モデルを使用しても問題ありません。
:::

# 簡易的な移動処理の実装
プレイヤーが移動し、移動方向に回転し、アニメーションと連動する簡易的な移動処理を実装します。

### スクリプト作成
まず、移動処理を作るために `PlayerController.cs` という名前のスクリプトファイルを作成します。
このスクリプトでは以下の流れで処理を行います。
1. 入力値の取得
2. 入力方向への移動
3. 移動方向への回転
4. アニメーションの制御

ソースコードの中身は以下です。
```cs:PlayerController.cs
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerController : MonoBehaviour
{
    [SerializeField]
    private Animator animator;
    [SerializeField]
    private CharacterController characterController;
    [SerializeField]
    private float locomotionSpeed;
    private InputAction moveAction;
    private readonly int speedHash = Animator.StringToHash("Speed");

    private void Start()
    {
        moveAction = InputSystem.actions.FindAction("Move");
    }

    private void FixedUpdate()
    {
        // 入力値
        var input = moveAction.ReadValue<Vector2>();

        // 入力方向をワールド空間に変換
        var moveVec = new Vector3(input.x, 0f, input.y);

        // 移動入力の大きさを取得（小さなノイズは0として扱う）
        var moveMag = moveVec.magnitude > 0.1f ? moveVec.magnitude : 0f;

        // 実際の移動
        characterController.Move(moveVec * moveMag * locomotionSpeed * Time.fixedDeltaTime);

        if (moveVec.sqrMagnitude > 0.1f)
        {
            // 向きを移動方向に合わせる
            var targetRotation = Quaternion.LookRotation(moveVec);
            transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, Time.deltaTime * 10f);
        }

        // アニメーション操作
        animator.SetFloat(speedHash, moveMag);
    }
}
```

この実装では、コントローラーのアナログスティックの倒し具合によって移動速度が変化します。

### コンポーネントを追加
キャラクターオブジェクトに対して、動作に必要なコンポーネントを設定していきます。
`Hierarchy (ヒエラルキー)` からオブジェクトを選択した状態で、`Inspector (インスペクター)` ウィンドウ下部の `Add Component` をクリックし、以下のコンポーネントを追加してください。
- `Character Controller`
  キャラクターの衝突判定や段差処理を簡単に行うためのコンポーネントです。
  Rigidbody を使用せずに、スクリプトから直接移動制御を行えるのが特徴です。
  地面との当たり判定、スロープの上り下り、段差の超えなどを自動的に補助してくれます。
  モデルデータに合わせてプロパティを設定してください。
  各プロパティの詳細は[こちら](https://docs.unity3d.com/6000.0/Documentation/Manual/class-CharacterController.html)の公式ドキュメントをご確認ください。
- `Player Controller (Script)`
  先ほど作成したスクリプトです。
  プロパティの `Locomotion Speed` は移動速度です。

コンポーネントを追加して、プロパティを設定すると以下のようになります。
![](/images/unitychan-movement-3d/unitychan-add-component.png)

今回のように単純な平面上のみの移動であれば、`Transform` で直接移動させるだけでも動作しますが、将来的に段差や障害物のあるステージへ拡張することを考慮し、汎用性の高い `Character Controller` を使用しています。

### 一旦実行して動作確認
ここまで設定できたら、Unity の再生ボタンを押して動作を確認します。
WASD キーまたはスティック入力で、キャラクターが前後左右に移動すれば成功です。
![](/images/unitychan-movement-3d/unitychan-movement-simple.gif)
この時点ではまだカメラは固定されているため、移動方向はワールド座標基準となります。
仮にカメラを回転させたとしても現時点では移動方向は変化しません。

# カメラの追跡処理を実装
カメラの追跡処理は Cinemachine を使うことで、ノーコードで実現できます。
:::message
本記事では Cinemachine の詳しい仕組みについては深掘りしません。
公式ドキュメントを添付しますので、詳細はそちらをご参照ください。
:::

### Cinemachine のインストール
カメラの追跡処理を実装するために、Unity　公式のカメラ制御パッケージ `Cinemachine` を導入します。
Unity 上部メニューから `Window (ウィンドウ)` > `Package Manager (パッケージマネージャー)` を開きます。
検索バーに `Cinemachine` と入力して、パッケージをインストールします。
![](/images/unitychan-movement-3d/package-manager.png)

### FreeLook Camera を追加
キャラクターを追跡するためのオブジェクトである `Cinemachine` の `FreeLook Camera` を作成します。
`Hierarchy (ヒエラルキー)` ウィンドウ左上の「＋」ボタンをクリックし、`Cinemachine` > `Targeted Cameras` > `FreeLook Camera` を選択して作成します。
![](/images/unitychan-movement-3d/create-freelook-camera.png)

### FreeLook Camera のプロパティの編集
作成した `FreeLook Camera` には以下のコンポーネントが自動で追加されています。
`FreeLook Camera` を作成した時点でマウスやスティック入力でカメラ操作が可能になっていますが、必要に応じて以下の設定を調整してください。

- **Cinemachine Camera**
シーン内に `Cinemachine Camera` を表示させるためのコンポーネントです。
`Tracking Target` に追跡対象のキャラクターオブジェクトを設定します。
公式ドキュメントは[こちら](https://docs.unity3d.com/Packages/com.unity.cinemachine@3.1/manual/CinemachineCamera.html)
![](/images/unitychan-movement-3d/cinemachine-camera.png)

- **Cinemachine Orbital Follow**
`Cinemachine Camera` をターゲットの周囲で自由に動かすためのコンポーネントです。
今回は `Top`、`Center`、`Botton` の数値を少し調整しています。
公式ドキュメントは[こちら](https://docs.unity3d.com/Packages/com.unity.cinemachine@3.1/manual/CinemachineOrbitalFollow.html)
![](/images/unitychan-movement-3d/cinemachine-orbital-follow.png)

- **Cinemachine Rotation Composer**
カメラをターゲットの方向に向けるための回転制御コンポーネントです。
キャラクターの中央あたりにカメラを向くようにしたので、`Target Offset` のY座標の値を `1` にしています。
公式ドキュメントは[こちら](https://docs.unity3d.com/Packages/com.unity.cinemachine@3.1/manual/CinemachineRotationComposer.html)
![](/images/unitychan-movement-3d/cinemachine-rotation-composer.png)

- **Cinemachine FreeLook Modifier**
レンズ、ノイズ、ダンピング、構図、カメラ距離などの設定を変化させることができるコンポーネントです。
必須のコンポーネントではないでデフォルトで非アクティブになっています。
公式ドキュメントは[こちら](https://docs.unity3d.com/Packages/com.unity.cinemachine@3.1/manual/CinemachineFreeLookModifier.html)
![](/images/unitychan-movement-3d/cinemachine-freelook-modifier.png)

- **Cinemachine Input Axis Controller**
`Cinemachine Camera` をユーザー入力やスクリプトで動かすためのコンポーネントです。
デフォルトで Cinemachine が用意した Input Action が設定されているので、本来は自分で用意したものに変えた方がいいと思いますが、今回は一応動いているのでこのままデフォルトのままにします。
公式ドキュメントは[こちら](https://docs.unity3d.com/Packages/com.unity.cinemachine@3.1/manual/CinemachineInputAxisController.html)
![](/images/unitychan-movement-3d/cinemachine-input-axis-controller.png)

### 一旦実行して動作確認
Unity の再生ボタンを押して動作を確認します。
マウスや右スティックを動かすと、カメラがキャラクターを追跡しながら周囲を回転します。

![](/images/unitychan-movement-3d/unitychan-freelook-camera.gif)
:::message
マウスカーソルを非表示にするには別途処理が必要ですが、本記事では割愛します。
:::

# カメラ方向に合わせた移動の実装
キャラクターの基本的な移動と、Cinemachine によるカメラ追跡の準備が整いました。
ここからは本記事の本題であるカメラ方向を基準にしたキャラクターの移動処理を実装していきます。

### スクリプト修正
カメラ方向に合わせて移動させるため、`PlayerController.cs` を修正します。

まず、移動方向を計算するために カメラの `Transform` を取得するフィールドを追加します。
```cs:PlayerController.cs
[SerializeField]
private Transform cameraTransform;
```
`Inspector (インスペクター)` で `Main Camera` の `Transform` を設定してください。
![](/images/unitychan-movement-3d/attach-camera-transform.png)

次に、移動方向をワールド座標基準からカメラ基準に変更します。
そのために、カメラのZ軸とX軸を使って移動ベクトルを計算する処理を追加し、既存の `moveVec` を置き換えます。
```diff cs:PlayerController.cs
+   // カメラの向き
+   var forward = cameraTransform.forward;  // Z軸
+   var right = cameraTransform.right;      // X軸
+
+   // 上下方向の成分を除去
+   forward.y = 0f;
+   right.y = 0f;
+
-   // 入力方向をワールド空間に変換
-   var moveVec = new Vector3(input.x, 0f, input.y);
+   // 入力値とカメラ向きに応じて移動ベクトルを計算
+   var moveVec = forward * input.y + right * input.x;
```
この変更により、前後移動がカメラのZ軸方向、左右移動がX軸方向に連動し、TPS やアクションゲームで一般的なカメラの向きへ進む操作が可能になります。

:::details ソースコード全体
```cs:PlayerController.cs
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerController : MonoBehaviour
{
    [SerializeField]
    private Animator animator;
    [SerializeField]
    private CharacterController characterController;
    [SerializeField]
    private float locomotionSpeed;
    [SerializeField]
    private Transform cameraTransform;

    private InputAction moveAction;
    private readonly int speedHash = Animator.StringToHash("Speed");

    private void Start()
    {
        moveAction = InputSystem.actions.FindAction("Move");
    }

    private void FixedUpdate()
    {
        // 入力値
        var input = moveAction.ReadValue<Vector2>();

        // カメラの向き
        var forward = cameraTransform.forward;  // Z軸
        var right = cameraTransform.right;      // X軸

        // 上下方向の成分を除去
        forward.y = 0f;
        right.y = 0f;

        // 入力値とカメラの向きに応じて移動ベクトルを計算
        var moveVec = forward * input.y + right * input.x;

        // 移動入力の大きさを取得（小さなノイズは0として扱う）
        var moveMag = moveVec.magnitude > 0.1f ? moveVec.magnitude : 0f;

        // 実際の移動
        characterController.Move(moveVec * moveMag * locomotionSpeed * Time.fixedDeltaTime);

        if (moveVec.sqrMagnitude > 0.1f)
        {
            // 向きを移動方向に合わせる
            var targetRotation = Quaternion.LookRotation(moveVec);
            transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, Time.deltaTime * 10f);
        }

        // アニメーション操作
        animator.SetFloat(speedHash, moveMag);
    }
}
```
:::

### カメラ基準移動ベクトルの計算原理
キャラクターをカメラが向いている方向に合わせて動かすためのソースコードについて、数式なども用いながら解説します。

1. **カメラのZ軸とX軸を取得する**
カメラの向いている方向を示すベクトルを取得します。
これらはローカル座標系の軸をカメラの回転行列Rを用いて変換したものです。
  - Z軸：前方向 = ローカルZ軸を回転した方向
  - X軸：右方向 = ローカルX軸を回転した方向

$$forward = R \cdot (0,0,1), \quad right = R \cdot (1,0,0)$$
```cs
// カメラの向き
var forward = cameraTransform.forward;  // Z軸
var right = cameraTransform.right;      // X軸
```

2. **カメラのY軸を除く**
カメラが上や下を向くと、`forward` や `right` に Y方向の成分（上下方向）が混ざります。
そのまま使うと、キャラクターが上下に移動してしまうため、地面に平行な方向のみを使用します。
これにより、キャラクターは地面に平行な方向のみで移動します。
```cs
// 上下方向の成分を除去
forward.y = 0f;
right.y = 0f;
```

3. **入力ベクトルをカメラ基準で合成**
入力値をカメラのZ軸とX軸に沿って合成することで、カメラが向いている方向に動く移動ベクトルを作ります。
- 入力値Yの分だけカメラの前へ進む
- 入力値Xの分だけカメラの右へ移動する
$$\vec{m} = \hat{f} \cdot input_y + \hat{r} \cdot input_x$$
```cs
// 入力値とカメラ向きに応じて移動ベクトルを計算
var moveVec = forward * input.y + right * input.x;
```

### 最終動作確認
Unity の再生ボタンを押して動作を確認します。
カメラを動かしながら、キャラクターを移動させてみてください。
カメラ方向に従ってキャラクターがスムーズに移動しているはずです。
![](/images/unitychan-movement-3d/unitychan-movement.gif)

# 最後に
Unity の機能が優秀すぎて、コードをほとんど書かずにカメラの向きに合わせた移動処理が実現できました。
実際にゲームを制作する際は、もう少し細かく設定等をする必要があると思いますが、これだけでも十分理想に近い処理ができます。
