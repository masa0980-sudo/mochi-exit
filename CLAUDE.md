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
   「かわいいが少し不穏」。実測 約14 ノード/秒。
2. **PlayCounts** — Firestore REST（SDK 無し）。`gameId:"mochi-exit"`、ローカル配信では書かない。
3. **Best** — `localStorage["mochi-exit:best"]` に `{timeMs, miss}` を JSON 1 キーで保存。
4. **Scene / ANOMALIES** — 本作の核。`buildScene(round)` が**部品オブジェクトの集合**
   （空・のれん・看板・提灯配列・団子の色順・臼の湯気・ポスター・街灯・扉・出口看板・もち・猫・
   床タイル・影の向き・時計）を返す。**異変は `ANOMALIES[i].apply(scene)` でこのオブジェクトを
   書き換えるだけ**で、描画側は異変の存在を知らない。`level` 1=明らか / 2=中間 / 3=微妙、
   `desc` は判定後のメッセージに出す。
   `pickAnomaly(round)`: 出現率 `ANOMALY_RATE=0.6`、`usedIds` で一巡するまで再出題しない、
   `allowedLevels(round)` は `<3:[1] / <6:[1,2] / それ以上:[1,2,3]`、最上位 level を重み 2 で優先。
5. **描画** — `renderScene(scene, camX, t)`。部品ごとの `draw*` を奥から順に呼ぶ。
   `camX` は歩行演出の横スクロール（遠景は 0.3 倍のパララックス）。静止中は rAF を回さない。
6. **Game** — `phase: idle | walking | fading | ended`。`choose(dir)` が判定→`walk()` 演出→
   `newScene()`。`round>=8` で `finish()`。画面は `SCREENS=["title","play","result"]` + `show()`。

### 異変を追加するとき

1. `buildScene()` に部品の状態を足す（描画にも反映する）
2. `ANOMALIES` に `{id, level, desc, apply}` を1行足す
3. **必ず視認性を自動検証する**: `window.__exit.renderOnly(id, round)` は判定に触れず
   その異変だけを描いた `toDataURL()` を返す。基準（`renderOnly(null, round)`）とのピクセル差分が
   小さい異変は「見えていない」。初版で `mochi_facing` は**差分 0**（もちが左右対称で向きを変えても
   何も変わらなかった）、`sign_typo`（「もぢ」の濁点 2 つ）は 31px で発覚し、団子串を持たせる／
   「もち家」に変えて直した。しきい値は 640×400 で **80px 以上**（level 3 の微妙な異変がこの帯）。
   スマホでは描画が約 0.6 倍になるので、これ以下だと実機で見つけられない。
4. `desc` は「〜だった／〜ていた」の過去形で統一（判定後に「正解：〜」「見逃した… 〜」と続く）

### 描画レイアウトの制約

- `VIEW_W=640, VIEW_H=400, GROUND_Y=300`。部品の x 座標は固定配置で、重なりを避けた:
  ポスター 90–160 / 団子台 175–285 / もち 320 / 扉 360–422 / 猫 455 / 臼 486–554 / 街灯 600。
  提灯はロープ吊り（y=30）で看板（y=40–80）と重ならない位置にした（初版は重なっていた）。
- もちは `mochi-jump` の `drawPlayer()` の顔を流用し、**団子串を片手に持たせて左右非対称**にしてある
  （`mochi_facing` 異変が成立するため。外さないこと）。
- `#canvasWrap` は flex 縦並び中央揃え、選択ボタンは canvas 直下の通常フロー、
  `#stage` は `100dvh`、ボタンは `pointerdown` で即反応 —
  Uniko の CLAUDE.md / メモリ `mobile-game-layout-touch` の4原則そのまま。

## `window.__exit` — テストフック

`debugState()` / `forceAnomaly(id|null)`（次の `newScene` で1回だけ効く。`undefined` でランダムに戻る）/
`choose("go"|"back")` / `startGame()` / `anomalies` / `renderOnly(id, round)` / `pickAnomaly(round)` /
`resetUsed()`。通しテストは「`forceAnomaly(null)` → 現在シーンの `anomaly` を見て正解側を `choose`」を
8回で `#result` が active になることを確認する。`walk()` の演出が約 1 秒あるので各手の後に
1200ms 待つ。

## Scope

出口8で脱出する1ステージのみ。ランキング（Firestore 書き込み）、複数ステージ、ホラー演出、
3D は対象外。異変は 21 種（level 1: 6 / level 2: 7 / level 3: 8）。
