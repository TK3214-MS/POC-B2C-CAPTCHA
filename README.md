# Azure AD B2C サインインページへの Google reCAPTCHA 連携
## 概要
このドキュメントでは、以下を構成する事により Azure AD B2C サインインページに Google reCAPTCHA によるサインインチャレンジを実装する手順をステップバイステップで解説します。
- Azure AD B2C カスタムポリシー
- Azure AD B2C カスタムサインインページ用 HTML ソース
- Google reCAPTCHA
- Azure Functions による Google reCAPTCHA へのトークン Validation

Azure AD B2C と連携した Web アプリシナリオのセキュリティ強化にお役立て頂けますと幸いです。

Azure AD B2C カスタムポリシーを利用したGoogle reCAPTCHA の Azure AD B2C サインインへの統合がお試し頂けるデモサイトをご確認頂けます。

[デモライブサイト](https://b2cprod.b2clogin.com/b2cprod.onmicrosoft.com/oauth2/v2.0/authorize?p=B2C_1A_Captcha_signin&client_id=51d907f8-db14-4460-a1fd-27eaeb2a74da&nonce=defaultNonce&redirect_uri=https://jwt.ms&scope=openid&response_type=id_token&prompt=login)

![Architecture](https://github.com/TK3214-MS/POC-B2C-CAPTCHA/assets/89323076/b437c5b4-ab60-458b-a5ab-3cd0805385f2)

1. ユーザーが Web アプリ上で Azure AD B2C サインイン試行を実施
2. Azure AD B2C に認証要求をリダイレクト
3. カスタムポリシーで定義された API エンドポイント (Azure Function) を呼び出し
4. Google reCAPTCHA にユーザー認証チャレンジ情報を送信し、正当性を評価
5. 認証チャレンジが正当なものの場合、サインイン成功応答をユーザーに返却

## 事前準備
### 1. Azure AD B2C テナントの作成
新規 Azure AD B2C テナントを作成します。

[Azure AD B2C テナントを作成する](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/tutorial-create-tenant#create-an-azure-ad-b2c-tenant)

### 2. Google reCAPTCHA 
Google reCAPTCHA にサインアップし、新しいサイトを"チャレンジ(v2)"タイプで登録します。

[Google reCAPTCHA](https://www.google.com/recaptcha/admin/create)

登録ウィザードで以下構成を行います。

- ドメイン : Azure AD B2C ドメイン (xxx.b2clogin.com) を設定して下さい。
- reCAPTCHA ソリューションの入手元を検証する : チェックボックスをオフにする。

### 3. Azure AD B2C カスタムポリシーを利用する為の環境準備

必要なアプリケーション登録、スターターパックのアップロードを行います。

[環境を準備する](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/custom-policy-overview#prepare-your-environment)

### 4. 環境値の設定

適切なファイルに必要な設定値を設定します。

- Sign in Page/customCaptcha.html
```
<div class="g-recaptcha" data-callback="on_captcha_filled" data-sitekey="<Google reCAPTCA で登録したサイトキー値>"></div>
<br><div id="signup">
<a href="https://<Azure AD B2C ドメイン名>.b2clogin.com/<Azure AD B2C ドメイン名>.onmicrosoft.com/oauth2/v2.0/authorize?p=<サインアップ用ポリシー名>&client_id=<Identity Experience Framework アプリケーションのアプリケーション ID>&nonce=defaultNonce&redirect_uri=https%3A%2F%2Fjwt.ms&scope=openid&response_type=id_token&prompt=login">Sign Up</a></div><br>
<div id="forgotpass"> <a href="https://<Azure AD B2C ドメイン名>.b2clogin.com/<Azure AD B2C ドメイン名>.onmicrosoft.com/oauth2/v2.0/authorize?p=<パスワードリセット用ポリシー名>&client_id=<Identity Experience Framework アプリケーションのアプリケーション ID>&nonce=defaultNonce&redirect_uri=https%3A%2F%2Fjwt.ms&scope=openid&response_type=id_token&prompt=login">Forgot my password</a>
</div>
```
- Policy/Captcha_TrustFrameworkExtensions.xml
```
<BasePolicy>
 <TenantId><Azure AD B2C ドメイン名>.onmicrosoft.com</TenantId>
 <PolicyId>B2C_1A_TrustFrameworkExtensions</PolicyId>
</BasePolicy>

---

<ContentDefinition Id="api.signuporsignin">
 <LoadUri>https://<ストレージアカウント名>.blob.core.windows.net/<コンテナ名>/customCaptcha.html</LoadUri>
</ContentDefinition>


<ContentDefinition Id="api.selfasserted">
 <LoadUri>https://<ストレージアカウント名>.blob.core.windows.net/<コンテナ名>/customCaptcha.html</LoadUri>
</ContentDefinition>

---

<Metadata>
 <Item Key="ServiceUrl">https://<Azure Function アプリ名>.azurewebsites.net/api/identity</Item>
 <Item Key="AuthenticationType">None</Item>
 <Item Key="SendClaimsIn">Body</Item>
 <!-- JSON, Form, Header, and Query String formats supported -->
 <Item Key="ClaimsFormat">Body</Item>
 <!-- Defines format to expect claims returning to B2C -->
 <!-- REMOVE the following line in production environments -->
 <Item Key="AllowInsecureAuthInProduction">true</Item>
</Metadata>
```

- Policy/Signin.xml
```
<TrustFrameworkPolicy
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:xsd="http://www.w3.org/2001/XMLSchema"
  xmlns="http://schemas.microsoft.com/online/cpim/schemas/2013/06"
  PolicySchemaVersion="0.3.0.0"
  TenantId="<Azure AD B2C ドメイン名>.onmicrosoft.com"
  PolicyId="B2C_1A_Captcha_signin"
  PublicPolicyUri="http://6157010b2cpoc.onmicrosoft.com/B2C_1A_Captcha_signin">
```

- Function API/local.settings.json
```
{
  "IsEncrypted": false,
  "Values": {
    "B2C_Recaptcha_Secret": "<Google reCAPTCHA シークレット>"
  }
}
```

## カスタムポリシー展開
編集したカスタムポリシーを Azure AD B2C にアップロードします。

- Signin.xml
- Captcha_TrustFrameworkExtensions.xml

[ポリシーのアップロード](https://learn.microsoft.com/en-us/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-custom-policy#upload-the-policies)

## Azure Function アプリの展開
以下手順に従って Azure Function アプリの作成、ソースコードのアップロードを行います。

[Visual Studio Code を仕様して Azure Functions を開発する](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-develop-vs-code?tabs=csharp)

## カスタムページ要素のホスティング
Azure Storage Account を作成し、Blob 内でコンテナを作成しHTMLファイルをアップロードします。

この際、Sign in Page/customCaptcha.html で設定したストレージアカウント名と共通名を設定して下さい。

[ページ コンテンツのホスト](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/customize-ui-with-html?pivots=b2c-custom-policy#hosting-the-page-content)