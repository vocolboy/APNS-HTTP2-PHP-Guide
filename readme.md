# APNS (HTTP/2) with PHP simple push ( Ubuntu 14.04)

# 內容物
 - CURL HTTP/2 伺服器啟用
 - APNS 憑證產生教學
 - PHP simple push code

## Requirements
 - curl >= 7.43.0   | https://curl.haxx.se/docs/http2.html
 - nghttp2
 - openssl >= 1.0.2j | https://www.openssl.org/  | APNS要求~

##### 先看看你的 curl 版本
```sh
> curl --version
curl 7.35.0 (x86_64-pc-linux-gnu) libcurl/7.35.0 OpenSSL/1.0.1f zlib/1.2.8 libidn/1.28 librtmp/2.3
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtmp rtsp smtp smtps telnet tftp
Features: AsynchDNS GSS-Negotiate IDN IPv6 Largefile NTLM NTLM_WB SSL libz TLS-SRP
```
Features 裡面沒有http2 不支援 OpenSSL也過舊一起更新吧QQ

##### 不死心先執行看看
```sh
curl --http2 -I https://google.com.tw/
# Unsupported protocol error (不支援)
```

# CURL HTTP/2 伺服器啟用
Install Git
```sh
sudo apt-get install -y git
```

Install nghttp2
```sh
# Get build requirements
# Some of these are used for the Python bindings
# this package also installs
sudo apt-get install g++ make binutils autoconf automake autotools-dev libtool pkg-config zlib1g-dev libcunit1-dev libssl-dev libxml2-dev libev-dev libevent-dev libjansson-dev libjemalloc-dev cython python3-dev python-setuptools

# Build nghttp2 from source
git clone https://github.com/tatsuhiro-t/nghttp2.git
cd nghttp2
autoreconf -i
automake
autoconf
./configure
make
sudo make install
```

Install or Upgrade to latest curl
```sh
cd /tmp
sudo apt-get build-dep curl
wget http://curl.haxx.se/download/curl-7.52.1.tar.bz2
tar -xvjf curl-7.52.1.tar.bz2
cd curl-7.52.1
./configure --with-nghttp2=/usr/local --with-ssl
make
sudo make install

echo '/usr/local/lib' > /etc/ld.so.conf.d/local.conf    #可能需要sudo su 執行
sudo ldconfig
```

滾完鍵盤測試一下 (成功回傳)
```sh
curl --http2 -I https://google.com.tw
HTTP/2 301
location: https://www.google.com.tw/
content-type: text/html; charset=UTF-8
date: Mon, 09 Jan 2017 08:49:08 GMT
expires: Wed, 08 Feb 2017 08:49:08 GMT
cache-control: public, max-age=2592000
server: gws
content-length: 223
x-xss-protection: 1; mode=block
x-frame-options: SAMEORIGIN
alt-svc: quic=":443"; ma=2592000; v="35,34"
```

Upgrade OpenSSL
```sh
cd /tmp
wget https://openssl.org/source/openssl-1.1.0c.tar.gz
tar -zxvf openssl-1.1.0c.tar.gz
cd openssl-1.1.0c
./config
sudo make
sudo make install
sudo mv /tmp/openssl-1.1.0c /root/
sudo mv /usr/bin/openssl /usr/bin/openssl.old
sudo ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl
echo "/usr/local/ssl/lib" >> /etc/ld.so.conf
ldconfig
```

確認OpenSSL完成更新
```sh
openssl version
#OpenSSL 1.1.0c  10 Nov 2016
```

# APNS 產生憑證
這邊只寫現有ＩＤ產生出憑證，至於要怎麼新建App ID... google it~

請用Mac OS 產生:

