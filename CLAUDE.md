# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

「8番出口」のエッセンス（同じ通路を繰り返し歩く／異変があれば引き返す／正解を重ねて出口8へ／
異変なしの回も混ざる疑心暗鬼）だけを2Dに抽出した異変発見ゲーム。舞台は `mochi-jump` と同じ
和菓子商店街で、通行人は同作の「もち」。3D・歩行アクション・ホラー演出は意図的に持たない。

姉妹リポジトリ（`mochi-jump` / `neon-void` / `tennis-game`）と同じ構成: **`index.html` 一枚、
Canvas 2D、ビルド無し、依存なし、GitHub Pages**。ツールを足さないこと。

## Commands

- ローカル実行: `python -m http.server 8470` してから `http://localhost:8470/index.html`
  （`file://` でも動くが、`PlayCounts` は localhost/file では Firestore に書かない）
- 検証: Playwright headless で `window.__exit` を叩く（下記）。自動テスト基盤は無い
- デプロイ: `main` に push すると `.github/workflows/deploy-pages.yml`（静的アップロードのみ）が
  Pages を更新。初回は Pages サイトが無いと `configure-pages` が失敗するので
  `gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow` で先に作る（2026-09-22 実際に踏んだ）

## Architecture（`index.html` 内の番号付きモジュール）

1. **Sfx** — WebAudio。効果音（`step/good/bad/clear`）と BGM（先読み 0.6 秒のスケジューラ、
   `tennis-game`/`neon-void` と同じ方式）。曲は BPM 96 の Am7→Dm7→E7→Am7 ループで
   「かわいいが少し不穏」。実測 約14 ノード/秒。**ベースは A2(110Hz)〜E3 の帯に置き、キックは
   使わない**: 初版は 120→45Hz のサイン波キックと 73〜82Hz のサブベースがあり「重低音が時々混じって
   耳障り」と指摘された（2026-09-22）。低域を足すときは 110Hz 未満を避ける。翌日さらに「タ、タ、タ」
   （拍頭のタップ音+ハイハット）も耳障りと指摘され**打楽器を全廃**した。パッド+ベース+リードだけの
   アンビエント寄りの曲で、歩行中の足音も 400ms 間隔・小音量の sine に抑えてある。リズム要素を
   足したくなったら先にユーザーに聞くこと。
