（由于技术原因，联网搜索暂不可用）

NextAuth.jsの **Credentials Provider** を使用してカスタムページでログインしようとした際に `401 Unauthorized` エラーが発生する場合、認証リクエストが失敗していることを示しています。このエラーは、以下のような原因が考えられます:

1. クレデンシャル（ユーザー名やパスワード）が正しくない。
2. APIエンドポイントが正しく設定されていない。
3. クレデンシャルの処理方法に問題がある。

以下に、この問題をデバッグして解決するための具体的な手順を説明します。

---

### 1. **Credentials Provider の設定を確認する**
NextAuth.jsの設定で、`authorize` 関数が正しく実装されているか確認します。この関数は、クレデンシャルを検証し、有効な場合にユーザーオブジェクトを返す役割を担います。

例:
```javascript
import NextAuth from "next-auth";
import Providers from "next-auth/providers";

export default NextAuth({
  providers: [
    Providers.Credentials({
      name: "Credentials",
      credentials: {
        username: { label: "Username", type: "text" },
        password: { label: "Password", type: "password" },
      },
      authorize: async (credentials) => {
        // APIを呼び出してクレデンシャルを検証
        const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/auth/login`, {
          method: "POST",
          body: JSON.stringify(credentials),
          headers: { "Content-Type": "application/json" },
        });

        const user = await res.json();

        if (res.ok && user) {
          return user; // 認証が成功した場合はユーザーオブジェクトを返す
        } else {
          return null; // 認証が失敗した場合はnullを返す
        }
      },
    }),
  ],
  // その他のNextAuth.js設定...
});
```

**よくある問題:**
- APIエンドポイント (`/auth/login`) が正しくない、または到達できない。
- APIのレスポンスが有効なユーザーオブジェクトを返していない。
- `authorize` 関数がエラーを適切に処理していない。

---

### 2. **APIエンドポイントを確認する**
`authorize` 関数で呼び出しているAPIエンドポイントが正しく動作しているか確認します。**Postman** や **cURL** を使用して、以下のことを確認してください:
- 正しいクレデンシャル（例: `username` と `password`）を受け取る。
- 有効なユーザーオブジェクトまたは適切なエラーレスポンスを返す。

例:
```bash
curl -X POST https://your-api-url/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpassword"}'
```

**よくある問題:**
- APIエンドポイントがリクエストを正しく処理していない。
- APIが無効なクレデンシャルや認証ヘッダーの欠落により `401 Unauthorized` を返す。

---

### 3. **リクエストのクレデンシャルを確認する**
ログインフォームから `authorize` 関数に正しくクレデンシャルが送信されているか確認します。

例:
```javascript
import { signIn } from "next-auth/react";

const handleLogin = async (e) => {
  e.preventDefault();
  const result = await signIn("credentials", {
    redirect: false,
    username: e.target.username.value,
    password: e.target.password.value,
  });

  if (result.error) {
    console.error("ログイン失敗:", result.error);
  } else {
    console.log("ログイン成功!");
  }
};
```

**よくある問題:**
- フォームのフィールド (`username` と `password`) が `credentials` オブジェクトのキーと一致していない。
- クレデンシャルが `signIn` 関数に正しく渡されていない。

---

### 4. **`authorize` 関数をデバッグする**
`authorize` 関数にログを追加して、リクエストとレスポンスを確認します。

例:
```javascript
authorize: async (credentials) => {
  console.log("受け取ったクレデンシャル:", credentials);

  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/auth/login`, {
    method: "POST",
    body: JSON.stringify(credentials),
    headers: { "Content-Type": "application/json" },
  });

  console.log("APIレスポンスステータス:", res.status);
  const user = await res.json();
  console.log("APIレスポンスデータ:", user);

  if (res.ok && user) {
    return user;
  } else {
    return null;
  }
},
```

**よくある問題:**
- APIのレスポンスステータスが `200 OK` ではない。
- APIのレスポンスに有効なユーザーオブジェクトが含まれていない。

---

### 5. **環境変数を確認する**
NextAuth.jsの設定で使用している環境変数（例: `NEXT_PUBLIC_API_URL`）が、デプロイ環境（例: AWS）で正しく設定されているか確認します。

例 `.env` ファイル:
```bash
NEXT_PUBLIC_API_URL=https://your-api-url
```

**よくある問題:**
- デプロイ環境で環境変数が設定されていない、または間違っている。
- APIのURLがデプロイされたNext.jsアプリからアクセスできない。

---

### 6. **セッションとCookieの設定を確認する**
認証が成功してもセッションが維持されない場合、NextAuth.jsのセッションとCookieの設定を確認します。

例:
```javascript
export default NextAuth({
  providers: [
    // Credentials provider...
  ],
  session: {
    strategy: "jwt", // セッション管理にJWTを使用
  },
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

**よくある問題:**
- セッションストラテジーが正しく設定されていない。
- `jwt` または `session` コールバックがユーザーデータを正しく処理していない。

---

### 7. **CORS設定を確認する**
APIが別のドメインでホストされている場合、APIサーバーがNext.jsアプリからのクロスオリジンリクエストを許可しているか確認します。

例 Django CORS設定:
```python
# settings.py
CORS_ALLOWED_ORIGINS = [
    "https://your-nextjs-app-domain.com",
    "http://localhost:3000",
]
```

**よくある問題:**
- APIサーバーがCORS制限によりNext.jsアプリからのリクエストをブロックしている。

---

### 8. **ローカル環境とAWS環境を比較する**
AWSにデプロイした後に問題が発生している場合、ローカル環境とAWS環境の違いを確認します。以下の点をチェックしてください:
- 環境変数。
- ネットワーク設定（セキュリティグループ、VPC設定など）。
- デプロイされたNext.jsアプリからAPIにアクセスできるか。

---

### デバッグのまとめ
1. NextAuth.jsの `authorize` 関数を確認する。
2. APIエンドポイントを独立してテストする。
3. ログインフォームからクレデンシャルが正しく渡されているか確認する。
4. `authorize` 関数にログを追加してデバッグする。
5. デプロイ環境の環境変数を確認する。
6. セッションとCookieの設定を確認する。
7. CORS設定が正しく行われているか確認する。
8. ローカル環境とAWS環境の違いを比較する。

これらの手順を踏むことで、`401 Unauthorized` エラーの原因を特定し、解決できるはずです。もし問題が解決しない場合は、APIのレスポンスやNextAuth.jsの設定の詳細を共有していただければ、さらに具体的なアドバイスを提供できます。