1. 登入你的 控制台 [https://developer.apple.com/account/ios/certificate/](https://developer.apple.com/account/ios/certificate/)

2. 選擇 Certificates -> all

3. 按下右上角 + -> 選擇 Production 之下的 Apple Push Notification service SSL (Sandbox & Production) ->下一步

4. 選擇所要產生的App IDs (這也是之後的 apns-topic ) -> 下一步

5. 跟著螢幕指示 -> Launchpad -> 鑰匙圈存取 -> 左上角 "鑰匙圈存取"-> 憑證輔助程式 -> 從憑證授權要求憑證 -> 填完資訊儲存到硬碟 -> 回到 apple 頁面上傳 -> 下一步

6. 下載會獲得一個 "aps.cer" -> 按一下登入到鑰匙圈 -> 登入完成後找到剛剛登入的憑證 名稱 " Apple Push Services:xxxxxxxxxxxxxxxx" -> 右鍵輸出 server_certificates.p12

7. 開啟 Terminal 到檔案位子
```sh
openssl pkcs12 -in server_certificates.p12 -out server_certificates.pem -nodes -clcerts
```

#### 現在就可以使用APNS HTTP/2 推播拉 ~

# APNS with PHP

可以先在 terminal 做推送
```sh
> curl -d '{"aps":{"alert":"[MESSAGE]","sound":"default"}}' --cert "[PATH TO APS CERTIFICATE.pem]":"" -H "apns-topic: [BUNDLE IDENTIFIER]" --http2 https://api.development.push.apple.com/3/device/[TOKEN]
```

會像這樣 發送成功後往下一步前進
```sh
> curl -d '{"aps":{"alert":"Hi!","sound":"default"}}' --cert "server_certificates.pem":"" -H "apns-topic: vocolboy.samplepush" --http2  https://api.development.push.apple.com/3/device/c4942de20b30792b8f1e04c2f5488e05a24bf72f14f1fc849e6581faf957d8e1
```

先產生 phpinfo 檔案確定所需配件都有了
```sh
> php -i > phpinfo.txt
```

比對一下這幾項有符合就可以了~
> 詳情看開頭的 Requirements

```sh
...
curl

cURL support => enabled
cURL Information => 7.52.1
...
HTTP2 => Yes
...
SSL Version => OpenSSL/1.0.2j
...
```

----
# PHP sendHTTP2Push code
```
/**
 * @param $http2ch          the curl connection
 * @param $http2_server     the Apple server url
 * @param $apple_cert       the path to the certificate
 * @param $app_bundle_id    the app bundle id
 * @param $message          the payload to send (JSON)
 * @param $token            the token of the device
 * @return mixed            the status code
 */
function sendHTTP2Push($http2ch, $http2_server, $apple_cert, $app_bundle_id, $message, $token) {

    // url (endpoint)
    $url = "{$http2_server}/3/device/{$token}";

    // certificate
    $cert = realpath($apple_cert);

    // headers
    $headers = array(
        "apns-topic: {$app_bundle_id}",
        "User-Agent: My Sender"
    );

    // other curl options
    curl_setopt_array($http2ch, array(
        CURLOPT_URL => $url,
        CURLOPT_PORT => 443,
        CURLOPT_HTTPHEADER => $headers,
        CURLOPT_POST => TRUE,
        CURLOPT_POSTFIELDS => $message,
        CURLOPT_RETURNTRANSFER => TRUE,
        CURLOPT_TIMEOUT => 30,
        CURLOPT_SSL_VERIFYPEER => false,
        CURLOPT_SSLCERT => $cert,
        CURLOPT_HEADER => 1
    ));

    // go...
    $result = curl_exec($http2ch);
    if ($result === FALSE) {
      throw new Exception("Curl failed: " .  curl_error($http2ch));
    }

    // get response
    $status = curl_getinfo($http2ch, CURLINFO_HTTP_CODE);

    return $status;
}
```

Use
```
// open connection
$http2ch = curl_init();
curl_setopt($http2ch, CURLOPT_HTTP_VERSION, CURL_HTTP_VERSION_2_0);

// send push
$apple_cert = '/certificates/samplepush/development.pem';
$message = '{"aps":{"alert":"Hi!","sound":"default"}}';
$token = 'c4942de20b30792b8f1e04c2f5488e05a24bf72f14f1fc849e6581faf957d8e1';
$http2_server = 'https://api.development.push.apple.com'; // or 'api.push.apple.com' if production
$app_bundle_id = 'it.tabasoft.samplepush';

$status = sendHTTP2Push($http2ch, $http2_server, $apple_cert, $app_bundle_id, $message, $token);
echo "Response from apple -> {$status}\n";

// close connection
curl_close($http2ch);
```

### 打完收工
> 心得 APNS 真的坑
> 重點是這個Server Certificates一年後就失效要renew ....fuck