2. **PlayCounts** — Firestore REST（SDK 無し）。`gameId:"mochi-exit"`、ローカル配信では書かない。
3. **Best** — `localStorage["mochi-exit:best"]` に `{timeMs, miss}` を JSON 1 キーで保存。
4. **Scene / THEMES** — 本作の核。**テーマ = 通りそのもの**で、店・アイテム・異変がテーマごとに別
   （2026-09-23、「テーマごとにアイテムや異変を変えて、各30個に」という要望で色違いから作り替えた）:
   `dusk`=夕焼けのもち屋 / `morning`=朝の茶屋 / `twilight`=たそがれの縁日。各テーマは
   `{id, label, hold, clouds, sky, farShops, farRoof, floor, floorBase, build(s), back(s,cam,t), front?(s,cam,t), anomalies:[30]}`。
   **通りはプレイヤーが選ぶ**: 「はじめる」→ `openSelect()` の選択画面（`#select`）で3つの通り
   ＋「おまかせ（ランダム）」から選ぶ（2026-09-23 要望）。タイトル自体にはテーマ選択を置かない
   （タイトルはロゴ+はじめる+あそびかただけ、という固定方針のため）。カードのサムネイルは
   `renderScene(buildScene(0))` を `toDataURL` したもの＝本番と同じ絵なので、描画を変えれば自動で追従する。
   `startGame(themeIdx|null)` で決まり、そのプレイ中はずっと同じ通り（途中で変わると「異変？」と紛らわしい）。
   「もういちど」は同じ選び方（`lastThemeChoice`、おまかせならまた抽選）で始める。`buildScene(round)` は全テーマ共通の部品
   （空・遠景・床タイル・影・出口看板・もち）を作ってから `theme.build(s)` でそのテーマの部品を足す。
   `s.theme` をシーンに持たせてあり、`renderScene()` は `drawSky→drawFarShops→drawFloor→theme.back→
   drawExitSign→drawMochis→theme.front→夜の暗幕→右上チップ` の順に描く。
   **異変は `theme.anomalies[i].apply(scene)` でこのオブジェクトを書き換えるだけ**で、描画側は異変の存在を
   知らない。id はテーマ内で一意（`exit_number`・`mochi_facing` などは別テーマに同名があってよい。
   発見記録はテーマごとに分けて保存する）。各テーマ level 1:9 / 2:10 / 3:11。
   `desc` は**画面には出さない**（判定後は「正解」「不正解… 出口0に戻る」だけ。何が異変だったかを
   教えると詰まらなくなる、というユーザー方針。2026-09-22）。`desc` はデバッグ用の識別ラベル。
   もちは全テーマ共通だが**持ち物がテーマごとに違う**（`hold`: 団子串 / 湯呑 / 金魚袋。縁日には
   風船に持ち替える異変もある）。持ち物で左右非対称にしてあるので `mochi_facing` が成立する。
   `pickAnomaly(round)`: `currentTheme.anomalies` から出現率 `ANOMALY_RATE=0.6`、`usedIds` で一巡するまで
   再出題しない、`allowedLevels(round)` は `<3:[1] / <6:[1,2] / それ以上:[1,2,3]`、最上位 level を重み 2 で優先。
   **1回脱出した後（`foundUnlocked`）は未発見の異変だけから出す**（「脱出後はまだ出ていない/正解していない
   異変を出して」という要望。2026-09-23）: round に合う level の未発見 → 無ければ通常の出題（発見済みも出る）。
   **level を越えて未発見を出すことはしない**（最初は level を問わず出していたが、序盤に微妙な異変が出るのは
   困る、序盤は簡単なものに限りたい、とユーザーが決めた。2026-09-25）。
   出現率 0.6 は変えない（異変なしの回が混ざる疑心暗鬼は残す）。
   **Found**（3b）: 異変のある通路で正しく「引き返す」を選ぶと、その異変 id を
   `localStorage["mochi-exit:found"] = {テーマid:[id...]}` に記録する（未クリアでも記録はする）。
   選択画面のカードにも「見つけた異変 n / 30」を出す（脱出後のみ）。表示は**1回脱出してから**（`Best.load().timeMs > 0` → `foundUnlocked`）で、キャンバス右上の
   テーマ名チップの左に「発見 n/30」を出す。結果画面にもそのテーマの発見数と今回の新規数(+k)を出す
   （「一回クリアしたあと、見つけた異変の数をカウントして画面の端に」という要望。2026-09-23）。
5. **描画** — `renderScene(scene, camX, t)`。部品ごとの `draw*` を奥から順に呼ぶ。
   `camX` は歩行演出の横スクロール（遠景は 0.5 倍のパララックス）。静止中は rAF を回さない。
   歩行演出 `walk(dirSign, prepare, arrive)` は**継ぎ目のない連続スクロール**: `prepare()` で次の
   シーンを先に作り、今の通路と次の通路を横に並べて（`clip` してから `renderScene`）1画面ぶんを
   1.4 秒の ease-in-out で流す。暗転・ワイプ・出口番号の表示・周辺減光は**すべて廃止**した
   （「切り替わったと分からないレベルで滑らかに」という要望。周辺減光は「丸い画面が出る」、
   ワイプ+出口番号は「切り替えが目立ちすぎる」と不評だった）。継ぎ目を消すため
   **遠景のパララックスは 0.5 倍・周期 160px**（1画面 640px 進むと遠景は 320px = 2 周期ぶん動く）、
   床タイルは周期 64px（640 の約数）にしてある。倍率や周期を変えるときはこの整数倍関係を保つこと。
   `newScene()` は `phase==="walking"` の間は描画しない（walk() が2枚並べて描く）。
6. **Game** — `phase: intro | idle | walking | fading | ended`。`startGame()` はまず `startIntro()` で
   **異変の無い「いつもの通路」を判定なしで見せる**（8番出口の最初の通路と同じ。基準を覚える
   フェーズが無いと何が異変か判断できない、というユーザー指摘で追加）。「覚えた！」で `leaveIntro()`
   → `walk()` → 最初の判定シーンへ。以降は `choose(dir)` が判定→`walk()`→`newScene()`。
   `round>=8` なら `walk()` の `prepare` で異変なしの出口8の通路を作り、到着後に `finish()`。
   画面は `SCREENS=["title","play","result"]` + `show()`。**タイトルはロゴ+はじめる+あそびかた
   だけ**で、ルール文は `#howtoOverlay` モーダル（ユーザーの固定方針。メモリ
   `game-title-simple-howto-button`）。`#title` に `position:relative` を足さないこと
   （`.screen` の `absolute;inset:0` が外れてタイトルが内容の高さに縮み、下が空く。実際に踏んだ）。

