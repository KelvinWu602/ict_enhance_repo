# 非對稱加密原理

在課堂上，我們學習了非對稱加密的基本原理，而實際上我們如何運用這些加密算法呢？這個教程將會一步步引領你完成加密、解密、簽名及認證四個步驟。

## 網絡安全的基石：OpenSSL

其實加密算法很多時都是由數學家提出的，因此開發加密算法程式的門檻極高，一般程序員無法獨力開發出足夠安全的程序，因此通常都會使用其他專業密碼學家及大神程序員所分享的Library。而在加密方面，最著名的函式庫便是OpenSSL.

> https://zh.wikipedia.org/zh-tw/OpenSSL

### Mac 安裝方法

1. 按下command + space，輸入terminal，打開終端機。
2. 輸入以下指令即可：
```
brew install openssl
openssl --version
```
3. 若可見版本訊息，即安裝成功。

### Windows 安裝方法

1. 在開始清單搜尋Powershell，用系統管理員身份打開。
2. 輸入以下指令安裝Linux:
```
wsl --install
```
3. 等安裝結束後，重新開機。
4. 再次打開Powersheel，輸入：
```
wsl
```
4. 設定好用戶名稱和密碼後，立即安裝OpenSSl：
```
sudo apt update
sudo apt install openssl libssl-dev
openssl version
```
5. 如果顯示到版本訊息，即已成功安裝。

## 運用OpenSSL建立一對公鑰及私鑰

### 建立公私鑰對
在剛才的命令行中輸入：
```
openssl genrsa -out key_pair.pem 2048
```
- `genrsa`:代表我們將運用RSA加密算法，除了RSA外，還有很多種加密算法，在此不贅述。
- `-out key_pair.pem`:用以指定輸出公私鑰對檔案的名稱。
- `2048`:用以指定鑰匙佔多少內存，此處指定2048位元。

完成後，你將會得到一個新檔案`key_pair.pem`，內容如下：
```
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDgrin9BttGOuXb
FvnPenqc77Qlzf1VQt4l0GLm5IGViLrINkWdxVjKfVTC0xO6KAFP35aYQIh+tNxs
vdjdguoPEny/uhQ/ZVWVJz6qccyClmp47OZ+cYX/nnNpUz/y45Ebz0efCldH26qn
pvH8UcQmjBb7YgbfkHCUzk7k1kEC0ztgNANkPgHp8mfnAop9M0jDY3lzcWPhAIwR
CtKKD4wmsptuarDRwfyk0W7leW4iE7l9hB17cPJwaNtmz3am5eKqFevwLs/8yMRU
8AQRdlvqvFVYBsHUoO19CbvEkrlKyFcm7MGGCqj58QHS7G+8PiuaCTXdIoviCE5/
WHTA3ZbNAgMBAAECggEACeZqt7k+XrsYJz39LnD7Z6BhS/wmYkQixUBIJ1vgwN3I
Oyu3CBNqzOtWWYpS93QtUJ9tw5IuYYTnJIx9+o668QSTxh/1JfD5YyvaRUjj5coq
cV8g7efjK2cIN1chfXqdCYBp675ZJ7Oscpw20/MnZTpthdClNGMpLsljbQ5qyWy/
qbNuUyGK0opuQguZErcAhvNFR820Osff0KQqKEvqX4dhdT5UMF40cVtG7sMD8oM3
4Ys3M5Ow+iDAJumSgA+9x6C04j/u5d5PUmhTH3DRwlsRJycwlio/XGuOeUY6fgF/
xOH//w0pBP4LxGozAnX6iAPhhb3zN+1WbXpAYm+S4QKBgQD9IzPnPZp+n6QCUa/9
hR59p6SFlEanX2ezIb8USKDKerQ1U7F5hejy54+j4nQaSZH9e1fY5sO7RpLGgeb/
yflqi9KeYrSBeKEJV5f2pZo12JJSA/H78t381S9pFZBTwFKag5PbzvzWKW/DO3UW
KwVkep1iIk7Q2ndN8U5jVHwhYQKBgQDjOJTt77tMJHPH4GCzudHioIo1FvOXyMB8
3rhV5TrWMwevpqef+ZFK2PaA1Ic/itWbaABdPnOWhdt256LnWTCnx4UO7NC0nVuY
nkw8sVhDfMJm87AW2suDwYvr0eXC2MLWmdZ7MnarG5X84hGE/eLPUs1FyiqAJ4rD
49vwEwCw7QKBgFyxTUo5tp7zWh03SFhvLHEauBXp681SFCj2DIAi8C30rJRyZyR2
soxv2ptKSvVtRzYoukxEhBvJhemGm83CacBoHuG8hxh50Y4YMx8wGL3q5fl+VFfL
4Rm5/rheGxFv9U97KuNscg0B81jsJr3NVxYqCANtSKsVtGYoHGom/6VBAoGBAKf3
JbCV7MCmmagBZ7qz/EEpJ8GDC+MCFbi4808butiotF/WNEd/tzW7GM23TZtdR/Yv
dUV4av20Sb2mEbgvKFZ+mQ+lY8qAIDu7mOOsvXB2A0cTkPH0H0lwg7x5Vv0oOy9k
XTaI4UwvgjqD6yuCem2D6hZTEgPWNzADeowHoBUpAoGABCiYIbiIZfUIwArB2qQI
FLI3TFZOW7AxqGzHnQkZHXPCGDHudbMXrjpI+Dl3mymkJdhfM0rXsVp/r8YwwILy
z9ZFGuylwR3uJpIxuP/o9E8Zy2cA3/3NfWhQNXgi7MmyhyacfWPAx8aPH7MQrDcK
+vJ6/tEZYZtj5fv8ZtLqyYk=
-----END PRIVATE KEY-----
```

