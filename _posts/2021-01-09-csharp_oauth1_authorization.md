---
layout: post
title:  "C# OAuth 1.0 Authorization 헤더값 생성"
date:   2021-01-09 16:50:00 +0900
categories: windows-dev
tags: dev
---

## C# OAuth 1.0 Authorization 헤더값 생성

### 1. 출처

[OAuth 1.0 Signature 제작 코드 by azure0777](https://m.blog.naver.com/azure0777/221449666569)

### 2. Comment

실 사이트에서 테스트를 못해봐서 제대로 동작하는지는 모르겠고, 대략 이렇게 구현한다는 정도로만 이해하기 위해 가져왔다.

삽질하느라 시간을 허비하지 말고 남이 만들어서 공개한 것을 쓰도록 하자. (남이 만든것을 쓸 떄는 GPL만 조심하자)

### 3. Source

```csharp
using System;
using System.Globalization;
using System.Security.Cryptography;
using System.Text;

namespace YottaCho
{
    public class OAuth1
    {
        /// <summary>
        /// API key (Application access key)
        /// </summary>
        public string ConsumerKey { get; set; }

        /// <summary>
        /// API secret (Application access secret)
        /// </summary>
        public string ConsumerSecret { get; set; }

        /// <summary>
        /// OAuth access token (User session key)
        /// </summary>
        public string AccessToken { get; set; }

        /// <summary>
        /// OAuth access token secret (User session secret)
        /// </summary>
        public string TokenSecret { get; set; }

        /// <summary>
        /// Create OAuth 1.0 Authorization header string
        /// </summary>
        /// <param name="method">HTTP method</param>
        /// <param name="url">Request url</param>
        /// <returns>Authorization header string</returns>
        public string CreateAuthorizationHeader(string method, string url)
        {
            string timestamp = ((long)(DateTime.UtcNow - new DateTime(1970, 1, 1)).TotalSeconds).ToString();
            string nonce = Convert.ToBase64String(new ASCIIEncoding().GetBytes(DateTime.Now.Ticks.ToString(CultureInfo.InvariantCulture)));

            string signature = CreateSignature(method, url, timestamp, nonce);
            string authorizationHeader = CreateAuthorizationValue(timestamp, nonce, signature);

            return authorizationHeader;
        }

        public string CreateAuthorizationValue(string timestamp, string nonce, string signature)
        {
            return $@"OAuth oauth_consumer_key=""{Uri.EscapeDataString(ConsumerKey)}"",oauth_token=""{Uri.EscapeDataString(AccessToken)}"",oauth_signature_method=""HMAC-SHA1""," +
                $@"oauth_timestamp=""{Uri.EscapeDataString(timestamp)}"",oauth_nonce=""{Uri.EscapeDataString(nonce)}"",oauth_version=""1.0"",oauth_signature=""{Uri.EscapeDataString(signature)}""";
        }

        public string CreateSignature(string method, string url, string timestamp, string nonce)
        {
            string source = $@"oauth_consumer_key={ConsumerKey}&oauth_nonce={nonce}&oauth_signature_method=HMAC-SHA1&oauth_timestamp={timestamp}&oauth_token={AccessToken}&oauth_version=1.0";
            string signatureBase = $@"{method}&{Uri.EscapeDataString(url)}&{Uri.EscapeDataString(source)}";
            string signatureKey = $@"{ConsumerSecret}&{TokenSecret}";

            HMACSHA1 hash = new HMACSHA1(new ASCIIEncoding().GetBytes(signatureKey));
            return Convert.ToBase64String(hash.ComputeHash(new ASCIIEncoding().GetBytes(signatureBase)));
        }
    }
}

```

끝.
