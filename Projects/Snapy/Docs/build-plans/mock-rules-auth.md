# Auth Service — Beeceptor Mock Rules

**Mock Server URL:** `https://auth-snapy.free.beeceptor.com`

**Free plan limit:** 3 custom rules. `POST /otp/request` is skipped — mock it client-side.

---

## Rule 1: POST /otp/verify

**Path:** `/otp/verify`
**Method:** POST
**Status Code:** 200

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhMWIyYzMtZDRlNS1mNmE3LWI4YzktZDBlMTIzNDU2NzgiLCJncmFkZV9pZCI6ImMxZDJlM2YtNGI1Ni03ODkwLTFhYmMtZGVmMTIzNDU2NzgiLCJyb2xlIjoidXNlciIsImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAwOTAwfQ.fake-signature-for-mock",
  "refreshToken": "mock-refresh-token-abc123xyz",
  "isNewUser": true,
  "user": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "phone": "+94771234567",
    "name": null,
    "gradeId": null,
    "stream": null,
    "role": "user",
    "createdAt": "2026-05-16T10:00:00Z"
  }
}
```

### Notes
- Toggle `isNewUser` between `true` and `false` in Beeceptor to test:
  - `true` → new user → redirects to onboarding (name → grade → stream)
  - `false` → returning user → redirects to home screen
- The `accessToken` is a fake JWT. It won't pass real signature verification but works for mock mode.
- Request body the frontend sends:
  ```json
  {
    "phone": "+94771234567",
    "otp": "123456"
  }
  ```
- Any OTP value should work in mock mode (Beeceptor doesn't validate input).

---

## Rule 2: GET /me

**Path:** `/me`
**Method:** GET
**Status Code:** 200

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "phone": "+94771234567",
  "name": "Kasun Perera",
  "gradeId": "c1d2e3f4-5b67-8901-abcd-ef1234567890",
  "stream": null,
  "role": "user",
  "createdAt": "2026-05-16T10:00:00Z"
}
```

### Notes
- Called on app launch when a refresh token exists (auto-login flow).
- Also called by the Profile screen to display user info.
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- For testing a user who hasn't completed onboarding, change to:
  ```json
  {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "phone": "+94771234567",
    "name": null,
    "gradeId": null,
    "stream": null,
    "role": "user",
    "createdAt": "2026-05-16T10:00:00Z"
  }
  ```

---

## Rule 3: PATCH /me

**Path:** `/me`
**Method:** PATCH
**Status Code:** 200

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "phone": "+94771234567",
  "name": "Kasun Perera",
  "gradeId": "c1d2e3f4-5b67-8901-abcd-ef1234567890",
  "stream": "science",
  "role": "user",
  "createdAt": "2026-05-16T10:00:00Z"
}
```

### Notes
- Called during onboarding for each step:
  - Step 1: `PATCH /me { "name": "Kasun Perera" }` → sets name
  - Step 2: `PATCH /me { "gradeId": "c1d2e3f4-..." }` → sets grade
  - Step 3: `PATCH /me { "stream": "science" }` → sets stream (A/L only)
- The mock just echoes back the "updated" profile regardless of what fields are sent.
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- Example request body:
  ```json
  {
    "name": "Kasun Perera"
  }
  ```

---

## Skipped: POST /otp/request

This endpoint only returns `{ "success": true }`. Mock it client-side instead of using a Beeceptor rule.

### iOS (Swift)
```swift
func requestOTP(phone: String) async throws -> Bool {
    #if MOCK
    try await Task.sleep(for: .seconds(1)) // simulate network delay
    return true
    #else
    let response: OTPResponse = try await apiClient.request(.requestOTP(phone: phone))
    return response.success
    #endif
}
```

### Android (Kotlin)
```kotlin
suspend fun requestOtp(phone: String): Boolean {
    if (BuildConfig.MOCK) {
        delay(1000) // simulate network delay
        return true
    }
    return apiClient.requestOtp(phone).success
}
```

---

## Testing the Full Auth Flow

```
┌──────────────────────────────────────────────────────────┐
│                    FRONTEND AUTH FLOW                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Phone Input Screen                                   │
│     → User enters "+94771234567"                         │
│     → No API call (mocked client-side)                   │
│     → Navigate to OTP screen                             │
│                                                          │
│  2. OTP Verify Screen                                    │
│     → User enters "123456" (any code works)              │
│     → POST /otp/verify  ←── Rule 1                      │
│     → Save accessToken + refreshToken locally            │
│     → Check isNewUser:                                   │
│       ├─ true  → Navigate to Name Input (onboarding)     │
│       └─ false → Navigate to Home Screen                 │
│                                                          │
│  3. Name Input Screen (if new user)                      │
│     → User enters "Kasun Perera"                         │
│     → PATCH /me { "name": "Kasun Perera" }  ←── Rule 3  │
│     → Navigate to Grade Select                           │
│                                                          │
│  4. Grade Select Screen                                  │
│     → User selects "Grade 11"                            │
│     → PATCH /me { "gradeId": "c1d2e3f4-..." }  ←─ Rule 3│
│     → If O/L (10-11): Navigate to Home Screen            │
│     → If A/L (12-13): Navigate to Stream Select          │
│                                                          │
│  5. Stream Select Screen (A/L only)                      │
│     → User selects "Science"                             │
│     → PATCH /me { "stream": "science" }  ←── Rule 3     │
│     → Navigate to Home Screen                            │
│                                                          │
│  6. Next App Launch (auto-login)                         │
│     → App finds saved refreshToken                       │
│     → GET /me  ←── Rule 2                               │
│     → Navigate directly to Home Screen                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Mock UUIDs Reference

Use these consistent IDs across all mock responses:

| Entity | UUID | Description |
|--------|------|-------------|
| User | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` | Test student |
| Grade 10 | `f1e2d3c4-b5a6-7890-1234-567890abcdef` | O/L Grade 10 |
| Grade 11 | `c1d2e3f4-5b67-8901-abcd-ef1234567890` | O/L Grade 11 |
| Grade 12 | `b2c3d4e5-6f78-9012-abcd-ef1234567890` | A/L Grade 12 |
| Grade 13 | `d3e4f5a6-7b89-0123-abcd-ef1234567890` | A/L Grade 13 |
