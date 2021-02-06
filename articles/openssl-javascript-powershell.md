---
title: "openssl equivalentなAESをchromeでもPowershellでも"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Chrome,JavaScript,PowerShell]
published: true
---

# 導入

- 社用PCに勝手なアプリケーションを入れないのは、大人の嗜みである。
- プレーンなPC環境でも、パスワードのメモとかは暗号化しておきたい。
- chromeのAPIは使うがjsライブラリは使わない方向で。
- opensslと同じ動きにして、相互運用出来ると便利かも。
- (免責)この実装でセキュリティを担保出来る保証はありません。

## encrypt

まずCLI

```bash
  echo "message" | openssl enc -aes-256-cbc -a -pbkdf2 -md sha512 -p -pass pass:hoge
```

これと等価なことをjavascript(chromeでのみ確認)でやる。

```javascript
  c={text:"message",algenc:"AES-CBC",algbit:256,algkd:"pbkdf2",alghash:"SHA-512",kdi:10000,c:crypto,u:Uint8Array,e:x=>(new TextEncoder()).encode(x)};
  c.passbin=(new TextEncoder()).encode("hoge");
  c.salt=c.c.getRandomValues(new c.u(8))
  c.c.subtle.importKey("raw",c.passbin,{name:c.algkd},!1,["deriveBits"])
    .then(km => c.c.subtle.deriveBits( { name: c.algkd, salt: c.salt, iterations: c.kdi, hash: c.alghash }, km, 8*(32+16) )) // key(32)+iv(16)
    .then(b => (c.db=b)&&(c.iv=b.slice(32,48))&&c.c.subtle.importKey("raw",b.slice(0,32),c.algenc,!1,["encrypt"]))
    .then(k  => (c.key=k)&&c.c.subtle.encrypt({name:c.algenc,iv:c.iv},c.key,c.e(c.text)))
    .then(e=>c.encrypted=window.btoa(String.fromCharCode(...([...c.e("Salted__"),...c.salt,...(new c.u(e))]))))
    .finally(()=>console.log(c.encrypted))
```

powershellでもやる。

```powershell
  function enc($s, $p){
    $salt = New-Object byte[] 8
    $rng  = New-Object System.Security.Cryptography.RNGCryptoServiceProvider
    $rng.getBytes($salt)
    $kiv  = (New-Object System.Security.Cryptography.Rfc2898DeriveBytes([System.Text.Encoding]::UTF8.GetBytes($p), $salt, 10000, [System.Security.Cryptography.HashAlgorithmName]::SHA512)).GetBytes(48)
    $aes = New-Object System.Security.Cryptography.AesManaged
    $aes.KeySize   = 256
    $aes.BlockSize = 128
    $aes.Mode      = [System.Security.Cryptography.CipherMode]::CBC
    $aes.Padding   = [System.Security.Cryptography.PaddingMode]::PKCS7
    $aes.Key       = $kiv[0..31]
    $aes.IV        = $kiv[32..47]
    $encryptor = $aes.CreateEncryptor($aes.Key, $aes.IV)
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($s)
    $ms = [System.IO.MemoryStream]::new()
    $ms.Write([System.Text.Encoding]::UTF8.GetBytes("Salted__") + $salt, 0, 16)
    $cs = New-Object System.Security.Cryptography.CryptoStream($ms, $encryptor, [System.Security.Cryptography.CryptoStreamMode]::Write)
    $cs.Write($bytes, 0, $bytes.length)
    $cs.FlushFinalBlock()
    $cs.Close()
    [System.Convert]::ToBase64String($ms.ToArray())
    $ms.Close()
    $encryptor.Dispose()
    $aes.Dispose()
  }
  enc "message" "hoge"
```

## decrypt

そしてcliでdecrypt。

```bash
echo "U2FsdGVkX1+6SGosjqp5QKdNOwVIn5KAvOQtTqObCAE=" | openssl enc -aes-256-cbc -a -d -pbkdf2 -md sha512 -p -pass pass:hoge
```

等価なjavascript

```javascript
  c={enctext:"U2FsdGVkX1+6SGosjqp5QKdNOwVIn5KAvOQtTqObCAE=",algenc:"AES-CBC",algbit:256,algkd:"pbkdf2",alghash:"SHA-512",kdi:10000,c:crypto,u:Uint8Array};
  c.passbin=(new TextEncoder()).encode("hoge");
  c.encbin=new c.u(window.atob(c.enctext).split("").map(a=>a.charCodeAt(0)));
  c.salt=c.encbin.slice(8,16);
  c.c.subtle.importKey("raw",c.passbin,{name:c.algkd},!1,["deriveBits"])
    .then(km => c.c.subtle.deriveBits( { name: c.algkd, salt: c.salt, iterations: c.kdi, hash: c.alghash }, km, 8*(32+16)) ) // key(32)+iv(16)
    .then(b => (c.db=b)&&(c.iv=b.slice(32,48))&&c.c.subtle.importKey("raw",b.slice(0,32),c.algenc,!1,["decrypt"]))
    .then(k  => (c.key=k)&&c.c.subtle.decrypt({name:c.algenc,iv:c.iv},c.key,c.encbin.slice(16)))
    .then(d=>c.decodedtext=(new TextDecoder("utf-8")).decode(new c.u(d)))
    .finally(()=>console.log(c.decodedtext))
```

powershell

```powershell
  function dec($s, $p){ # base64str
    $encText = [System.Convert]::FromBase64String($s);
    $salt = $encText[8..15]
    $kiv  = (New-Object System.Security.Cryptography.Rfc2898DeriveBytes([System.Text.Encoding]::UTF8.GetBytes($p), $salt, 10000, [System.Security.Cryptography.HashAlgorithmName]::SHA512)).GetBytes(48)
    $aes = New-Object System.Security.Cryptography.AesManaged
    $aes.KeySize   = 256
    $aes.BlockSize = 128
    $aes.Mode      = [System.Security.Cryptography.CipherMode]::CBC
    $aes.Padding   = [System.Security.Cryptography.PaddingMode]::PKCS7
    $aes.Key       = $kiv[0..31]
    $aes.IV        = $kiv[32..47]
    $decryptor = $aes.CreateDecryptor()
    $ms1 = [System.IO.MemoryStream]::new($encText[16..($encText.Length-1)])
    $cs = New-Object System.Security.Cryptography.CryptoStream($ms1, $decryptor, [System.Security.Cryptography.CryptoStreamMode]::Read)
    $ms2 = [System.IO.MemoryStream]::new()
    $cs.CopyTo($ms2)
    $ms1.Close()
    $ms2.Close()
    $cs.Close()
    [System.Text.Encoding]::UTF8.GetString($ms2.ToArray())
    $decryptor.Dispose()
    $aes.Dispose()
  }
  dec "U2FsdGVkX1+6SGosjqp5QKdNOwVIn5KAvOQtTqObCAE=" "hoge"
```

# 学び

- 毎回encrypt結果が異なるのはランダムなソルトが付加されているため
- opensslの標準形式は `Salted__` という文字列とソルト値(8bytes)を最初に付加してくる
- ソルトが8bytesで十分なのか調べたら16bytes以上がいいよ、みたいな情報があったがopensslは過去との互換性でこのフォーマットを変えられないっぽく、将来のIssueになっていた
- ベータ中のopenssl 3.0をコンパイルして動作を見てみたけど、openssl 3.0でも8bytesソルトが標準形式っぽかった。
- encryptにopenssl使わないなら標準形式に拘らなくてもよかった。

