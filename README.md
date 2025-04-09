# Ruttala-lava-raju.github.io

# 📘 Okta IDX Authentication Integration

## Overview

This module integrates **Okta Identity Engine (IDX)** into a React application using `@okta/okta-auth-js` and `@okta/okta-react`. It supports:

- **Username/password login**
- **Email link (Magic Link) login**
- **Session rehydration with token renewal**
- **Token management and navigation post-authentication**

---

## 🔧 Environment Setup

Ensure the following packages are installed:

```bash
npm install @okta/okta-auth-js @okta/okta-react
```

### Environment Variables (example)

```env
REACT_APP_OKTA_CLIENT_ID=your-client-id
REACT_APP_OKTA_ISSUER=https://your-okta-domain/oauth2/default
REACT_APP_OKTA_REDIRECT_URI=http://localhost:3000/login/callback
```

---

## 🚀 Authentication Flow

### 1. **Session Validation on Mount**

The component checks if a valid session exists, retrieves tokens silently, and navigates accordingly.

```tsx
useEffect(() => {
  oktaAuth.session.exists().then((response) => {
    if (response) {
      setSessionLoading(true);
      oktaAuth.token
        .getWithoutPrompt()
        .then(async (response) => {
          oktaAuth.tokenManager.setTokens(response.tokens);
          await navigateWithAccessToken(response.tokens.accessToken?.accessToken, response.tokens.refreshToken?.refreshToken);
        })
        .finally(() => setSessionLoading(false));
    }
  });
}, []);
```

---

### 2. **Username/Password Authentication Flow**

This flow leverages Okta IDX to perform traditional email/password login:

```tsx
await oktaAuth.idx.startTransaction({ flow: 'authenticate' });
const { status, tokens, messages } = await oktaAuth.idx.proceed({
  username: formData.email,
  password: formData.password,
  authenticator: AuthenticatorKey.OKTA_PASSWORD,
});
if (status === IdxStatus.SUCCESS && tokens) {
  oktaAuth.tokenManager.setTokens(tokens);
  await navigateWithAccessToken(tokens.accessToken?.accessToken, tokens.refreshToken?.refreshToken);
}
```

---

### 3. **Magic Link (Email) Login Flow**

This advanced flow sends an email with a verification link and continuously polls for completion.

```tsx
const idxTransaction = await oktaAuth.idx.authenticate({
  username: email,
  authenticator: AuthenticatorKey.OKTA_EMAIL,
  methodType: 'email',
});

const pollUrl = nextStep?.authenticator?.poll?.href;
while (!isLogin) {
  const pollResponse = await fetch(pollUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ stateHandle }),
  });
  const pollData = await pollResponse.json();
  if (pollData.successWithInteractionCode) {
    const { interactionCode } = pollData.successWithInteractionCode.value.find(item => item.name === 'interaction_code');
    const { clientId, codeVerifier, redirectUri, scopes } = await oktaAuth.idx.getTransactionMeta();
    const tokenResponse = await oktaAuth.token.exchangeCodeForTokens({ interactionCode, clientId, codeVerifier, redirectUri, scopes });
    oktaAuth.tokenManager.setTokens(tokenResponse.tokens);
    await navigateWithAccessToken(tokenResponse.tokens.accessToken?.accessToken, tokenResponse.tokens.refreshToken?.refreshToken);
    break;
  }
  await new Promise(resolve => setTimeout(resolve, 2000));
}
```

---

## 🧠 Token Management

All token handling is centralized using Okta’s `tokenManager`:

```tsx
oktaAuth.tokenManager.setTokens(tokens);
```

It handles both access and refresh tokens, which are used in secured API requests and session restoration.

---

## 🔒 Security Considerations

- Tokens are securely managed using Okta's built-in `tokenManager`.
- Session revalidation is performed to reduce unnecessary logins.
- Polling for email authentication is rate-limited via `setTimeout`.

---

## 🧪 Error Handling

All major errors (network, IDX status, validation) are captured and routed to UI error messages:

```tsx
if (messages) {
  const errorMessage = messages[0].message;
  setErrors({ form: errorMessage });
}
```

---

# 📘 Okta IDX Registration Integration