這些奇怪的文字包含了以下資訊：
- 私鑰 private key
- 公鑰 public key 
- 加密算法 encryption algorithm
- 少量其他數據，不影響理解

既然公鑰和私鑰放在一起，我們希望可以將公鑰提取出來，放在另一個檔案，方便派發給他人，這是可運行：
```
openssl rsa -in dora_private.pem -pubout -out dora_public.pem
```
- `rsa`:告訴openssl這個檔案是用了RSA
- `-in key_pair.pem`:指定輸入檔案為剛才的檔案
- `-pubout`:指定要提取公鑰, output public key
- `-out public_key.pem`:指定輸出檔案的名稱

完成後，你會獲得`public_key.pem`這個檔案：
```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4K4p/QbbRjrl2xb5z3p6
nO+0Jc39VULeJdBi5uSBlYi6yDZFncVYyn1UwtMTuigBT9+WmECIfrTcbL3Y3YLq
DxJ8v7oUP2VVlSc+qnHMgpZqeOzmfnGF/55zaVM/8uORG89HnwpXR9uqp6bx/FHE
JowW+2IG35BwlM5O5NZBAtM7YDQDZD4B6fJn5wKKfTNIw2N5c3Fj4QCMEQrSig+M
JrKbbmqw0cH8pNFu5XluIhO5fYQde3DycGjbZs92puXiqhXr8C7P/MjEVPAEEXZb
6rxVWAbB1KDtfQm7xJK5SshXJuzBhgqo+fEB0uxvvD4rmgk13SKL4ghOf1h0wN2W
zQIDAQAB
-----END PUBLIC KEY-----
```

這個檔案便是你的*公鑰*，可以隨時分發給其他人。

## 加密及解密

### 準備明文訊息

要嘗試加密解密，先準備要被加密的檔案。在指令行中，你可以輸入：
```
echo 'I like ICT!' > my_message.txt
```

然後你就會得到一個`my_message.txt`檔案，要查看檔案內容，你可以：
```
cat my_message.txt
```

### 用公鑰加密

你可以輸入以下指令：
```
openssl rsautl -encrypt -inkey public_key.pem -pubin -in my_message.txt -out my_message.encrypted
```
- `rsautl`:指定正在用RSA算法
- `-encrypt`:指定正在加密
- `-inkey public_key.pem`:指定輸入檔
- `-pubin`:指定輸入檔類型為公鑰public key
- `-in my_message.txt`:指定待加密檔案
- `-out my_message.encrypted`:指定輸出檔案名稱

這會建立一個新檔案`my_message.encrypted`，內容是加密後的my_message.txt。
```
ߎDd��ЬP,�1��P�|5��o�ט8Q��r�OQ���?��I*�2����-���ފ�Y�L
���J�wy�{�;�K8�"�c͒��;+Z�b�^&������o3�n�dP���A���b�IJ�X�|d�������@��?*���˶�� ��ե�c('4.����`;F��S,�?�wt�;�f�6cd�}�%$!t6"l{g�v��nЂ���0���q�
=��ʴ�2���Ao��M�\Q./xcϤu���
```
由於加密後的內容只是一串數字，因此無法被編碼成可閱讀的字元。

### 用私鑰解密

```
openssl rsautl -decrypt -inkey key_pair.pem -in my_message.encrypted
```
唯一的不同是指定當前操作為解密，運行後便可以看到原文的內容。

### 試試看

1. 如果用同一條公鑰解密，會成功嗎？
2. 如果生成另一條私鑰，並用其解密，會發生什麼事？

## 數位簽章及認證

### 用私鑰簽署

```
openssl dgst -sha256 -sign dora_private.pem -out message.dora.signature message.txt
```
- `dgst`:代表digital signature 數位簽章。
- `-sha256`:一種雜湊算法，將被簽署的內容變成固定長度的字串，加快並化簡加密的過程，要了解更多可以搜尋Hash function。
- `-sign key_pair.pem`:指定私鑰
- `-out my_message.sign`:指定輸出檔案名稱
- `my_message.txt`:指定被簽署的檔案

完成後，你會得到`my_message.sign`檔案，這就是你的簽名！

所謂的數位簽名，也是一個隨機數字，因此無法解碼成可閱文字：
```
W�N%���
       ��� �='���V�\3#
                      �,�M����15��L���K��>�2S�L:�?%%�j�d�mE��ŅJk�����TX�fC3�v�+uN�s���f�M�)y�
k�m��Z�+��Ǡ��O�A�eb�QA�ء��a�^��-�t��ɓ�A �7�
```

### 用公鑰驗證簽名真實性

```
openssl dgst -sha256 -verify dora_public.pem -signature message.dora.signature message.txt
```
- `dgst`:代表digital signature 數位簽章
- `-sha256`:一種雜湊算法
- `-verify public_key.pem`:指定公鑰
- `-signature my_message.sign`:指定要驗證的簽名檔案
- `my_message.txt`:指定被簽署的檔案

如無意外的話，你會見到`Verified OK`訊息，代表驗證成功！

### 試試看

1. 如果你生成另一條公鑰並驗證這個簽名，還會成功嗎？
