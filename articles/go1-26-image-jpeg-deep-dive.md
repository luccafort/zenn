---
title: "Go 1.26 Draft Release Noteからimage/jpegを読み解く"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "image", "jpeg"]
published: true
published_at: 2025-12-10 00:00
---

本記事は[Money Forward Kansai Advent Calendar 2025](https://adventar.org/calendars/11342) 12月10日の記事です。
前回の記事はnagamoさんの「TODO:公開後に置き換え予定」でした。

## 突然ですが本日が誕生日です

本日12月10日でめでたく（？）40歳の節目を迎えることになりました[@luccafort](https://x.com/luccafort)です。
現代社会の人生がおよそ80年ほどであることを考えるとようやく折り返し地点というべきか、もう折り返し地点というべきか悩み始めています。

少しでもこの記事がよいと感じたら「バッジを贈る」から皆さんの支援をお願いします。
いただいた金銭は技術書、もしくはマネーフォワード 京都開発拠点を会場提供しているコミュニティイベント（Not マネーフォワード主催）の軽食費などに当てようと思います。

さて、本記事は[Go 1.26 Draft Release Note（2025年12月10日時点）](https://tip.golang.org/doc/go1.26)を読んでいたときに気になった `image/jpeg` パッケージについて、どのような変更が入ろうとしているのかをお伝えする内容になっています。

## 本記事の目的と想定読者

### 外向けの綺麗な理由

本記事はGo 1.26のリリース前に `image/jpeg` パッケージにどのような変更が入ったか事前に知ることでリリース後のアップデートで思わぬトラブルに遭遇しないことを目的としています。

### 実際の目的や調べようと思ったきっかけ

……というのは外付けの理由で本当の理由はDraft Release Noteを読んでいたときに以下の文章を見かけ、「どのような変更が入ったのだろう？」と興味、関心が生まれたので調べてみたものになります。

> The JPEG encoder and decoder have been replaced with new, faster, more accurate implementations. Code that expects specific bit-for-bit outputs from the encoder or decoder may need to be updated.

ぼくが現在関わっているプロダクトでは画像を扱うことはあまりないのですが、気になったのは2点。

- Encoder/Decoderのリプレイスにより速くなったこと
- Encoder/Decoderの変更により更新が必要になるかもしれないこと

この2点が気になったため、どのような実装をしたのか、何に気をつけるといいのかについて調べてみようと思いました。

### 対象となる想定読者層

そのため、本記事の想定読者層は以下を想定しています。

- Go 1.26でリリースされる機能に興味がある方
- 普段Goを用いた開発をしており、`jpeg` 画像を取り扱う可能性がある方
- Goでコードを書いたり、読んだりすることが好きな方
- 画像の変換処理などに興味がある方

### 生成AIを調査アシスタントにしている点に関する注意点

本記事は `Cursor@composer 1` に調査をアシストしてもらい、執筆しています。
執筆自体はぼく個人が行っていますが、調査に関しては時間短縮のために生成AIを使って解説をしてもらったり、原因箇所を特定してもらったり、実装されているコードの解説をしてもらっています。
以下のような意見もあり、主張されている内容に対して個人的に同意できる点もありますが、同意できない点もありました。
そのため、本記事は生成AI（LLM）を使って調査、裏付けは人間が行う形で執筆している旨をお伝えさせていただきます。

[アドベントカレンダーをLLMで書くくらいなら何も書かない方がいい。](https://zenn.dev/watany/articles/ad14f8a352d62f)

できるだけ、確認や他の文献などを参照するようにしてはいますが、今回の内容はかなり専門的な部分を含み、自身の理解を超えている点がありました。
そのため、一部誤った情報が含まれている可能性があります、ご容赦いただければと思います。

## Go 1.26 `image/jpeg` ざっくりまとめ

Go 1.26の `image/jpeg` が3秒でわかる、ざっくりまとめです。

- Go 1.26 では `image/jpeg` の中身（Encoder/Decoder）がガッツリ入れ替わっている
- API はほぼそのまま使える、でもEncoder/Decoderは別物に近い
- `jpeg` 画像を生成しているサービスのビジュアルリグレッションテストで、許容範囲度を指定していない場合テストが落ちる可能性がある

### Go 1.26 変更点まとめ

「ざっくりまとめではわからない！」という方に向けたもう少し詳しく書いたものを掲載しておきます。
大きく以下のような点が更新される予定です。

- エンコーダ/デコーダ実装を置き換えることで、より高速・より正確な実装に変更
- パフォーマンスの向上
	- 現状でも十分速いが、特に大きな `jpeg` や複雑な画像ではボトルネックになりうるケースがあったのを改善
	- `decode` 側で約1割強の速度改善が見込まれるパッチが入る
- 数値的な「正確さ」
	- これまでのGoでは量子化・色変換などで多少ラフな部分があり、他実装と比較して差が出ることがあった
	- DCTや量子化周りの扱いが見直され「仕様に忠実な値に近づく」方向の変更を行った
- 壊れた `jpeg` への耐性
	- `restart marker` や `EOF` まわりでエラーや `panic` を起こしやすいケースへの対応
	- `restart marker` まわりの挙動を強化し、「`panic` ではなくエラーを返す」方向の修正が議論されている
- 互換性
	- これまでは「同じ入力 → 同じバイト列の出力」「同じピクセル値」を前提にしたテストが通っていた
	- Encoder/Decoderの置き換えによって、ビット単位での互換性は崩れる可能性あり（ゴールデンファイル比較しているテストなどは見直しが必要になるかも）

## Go 1.26 `image/jpeg` の変更点

それではさっそく実際に変更された箇所を具体的に見ていこうと思います。

### Encoder/Decoder の大幅な実装改善

これまで `image/jpeg` はEncoder/Decoderに対して、パフォーマンス改善やロバストネス改善など、少しずつパッチを当てる形で対応してきました。
今回はこれまでの改善とは異なり、大きく中身を書き換える対応を行っている点に注意ください。
APIとして利用する分にはおそらく問題が起こることはあまりないと予測されますが、中の実装はかなり変わっています。

おそらく本記事の中で最も難解な解説となります。ぼく自身も調べながら書いているので誤っている点があるかもしれません。
できるだけ、他の記事などを読み、表面上の理解はしたつもりで書いていますが、誤った記載がありましたらご連絡ください。
正直書いていても、まだ理解が20%もできていないように感じています。

#### スキャンループの分解と事前計算

大きく変わっている点として、これまでスキャン時のループに多重for文を用いて計算していましたが、事前に値を計算することで、高速化を実現しようとしています。
これによって分岐とループの計算処理が減ることが期待されています。

ref: [image/jpeg: decomposes scan loops and pre-computes values](https://go-review.googlesource.com/c/go/+/125138)

これまでの実装ではDCTブロックを走査するループ内で、毎回その場で行う計算部分が多くありました。
MCU（Minimum Coded Unit）ごとのインデックス計算を行うなど、実装としてわかりやすい反面、ループ内の処理が重くなりやすく、CPUキャッシュ効率や分岐予測の面で不利になりやすい状態でした。

:::details DCTブロックとは？

Discrete Cosine Transform（離散コサイン変換）の略。
`jpeg` では画像を `8×8` の小さなブロックに分割し、各ブロックごとにDCTという周波数変換をします。

### `jpeg` の処理の流れ（超ざっくり）
1. 画像を `8×8` ピクセルごとに分解（≒ DCTブロックの生成）
2. 各ブロックにDCT（周波数分析）をかける
3. 得られた64個の係数を量子化（丸め）してデータ量を減らす
4. 量子化された数値をハフマン符号化して圧縮する（可変長エンコード）

ref: [画像の圧縮（その２） 秋田大学](https://www.mis.med.akita-u.ac.jp/~kata/image/compress/imagecomp2.html)
:::

:::details ハフマンテーブルとは？
`jpeg` のDCT係数（DC/AC）はそのまま8bitや16bitでは保存されません。
代わりにハフマン符号化（可変長ビット列）で書かれています。

画像の周波数データは「0 が大量に出る」「小さな数値が多い」という性質があります。
ハフマン符号の特徴として以下の点があります。
* 出現頻度が高い値は短いビット列で表す
* 出現頻度が低い値は長いビット列で表す

上記の結果として圧縮率が高くなるようになっています。

`jpeg` では通常、「DC係数用ハフマンテーブル」と「AC 係数用ハフマンテーブル」の`2種類 ×（色成分ごとに複数）= 最大4セット` が存在します。

スキャンループ内では、DCTブロックごとにDC → AC の順でハフマンデコード（ビットを読む → 伸長する） を繰り返します。つまり、1ピクセルブロックあたり、64回程度のハフマンデコードが実行されることになります。
`大量の分岐 + ビット演算` が実行される、これが `jpeg decode` の最もCPUを食う処理になります。

refs: 
* [JPEGのハフマン符号 (1) DC成分](https://kurinkurin12.hatenablog.com/entry/20100115/1263566612)
* [IJGソフト内部　ハフマン符号化 2](https://kurinkurin12.hatenablog.com/entry/20100228/1267343152)
* [JPEGのハフマン符号　(3) ハフマンテーブルの記述](https://kurinkurin12.hatenablog.com/entry/20100116/1263624765)
* [JPEGファイルの理解を深める](https://qiita.com/daidai-ok/items/a25e1bca890c27846253)
:::

:::details 量子化テーブルとは
`jpeg` の圧縮の要は量子化 (Quantization) です。
DCTの64係数に対して、64の重み（分母）が用意され、`量子化後係数 = DCT係数 / 量子化テーブルの値` のように「割って丸める」ことで多くの微細情報を落とします

量子化テーブルの値が大きいほど係数はより小さく丸められ、高周波（細かい模様）が削られる。結果、画質は下がるが圧縮率が上がるという仕組みになっています。
デコード時は逆に `復元係数 = jpeg係数 × 量子化テーブル` として元のDCTスペースに戻しています。
この計算（64個分）が各ブロックごとに発生します。
:::

Go 1.25以前では以下の処理を毎回行っていました。

* テーブル参照
* インデックス計算
* 条件分岐

これらの処理を事前計算（pre compute）にすることで、内側のループは係数の読み取りに専念することができるようになります。
内側のループの中で参照テーブルを、外側のループで `pointer` を固定しておくことで以下の効果が期待できます。

* 分岐回数削減
* CPUキャッシュヒット率向上
* ポインタ計算の削減

これらの処理はハフマンデコードの最も重い計算部分なので、改善されることで全体的なデコード処理の高速化につながります。
（とても難しい……）

:::details スキャンループ処理のコードを読む

Go の標準ライブラリでは、`jpeg` デコードの「スキャン」（SOS: Start Of Scan）は `image/jpeg` パッケージの `scan.go` にまとまっており、すべて `(*decoder).processSOS` で処理されます。

[go.dev/image/jpeg/scan.go](https://go.dev/src/image/jpeg/scan.go)

##### MCU数や画像サイズを基に準備する部分

以下はMCU数や画像サイズを事前に準備する実装部分の抜粋です。

```go
// mxx and myy are the number of MCUs (Minimum Coded Units) in the image.
h0, v0 := d.comp[0].h, d.comp[0].v
mxx := (d.width + 8*h0 - 1) / (8 * h0)
myy := (d.height + 8*v0 - 1) / (8 * v0)
if d.img1 == nil && d.img3 == nil {
    d.makeImg(mxx, myy)
}
if d.progressive {
	for i := 0; i < nComp; i++ {
		compIndex := scan[i].compIndex
		if d.progCoeffs[compIndex] == nil {
			d.progCoeffs[compIndex] = make([]block, mxx*myy*d.comp[compIndex].h*d.comp[compIndex].v)
		}
	}
}
d.bits = bits{}
mcu, expectedRST := 0, uint8(rst0Marker)
```

##### MCU を縦（`my`）× 横（`mx`）に走査する二重ループ

```go
for my := 0; my < myy; my++ {
    for mx := 0; mx < mxx; mx++ {
        for i := 0; i < nComp; i++ {
            compIndex := scan[i].compIndex
            hi := d.comp[compIndex].h
            vi := d.comp[compIndex].v
            for j := 0; j < hi*vi; j++ {
                // ここで (bx, by) を計算してブロック単位で処理
                ...
                // 係数読み込みと復号
                ...
            } // for j
        } // for i
        mcu++
        // RST マーカー処理
        ...
    } // for mx
} // for my
```

##### 係数（DC/AC）を読む zig-zag ループ

上記のネストの中で、実際にDCT係数を読み取る部分がさらにループになっています。

```go
zig := zigStart
if zig == 0 {
    zig++
    // DC係数をハフマン復号
    value, err := d.decodeHuffman(&d.huff[dcTable][scan[i].td])
    ...
    b[0] = dc[compIndex] << al
}

if zig <= zigEnd && d.eobRun > 0 {
    d.eobRun--
} else {
    // AC係数のハフマン復号ループ
    huff := &d.huff[acTable][scan[i].ta]
    for ; zig <= zigEnd; zig++ {
        value, err := d.decodeHuffman(huff)
        ...
        if val1 != 0 {
            zig += int32(val0)
            ...
            ac, err := d.receiveExtend(val1)
            ...
            b[unzig[zig]] = ac << al
        } else {
            ...
        }
    }
}
```

:::

### 「DO NOT PANIC」系の修正

これまでは壊れた `jpeg` を読んだときに、`FormatError` や `io.ErrUnexpectedEOF` ではなく `panic` を起こしてしまうケースがあったようです。

ref: [image/gif / image/jpeg / image/png: DO NOT PANIC](https://go-review.googlesource.com/c/go/+/353850)

例えば `decoder.fill()` 内には、バッファ状態がおかしいと `panic("jpeg: fill called when unread bytes exist")` というコードがあり、変な入力からここに到達する可能性がある実装になっていました。

```go:go/src/image/jpeg/reader.go
// fill fills up the d.bytes.buf buffer from the underlying io.Reader. It
// should only be called when there are no unread bytes in d.bytes.
func (d *decoder) fill() error {
	if d.bytes.i != d.bytes.j {
		panic("jpeg: fill called when unread bytes exist")
	}
	// Move the last 2 bytes to the start of the buffer, in case we need
	// to call unreadByteStuffedByte.
	if d.bytes.j > 2 {
		d.bytes.buf[0] = d.bytes.buf[d.bytes.j-2]
		d.bytes.buf[1] = d.bytes.buf[d.bytes.j-1]
		d.bytes.i, d.bytes.j = 2, 2
	}
	// Fill in the rest of the buffer.
	n, err := d.r.Read(d.bytes.buf[d.bytes.j:])
	d.bytes.j += n
	if n > 0 {
		return nil
	}
	if err == io.EOF {
		err = io.ErrUnexpectedEOF
	}
	return err
}
```

前後でどのように変わるかを解説すると以下がわかりやすいかと思います。

* Go 1.25以前： 「壊れた `jpeg` → `runtime panic` → プロセスごと死亡」
* Go 1.26以降： 「壊れた `jpeg` → `jpeg.Decode` がエラーを返す → 呼び出し側でリトライ/ログ出力などで対処可能」

弊社マネーフォワードが提供するWebサービスやCLIツールから見ると「外部入力でプロセスごと落ちる」ケースというのはかなり厳しく、理想的に言えばエラーを返して欲しいと考えるのが自然です。それに対応しようということのようです。
これからは突然の `panic` ではなくエラーが変えることで、リトライ処理や適切なエラー処理を返すことができ、よりユーザーフレンドリーな振る舞いに変わっていくのではないかと思います。

### `restart marker`（RST）の扱いをより厳格に

ref: [image/jpeg: improve handling of JPEG restart markers in non-ideal cases](https://groups.google.com/g/golang-codereviews/c/D4XMjK5aDAE)

これまでは`image/jpeg` 実装は、「仕様通りのキレイな JPEG」を前提にした挙動が強めだったため、`restart marker` 周辺のデータが少しでも「教科書的な `jpeg` から外れている」とエラーになることがありました。

`invalid JPEG format: bad RST marker` のようなエラーで `decode` に失敗することがあったり、条件によっては panic まで行くケースもあったようです。

* Go 1.25以前： 少し壊れた `jpeg` → すぐエラー or 場合によっては `panic`
* Go 1.26以降： 少し壊れた `jpeg` → 可能な範囲で解釈・復旧して画像として `decode` されるケースが増える

ref: [image/jpeg: "bad RST marker" error when decoding #40130](https://github.com/golang/go/issues/40130)

Go 1.26ではLuaの `libjpeg` の「現実世界の壊れた JPEG に対するロバストさ」を参考に、`restart marker` の前にある余計なバイト（`spurious data`）をスキップしたり、予期しないマーカーが来ても なるべく復旧して`decode`を続行するロジックを入れようとしているようです。

### `decode` 性能を約12%改善

ref: [image/jpeg: improve decoder performance by ~12%](https://go-review.googlesource.com/c/go/+/647835)

`jpeg.Decode` は元々そこそこ速いものの大きな `jpeg` 画像（高解像度写真）やバッチ処理で大量に `decode` するケースでは、他の処理に対して目立つボトルネックになることがありました。

GitHubに報告が上がっている以下の例では `pixiv/go-libjpeg/jpeg` に比べ、`native jpeg` が数倍遅いと報告されています。

```
I noticed that the use of decode jpeg is very slow.

decode image jpeg 1920x1080

I test github.com/pixiv/go-libjpeg/jpeg and native jpeg

go 1.10 jpeg.decode ≈ 30 ms cpu ≈ 15 %
libjpeg jpeg.decode ≈ 7 ms cpu ≈ 4 %
```

ref: [image/jpeg: Decode is slow #24499](https://github.com/golang/go/issues/24499)

今回の変更では内部ループやメモリアクセスパターンが改善され、`decode`時間がおよそ1 割ほど短縮される見込みのようです。

#### `unrolling unzig and shift-clamp loops` の解説

ここからは実際に速度改善したコードを元にどのような変更が行われていったかをみたいと思います。

ref: [GitHub PR: image/jpeg: improve decoder performance by ~12% #71618](https://github.com/golang/go/pull/71618)

`reconstructBlock` は `8×8` の DCTブロック1個を「係数→画素」に復元する処理で、画像の全ピクセルに対して何度も呼ばれる超ホットパスです。

```go
// reconstructBlock dequantizes, performs the inverse DCT and stores the block
// to the image.
func (d *decoder) reconstructBlock(b *block, bx, by, compIndex int) error {
```

##### `unrolling unzig` について解説

変更前のコードでは以下のようになっています。

```go
qt := &d.quant[d.comp[compIndex].tq]
for zig := 0; zig < blockSize; zig++ {
    b[unzig[zig]] *= qt[zig]
}
```

変更前のコードにおける `unzig` は「`0..63` → 実際の [行×列] インデックス」の変換テーブルです。
各ループで`unzig[zig]` をロードし、そのインデックスで `b[...]` にアクセスし、`qt[zig]` をかけています。
つまり `64回のループ + 64回のunzig ロード + ループ制御の分岐` が走る計算になります。

変更後のコードでは以下のようになっています。

```go
qt := &d.quant[d.comp[compIndex].tq]

// This sequence exactly follows the indexes of the unzig mapping.
b[0] *= qt[0]
b[1] *= qt[1]
b[8] *= qt[2]
b[16] *= qt[3]
...
```

変更後のコードでは `unzig` テーブルへのロードは当たり前ですが0回になっています。
どの `zig` がどの `index` かは「コンパイル時に確定したリテラル」になっているため、ループカウンタや比較、ジャンプが消え、余分な計算を削減できます。
また `zig++` や `zig < blockSize` など、それに伴う分岐がなくなっています。

すべてのインデックスがコンパイル時定数なのでインデックスが有効な範囲内かどうか判定する `bounds check` の除去が容易になります。
これが「`unrolling unzig`」部分の解説になります。

##### `shift-clamp loops` の解説

変更前のコードでは以下のようになっています。

```go
// Level shift by +128, clip to [0, 255], and write to dst.
for y := 0; y < 8; y++ {
    y8 := y * 8
    yStride := y * stride
    for x := 0; x < 8; x++ {
        c := b[y8+x]
        if c < -128 {
            c = 0
        } else if c > 127 {
            c = 255
        } else {
            c += 128
        }
        dst[yStride+x] = uint8(c)
    }
}
```

変更前のコードではループのたびに以下の処理が64要素分、毎回実行されてしまいます。

- `y8, yStride` を定義する計算
- `y8+x` や `yStride+x` のインデックス計算
- `if` の条件分岐（ `c < -128`, `c > 127`, `else`）

一方で、変更後のコードは以下のようになります。

```go
writeDst := func(index int) {
    c := (*b)[index] + 128
    if c < 0 {
        c = 0
    } else if c > 255 {
        c = 255
    }
    dst[(index/8)*stride + (index%8)] = uint8(c)
}

writeDst(0)
writeDst(1)
writeDst(2)
...
```

`writeDst` はインラインされる前提になっています。index も `0〜63` がすべてコンパイル時定数になるので `(index/8)*stride + (index%8)` が完全に定数折り畳みされ、各要素に対する `dst[...]` のアドレス計算が「ただの即値オフセット」になっています。


また `if` 条件分岐も「`(値 + offset)` を `0〜255` にサチュレートする」標準形 になっています。コンパイラやCPUによってはこの形のほうが最適化しやすいケースがあるようです。
（詳しい人がいたら教えてください）

ともあれ、Decoderの改善やスキャンの改善による速度改善が見込めるため、大量に`jpeg`画像を生成する、もしくは高解像度の画像を扱う際の悩みが1つ改善されそうです。

### `jpeg` 画像のゴールデンテスト/スナップショットテストが失敗するかもしれない

これまで、以下のケースは問題なく、テストを通っていましたが、Go 1.26以降では失敗する可能性があります。

- `jpeg.Encode` の出力バイト列をそのままゴールデンファイル化しているテスト
- `jpeg.Decode` した画素値を「ピクセル単位で完全一致」させるテスト

#### `jpeg.Encode` の出力バイト列をそのままゴールデンファイル化しているテスト

これは前述した `Encoder` 実装が書き換わったことで、同じ `jpeg` 画像を渡したとしても内部的な `jpeg` のバイト列が変わる可能性があるためです。

`jpeg` 規格では「どのハフマンコードを使うか」「どの順番で書くか」はかなり自由度が高くなっています。
そのため、どちらも規格には適合し、（異なるデコーダ実装でも）見た目の画素値はほぼ同じだがバイト列はまったく一致しない、という現象が発生し得ます。

#### `jpeg.Decode` した画素値を「ピクセル単位で完全一致」させるテスト

「デコードしたときに画素値が ±1 程度ズレる」ような現象は、以下のような条件の際を変更、修正したときに起こり得ます。

- IDCT（逆離散コサイン変換）の丸め処理
- クロマサブサンプリングの補間
- 再スタートマーカー（RST）の扱い
- プログレッシブ `jpeg` の複数スキャンの合成順序

これは、デコーダの実装置き換えによって、“実質バグ”だった境界ケースが修正されるケースなどが該当します。
そのため、これまで（Go 1.25以前）と同じ画像を用いたテストとコードを用いても、Go 1.26以降ではテストがFailする可能性があります。
もし、`jpeg` まわりで バイト列一致テスト をしている場合、Go 1.26 に上げると落ちる可能性があるので「画素値レベル比較」もしくは、「許容誤差付き比較」に変えるなどの対策を行うほうが良いでしょう。

## まとめ

今回はGo 1.26の `image/jpeg` に組み込まれる変更点のみを列挙し、各変更点やどのようにコードが書き換わり、その意図は何かを読み解く実験を行いました。
生成AIを解説役として、各コードの意図を説明させることで、ぼく自身知らない単語や理論、計算について学ぶことができました。

普段、プロジェクトマネージャーを行っているとこういったコンピューターサイエンスの分野を学ぶ機会は意識的に取り組まなければなかなか実践できません。
今回はメジャーバージョンの大まかな機能の紹介ではなく、あえて1つのパッケージの変更にフォーカスし、深く掘り下げる形の記事を書いてみることで、自分自身と読者にとって勉強になるのではないか？という思いつきからスタートしましたが、想像以上に楽しくコードを読むことができました。

一部解説に関しては、自分自身もまだまだ理解できていない点があり、間違えている可能性がありますが、それも含めてよい学習体験になったのではないかと思います。