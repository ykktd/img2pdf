# img2pdf

複数の画像をブラウザ内で圧縮し、1画像1ページのPDFとして書き出す静的Webアプリです。

## Features

- JPG / PNG / WebPなどの複数画像を選択
- ドラッグ＆ドロップで追加
- サムネイル付きリストで順番並び替え
- 画質スライダーによるJPEG圧縮
- A4縦 / A4横 / 画像サイズそのまま
- 余白あり / なし
- 生成したPDFをダウンロード

画像処理とPDF生成はすべてブラウザ内で行われ、画像は外部サーバーへ送信されません。

## Development

```sh
npm install
npm run dev
```

## Build

```sh
npm run build
```

GitHub Pagesでは `/img2pdf/` 配下で配信される想定です。
