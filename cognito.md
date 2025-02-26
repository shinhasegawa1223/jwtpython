（由于技术原因，联网搜索暂不可用）

Next.js（NextAuth.js）、Django（Django Ninja）、およびAmazon Cognitoを使用して、ログイン機能付きの商品一覧を取得するAPIを構築する計画ですね。以下に、そのアーキテクチャと実装の概要を説明します。

### アーキテクチャ概要

1. **フロントエンド (Next.js)**
   - NextAuth.jsを使用して認証を管理します。
   - Amazon Cognitoを認証プロバイダーとして使用します。
   - ログイン後に、DjangoバックエンドのAPIを呼び出して商品一覧を取得します。

2. **バックエンド (Django Ninja)**
   - Django Ninjaを使用してREST APIを構築します。
   - `/items` エンドポイントで商品一覧を返すAPIを提供します。
   - 認証はAmazon Cognitoのトークンを使用して行います。

3. **認証 (Amazon Cognito)**
   - ユーザー認証とトークン発行を行います。
   - Next.jsとDjangoの両方でCognitoのトークンを検証します。

### 実装ステップ

#### 1. Amazon Cognitoの設定
- AWSマネジメントコンソールでCognitoユーザープールを作成します。
- アプリクライアントを作成し、必要な設定を行います。
- 必要な環境変数を取得します（ユーザープールID、クライアントIDなど）。

#### 2. Next.js (NextAuth.js) の設定

- NextAuth.jsをインストールします。
```bash
npm install next-auth
```

- `pages/api/auth/[...nextauth].js` を作成し、Cognitoプロバイダーを設定します。
```javascript
import NextAuth from "next-auth";
import Providers from "next-auth/providers";

export default NextAuth({
  providers: [
    Providers.Cognito({
      clientId: process.env.COGNITO_CLIENT_ID,
      clientSecret: process.env.COGNITO_CLIENT_SECRET,
      domain: process.env.COGNITO_DOMAIN,
    }),
  ],
  callbacks: {
    async jwt(token, user) {
      if (user) {
        token.id = user.id;
      }
      return token;
    },
    async session(session, token) {
      session.user.id = token.id;
      return session;
    },
  },
});
```

- 環境変数を設定します。
```bash
COGNITO_CLIENT_ID=your-cognito-client-id
COGNITO_CLIENT_SECRET=your-cognito-client-secret
COGNITO_DOMAIN=your-cognito-domain
```

#### 3. Django Ninja の設定

- Django Ninjaをインストールします。
```bash
pip install django-ninja
```

- `items` アプリを作成し、`/items` エンドポイントを実装します。
```python
from ninja import NinjaAPI
from ninja.security import HttpBearer
from django.http import JsonResponse

api = NinjaAPI()

class AuthBearer(HttpBearer):
    def authenticate(self, request, token):
        # Cognitoのトークンを検証するロジックを実装
        if token == "valid-token":
            return token

@api.get("/items", auth=AuthBearer())
def get_items(request):
    items = [
        {"id": 1, "name": "Item 1"},
        {"id": 2, "name": "Item 2"},
    ]
    return items
```

- `urls.py` にエンドポイントを追加します。
```python
from django.urls import path
from .api import api

urlpatterns = [
    path("api/", api.urls),
]
```

#### 4. Next.jsからDjango APIを呼び出す

- ログイン後にDjangoのAPIを呼び出すための関数を実装します。
```javascript
import { getSession } from 'next-auth/react';

export async function getItems() {
  const session = await getSession();
  const response = await fetch('http://localhost:8000/api/items', {
    headers: {
      'Authorization': `Bearer ${session.accessToken}`,
    },
  });
  return response.json();
}
```

