---
title: "WindowsでスクショをQRデコード"
emoji: "🖥️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [PowerShell]
published: true
---

# 導入

- リモート会議でパワポ資料やpdfが画面共有されている時にQRコードが映っていることがある
- QRコードの中身(URLなどの文字列)を即知りたい(スマホを使うとかはナシで…)

## recipe

- 環境はwindows、QRデコードはzxingを使うので、zxing.dllを入手しておいてください
- 実行すると画面のスクショをとってユーザの画像フォルダに保存しつつ、画像ファイルのEXIFのタイトル欄にQRコード文字列を書き込みます。

```powershell
Add-Type -AssemblyName System.Windows.Forms;
Add-Type -Path ".\zxing.dll";

$reader = New-Object -TypeName ZXing.BarcodeReader;
$reader.Options.TryHarder = 1;

$b = New-Object System.Drawing.Bitmap([System.Windows.Forms.Screen]::PrimaryScreen.Bounds.Width, [System.Windows.Forms.Screen]::PrimaryScreen.Bounds.Height);
$g = [System.Drawing.Graphics]::FromImage($b);
$g.CopyFromScreen((New-Object System.Drawing.Point(0,0)), (New-Object System.Drawing.Point(0,0)), $b.Size);
$g.Dispose();
$f = ("$($env:USERPROFILE)\Pictures\cap{0:yyyyMMdd_HHmmss}" -f (date))+".jpg"
$b.Save($f, [System.Drawing.Imaging.ImageFormat]::Jpeg)

$d = $reader.DecodeMultiple($b) | %{
  $dec = $_
  if ($dec -and $dec.Text){
    $dec.Text
  }
}
$b.Dispose()

$s = [System.IO.File]::Open($f, [System.IO.FileMode]::Open, [System.IO.FileAccess]::Read, [System.IO.FileShare]::ReadWrite + [System.IO.FileShare]::Delete)
$b = New-Object System.Drawing.Bitmap($s)
remove-item $f
$p = $b.PropertyItems | Select-Object -First 1

$p.Id = 270
$p.Type = 2
$p.Value = [System.Text.Encoding]::UTF8.GetBytes($d+"`0")
$p.Len = $p.Value.Count
$b.SetPropertyItem($p)

$b.Save($f, [System.Drawing.Imaging.ImageFormat]::Jpeg)
$b.Dispose()
```

- powershell呼び出しは各自好きな方法で。自分はpathの通ったところにbatファイルを置いて、batファイル内でpowershell呼び出ししています
- 黒い画面をなるべく最小化するなどは各自好きな方法でやってください。
- QRコードだけ知りたくて画像保存不要な場合は、後半の画像保存処理をやらずにclip.exeとかにパイプでも可

# 学び

- 驚くべきことに、EXIFプロパティ(system.drawing.imaging.propertyitem)はコンストラクタが存在しないのである！
  - <https://docs.microsoft.com/en-us/dotnet/api/system.drawing.imaging.propertyitem>
- 既存プロパティをコピるしかなく、一旦jpgで保存してそれを読み込んで上書きするという無茶をしている