## Overview

This component implements **user registration using Okta Identity Engine (IDX)** in a React application with `@okta/okta-auth-js` and `@okta/okta-react`. It handles:

- User registration via IDX
- Terms and conditions validation
- Token acquisition and storage
- Post-registration navigation

## 🚀 Registration Flow


### **Registering via Okta IDX**

Registration is performed via `oktaAuth.idx.register`, passing user information:

```tsx
const { tokens, messages } = await oktaAuth.idx.register({
  firstName,
  lastName,
  email,
  passcode: password,
});
```

If successful, tokens are saved and the user is redirected accordingly:

```tsx
oktaAuth.tokenManager.setTokens(tokens);
await afterOktaLogin(true, accessToken, refreshToken);
navigate('/setup-payment');
```

## 🧪 Error Handling

The code includes robust error mapping based on response messages:

```tsx
if (errorMessage.includes('Email already exists')) {
  setErrors({ email: errorMessage });
} else if (errorMessage.includes('invalid param password')) {
  setErrors({ password: 'Invalid password format' });
}
```

Fallback messaging and clearing the password field are included for security.

# 📘 Okta IDX Password Reset Flow Integration

## Overview

This set of components enables a complete **password recovery flow** using Okta Identity Engine (IDX) in a React application. It consists of:

1. **ResetPassword** – Initiates password recovery via email
2. **OktaOtp** – Handles OTP (email verification code)
3. **NewPassword** – Accepts and confirms new password

## 🔁 Flow Summary

1. **User enters email** on the Reset Password screen
2. **Okta sends a verification code** to their email
3. **User enters OTP code**
4. **User sets a new password**
5. **Tokens are issued and saved**

---

## 🔐 1. Reset Password (`ResetPassword.tsx`)

Initiates the password recovery by email using `recoverPassword` flow:

```tsx
await oktaAuth.idx.startTransaction({ flow: 'recoverPassword' });
await oktaAuth.idx.proceed({
  ...formData,
  authenticators: [AuthenticatorKey.OKTA_EMAIL],
});
```

Then verifies if OTP is required:

```tsx
const { nextStep } = await oktaAuth.idx.proceed({ methodType: 'email' });
if (nextStep?.inputs?.some((item) => item.name === 'verificationCode')) {
  navigate('/otp');
}
```

---

## 🔢 2. OTP Verification (`OktaOtp.tsx`)

Handles user input of a 6-digit code and proceeds with the flow:

```tsx
const { nextStep, messages } = await oktaAuth.idx.proceed({ ...formData });

if (nextStep?.inputs?.some((item) => item.name === 'password')) {
  navigate('/new-password');
}
```

Validation schema ensures OTP is 6 digits:

```tsx
object().shape({
  verificationCode: string().required().length(6),
});
```

---

## 🔑 3. New Password Setup (`NewPassword.tsx`)

Allows user to set and confirm a new password. If passwords match, proceeds with Okta:

```tsx
if (password !== passwordConfirmation) {
  setErrors({ form: t('reset.passwords_do_not_match') });
  return setSubmitting(false);
}
const { status, tokens } = await oktaAuth.idx.proceed({ password });
if (status === IdxStatus.SUCCESS && tokens) {
  oktaAuth.tokenManager.setTokens(tokens);
  await afterOktaLogin(true, tokens.accessToken?.accessToken, tokens.refreshToken?.refreshToken);
}
```

Validation ensures password has a minimum of 8 characters:

```tsx
object().shape({
  password: string().matches(/^.{8,}$/).required(),
  passwordConfirmation: string(),
});
```

---

## 🧪 Error Handling

Across all components, Okta IDX errors are parsed and routed into form errors:

```tsx
if (error.message.includes('invalid param email')) {
  setErrors({ username: t('reset.wrong_email') });
}
```

Special handling is in place for:

- Invalid tokens
- Password policy violations
- Duplicate accounts

---

## 📚 References

- [Okta Identity Engine SDK Docs](https://github.com/okta/okta-auth-js)
- [Okta React SDK](https://github.com/okta/okta-react)
- [Official Okta Documentation](https://developer.okta.com/docs/guides/)