- ページコンポーネントで商品一覧を取得します。
```javascript
import { useEffect, useState } from 'react';
import { getItems } from '../lib/api';

export default function Items() {
  const [items, setItems] = useState([]);

  useEffect(() => {
    getItems().then(data => setItems(data));
  }, []);

  return (
    <div>
      <h1>Items</h1>
      <ul>
        {items.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### まとめ
このアーキテクチャでは、Next.jsがフロントエンドと認証を担当し、DjangoがバックエンドのAPIを提供します。Amazon Cognitoを使用して認証を行い、
セキュアなAPI通信を実現します。各コンポーネントの設定と連携を適切に行うことで、ログイン機能付きの商品一覧取得システムを構築できます。




（由于技术原因，联网搜索暂不可用）

いいえ、`if token == "valid-token":` という記載は、実際のプロダクション環境では適切ではありません。これは単なる例示であり、実際の認証ロジックでは **Amazon Cognito が発行したトークンを検証する必要があります**。Cognito のトークンは JWT (JSON Web Token) 形式であり、以下のようなステップで検証する必要があります。

---

### 1. **Cognito トークンの検証手順**
Cognito が発行するトWT は以下の特徴を持っています：
- **署名付き**：Cognito の公開鍵で署名されているため、署名を検証することでトークンの改ざんを防ぎます。
- **有効期限**：トークンには有効期限 (`exp`) が含まれており、期限切れのトークンは無効です。
- **クレーム**：トークンにはユーザー情報やスコープなどのクレームが含まれています。

トークンを検証するためには、以下の手順を踏む必要があります：
1. **トークンの署名を検証する**：Cognito の公開鍵を使用して、トークンの署名が正しいことを確認します。
2. **トークンの有効期限を確認する**：トークンの `exp` クレームを確認し、有効期限内であることを確認します。
3. **トークンの発行者を確認する**：トークンの `iss` クレームが正しい Cognito ユーザープールの URL であることを確認します。

---

### 2. **Django での Cognito トークン検証実装**
Django で Cognito トークンを検証するためには、以下のライブラリや方法を使用できます。

#### 必要なライブラリのインストール
```bash
pip install python-jose[cryptography] requests
```

- `python-jose`：JWT のデコードと検証を行うためのライブラリ。
- `requests`：Cognito の公開鍵を取得するために使用。

#### Cognito トークンを検証するコード例
以下は、Django Ninja の `AuthBearer` クラスを拡張して、Cognito トークンを検証する例です。

```python
from ninja.security import HttpBearer
from jose import jwt, JWTError
import requests
from datetime import datetime

class AuthBearer(HttpBearer):
    def authenticate(self, request, token):
        try:
            # Cognito ユーザープールの情報
            region = 'your-cognito-region'  # Cognito ユーザープールのリージョン
            user_pool_id = 'your-user-pool-id'  # Cognito ユーザープール ID
            app_client_id = 'your-app-client-id'  # Cognito アプリクライアント ID

            # Cognito の公開鍵を取得
            jwks_url = f'https://cognito-idp.{region}.amazonaws.com/{user_pool_id}/.well-known/jwks.json'
            jwks = requests.get(jwks_url).json()

            # トークンのヘッダーからキー ID (kid) を取得
            header = jwt.get_unverified_header(token)
            kid = header['kid']

            # 公開鍵を取得
            key = next((key for key in jwks['keys'] if key['kid'] == kid), None)
            if not key:
                raise JWTError('Invalid key')

            # トークンを検証
            claims = jwt.decode(
                token,
                key,
                algorithms=['RS256'],
                audience=app_client_id,
                issuer=f'https://cognito-idp.{region}.amazonaws.com/{user_pool_id}'
            )

            # 有効期限を確認
            if datetime.utcnow() > datetime.fromtimestamp(claims['exp']):
                raise JWTError('Token expired')

            # トークンが有効であれば、クレームを返す
            return claims

        except JWTError as e:
            # トークンが無効な場合
            print(f"Token validation failed: {e}")
            return None
```

---

### 3. **Django Ninja での使用例**
上記の `AuthBearer` クラスを使用して、`/items` エンドポイントを保護します。

```python
from ninja import NinjaAPI
from .auth import AuthBearer

api = NinjaAPI()

@api.get("/items", auth=AuthBearer())
def get_items(request):
    items = [
        {"id": 1, "name": "Item 1"},
        {"id": 2, "name": "Item 2"},
    ]
    return items
```

---

### 4. **Next.js からのトークン送信**
Next.js 側では、NextAuth.js で取得した Cognito トークンを Django の API に送信します。

```javascript
import { getSession } from 'next-auth/react';

export async function getItems() {
  const session = await getSession();
  const response = await fetch('http://localhost:8000/api/items', {
    headers: {
      'Authorization': `Bearer ${session.accessToken}`,
    },
  });
  return response.json();
}
```

---

### 5. **環境変数の設定**
Cognito の設定を環境変数として管理します。

```bash
# Django の環境変数
COGNITO_REGION=your-cognito-region
COGNITO_USER_POOL_ID=your-user-pool-id
COGNITO_APP_CLIENT_ID=your-app-client-id
```

---

### まとめ
- `if token == "valid-token":` は単なる例示であり、実際のプロダクション環境では使用しないでください。
- Cognito トークンは JWT 形式であり、署名、有効期限、発行者を検証する必要があります。
- Django で `python-jose` ライブラリを使用して、Cognito トークンを検証するロジックを実装します。
- Next.js から Django の API にトークンを送信し、認証を実現します。

これで、セキュアな認証付き API を構築できます！







