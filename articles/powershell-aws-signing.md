---
title: "WindowsPCからS3にGET/PUTしたい"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PowerShell", "AWS", "S3"]
published: true
---

## tl;dr;

* Windows PCからS3にファイルアップロードしたいが、AWS-SDKは入れたくない、入れられない環境
* batファイルなどから下記powershellスクリプトを起動して、ファイルアップロード/ダウンロードさせる。
* `s3:getObject`, `s3:putObject`のみ可能な、IAMキーを発行して、そのアクセスキーとシークレットキーを仕込む。
* v4署名にあたって参考にしたのは
  * <https://docs.aws.amazon.com/ja_jp/general/latest/gr/signature-v4-examples.html>
  * <https://stackoverflow.com/questions/57835886/how-can-i-download-a-file-from-s3-via-powershell-without-installing-aws-sdk>

```powershell
class AWS_S3{
  [string] $access_key = "XXXX"
  [string] $secret_key = "XXXXXXXX"
  [string] $region     = "ap-northeast-1"
  [string] $bucketname = "yourbucket"
  [object[]] HmacSHA256($message, $secret){
    $hmacsha     = New-Object System.Security.Cryptography.HMACSHA256
    $hmacsha.key = $secret
    $signature   = $hmacsha.ComputeHash([Text.Encoding]::ASCII.GetBytes($message))
    return $signature
  }
  [object[]] getSignatureKey($key, $dateStamp, $regionName, $serviceName){
    $kSecret  = [Text.Encoding]::UTF8.GetBytes(('AWS4' + $key).toCharArray())
    $kDate    = $this.HmacSHA256($dateStamp    ,$kSecret);
    $kRegion  = $this.HmacSHA256($regionName   , $kDate);
    $kService = $this.HmacSHA256($serviceName  , $kRegion);
    $kSigning = $this.HmacSHA256("aws4_request", $kService);
    return $kSigning
  }
  [string] hash($request){
    $hasher  = [System.Security.Cryptography.SHA256]::Create()
    $content = [Text.Encoding]::UTF8.GetBytes($request)
    $bytes   = $hasher.ComputeHash($content)
    return ($bytes|ForEach-Object ToString x2) -join ''
  }
  [string] getURL($key, $method){
    $hostaws   = $this.bucketname + '.s3-' + $this.region + '.amazonaws.com'
    $now       = [DateTime]::UtcNow
    $amz_date  = $now.ToString('yyyyMMddTHHmmssZ')
    $datestamp = $now.ToString('yyyyMMdd')
    $signed_headers   = 'host'
    $credential_scope = $datestamp + '/' + $this.region + '/s3/' + 'aws4_request'
    $canonical_querystring = 'X-Amz-Algorithm=AWS4-HMAC-SHA256'
    $canonical_querystring += '&X-Amz-Credential=' + [uri]::EscapeDataString(($this.access_key + '/' + $credential_scope))
    $canonical_querystring += '&X-Amz-Date=' + $amz_date
    $canonical_querystring += '&X-Amz-Expires=86400'
    $canonical_querystring += '&X-Amz-SignedHeaders=' + $signed_headers
    $canonical_headers = 'host:' + $hostaws + "`n"
    $canonical_request = "$($method)`n"
    $canonical_request += "/" + $key + "`n"
    $canonical_request += $canonical_querystring + "`n"
    $canonical_request += $canonical_headers + "`n"
    $canonical_request += $signed_headers + "`n"
    $canonical_request += "UNSIGNED-PAYLOAD"
    $algorithm = 'AWS4-HMAC-SHA256'
    $canonical_request_hash = $this.hash($canonical_request)
    $string_to_sign = $algorithm + "`n"
    $string_to_sign += $amz_date + "`n"
    $string_to_sign += $credential_scope + "`n"
    $string_to_sign += $canonical_request_hash
    $signing_key = $this.getSignatureKey($this.secret_key, $datestamp, $this.region, "s3")
    $signature = $this.HmacSHA256($string_to_sign, $signing_key)
    $signature = ($signature|ForEach-Object ToString x2) -join ''
    $canonical_querystring += '&X-Amz-Signature=' + $signature
    $request_url = "https://" + $hostaws + "/" + $key + "?" + $canonical_querystring
    return $request_url
  }
  [string] get($key, $filepath){
    $hostaws = $this.bucketname + '.s3-' + $this.region + '.amazonaws.com'
    $url     = $this.getURL($key, "GET")
    return Invoke-RestMethod -Method GET -OutFile $filepath -Headers @{'host'=$hostaws} -Uri $url;
  }
  [string] put($filepath, $key){
    $hostaws = $this.bucketname + '.s3-' + $this.region + '.amazonaws.com'
    $url     = $this.getURL($key, "PUT")
    return Invoke-RestMethod -Method PUT -InFile $filepath -Headers @{'host'=$hostaws} -Uri $url;
  }
}

$s3 = New-Object AWS_S3
$s3.put("localfile.txt", "s3path/to/file.txt")
$s3.get("s3path/to/file.txt", "got-file.txt")
```