### 異変を追加するとき

1. そのテーマの `build(s)` に部品の状態を足す（`back`/`front` の描画にも反映する）
2. そのテーマの `anomalies` に `{id, level, desc, apply}` を1行足す（**各テーマ30個・level 9/10/11 を保つ**）
3. **必ず視認性を自動検証する**: `window.__exit.setTheme(i)`（`0..themeCount-1`）で全テーマに切り替え、
   各テーマで以下を行う。`window.__exit.renderOnly(id, round)` は判定に触れず
   その異変だけを描いた `toDataURL()` を返す。基準（`renderOnly(null, round)`）とのピクセル差分が
   小さい異変は「見えていない」。初版で `mochi_facing` は**差分 0**（もちが左右対称で向きを変えても
   何も変わらなかった）、`sign_typo`（「もぢ」の濁点 2 つ）は 31px で発覚し、団子串を持たせる／
   「もち家」に変えて直した。しきい値は 640×400 で **80px 以上**（level 3 の微妙な異変がこの帯）。**差分は輝度で数える**ので、
   色相だけ変える異変は通らない（赤い縁台→青い縁台は輝度差 11 で不合格だった。明るさも変わる色を選ぶ）。
   2026-09-23 の90個化では風鈴の短冊の色替えが 63px で落ち、短冊を大きくして 90px にした。
   スマホでは描画が約 0.6 倍になるので、これ以下だと実機で見つけられない。
4. `desc` は「〜だった／〜ていた」の過去形で統一（画面には出さないが一覧で読みやすいように）

### 描画レイアウトの制約

- `VIEW_W=640, VIEW_H=400, GROUND_Y=300`。部品の x 座標は固定配置で、重なりを避けた。
  **全テーマ共通で空けておく場所**: 右上 x≥440・y<30（テーマ名と発見数のチップ）、出口看板 x480–590・y44–114、
  もちの立ち位置 x≈295–360・y≈245–300。
  - もち屋: ポスター 90–160 / 団子台 175–285 / 扉 360–422 / 猫 455 / 臼 486–554 / 街灯 600、提灯ロープは x440 まで
  - 茶屋: 太陽(80,56) / 障子 60–170 / 野点傘 54–246 / 縁台 58–244 / お品書き 252–346 / 格子戸 360–442 / 竹垣 468–626
  - 縁日: 提灯列 x16–440 / 金魚すくい 40–260 / わたあめ 284–476 / 太鼓 530–590 / 月(600,168) / 星(612,112)
  提灯はロープ吊り（y=30）で看板（y=40–80）と重ならない位置にした（初版は重なっていた）。
- もちは `mochi-jump` の `drawPlayer()` の顔を流用し、**団子串を片手に持たせて左右非対称**にしてある
  （`mochi_facing` 異変が成立するため。外さないこと）。
- `#canvasWrap` は flex 縦並び中央揃え、選択ボタンは canvas 直下の通常フロー、
  `#stage` は `100dvh`、ボタンは `pointerdown` で即反応 —
  Uniko の CLAUDE.md / メモリ `mobile-game-layout-touch` の4原則そのまま。

## `window.__exit` — テストフック

`setTheme(i)`（`0..themeCount-1`。`anomalies`・`renderOnly` は現在テーマのものになる）/ `themeId()` /
`foundCount()` / `setFoundUnlocked(bool)` / `debugState()` / `skipIntro()`（intro を演出なしで抜ける。テストは `startGame()` の直後に必ず呼ぶ）/ `forceAnomaly(id|null)`（次の `newScene` で1回だけ効く。`undefined` でランダムに戻る）/
`choose("go"|"back")` / `startGame()` / `anomalies` / `renderOnly(id, round)` / `pickAnomaly(round)` /
`resetUsed()`。通しテストは「`forceAnomaly(null)` → 現在シーンの `anomaly` を見て正解側を `choose`」を
8回で `#result` が active になることを確認する。`walk()` の演出が 1.4 秒あるので各手の後に
1900ms 待つ。

## Scope

出口8で脱出する1ステージのみ（通りは3種からランダム）。ランキング（Firestore 書き込み）、ホラー演出、
3D は対象外。異変は各テーマ30種・計90種（各テーマ level 1: 9 / level 2: 10 / level 3: 11）。
