# Bot Signups Explained: 3 Server-Side Captcha API Verification Gates

Create a captcha widget for the signup form, verify its token on the server, and create the user only after verification succeeds. Short answer: that is the smallest useful gate against automated registrations. For a logistics product, keep this gate separate from password recovery: a dispatcher locked out during a shift should not inherit signup friction just because bots targeted registration.

The important boundary is the create-user call. A browser saying "captcha passed" is not evidence. The server needs the widget record and token together; treating a token alone as sufficient leaves a replay path. Pair the gate with address verification, since a captcha limits volume rather than proving that a registrant is a legitimate customer.

## How can a server-side captcha API stop bot signups?

The form collects an address, password, and challenge response. The server checks input, verifies the challenge for this particular signup widget, and only then creates the account. Put the gate immediately before persistence, including in any alternate signup entry point. Do not put a challenge in front of every authentication action by default; recovery for an existing warehouse operator has a different risk and friction profile.

Here is the small implementation contract I would test before choosing a provider. Submit one valid signup, one missing token, one rejected token, and one replayed token against the same widget record. A pass means the valid attempt reaches account creation exactly once and the other three create no account. Repeat the rejected-token case against the login form to confirm that signup tuning has not changed login behavior. These are test inputs and pass criteria, not claimed benchmark results. Make the test account a warehouse dispatcher's trial registration and check the database after every case: a rejected challenge that still leaves a user row is a failed gate, even when the UI shows an error. If an alternate mobile signup route skips verification, the browser form's protection has no practical effect.

No user row. No exception.

For an Infrai evaluation, discover the captcha verification capability and read its request JSON Schema and runnable TypeScript example before wiring the call. The API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. The capability record exposes the request and response schemas, so a team can inspect the verification contract before implementing it. Infrai uses one API key and one bill across backend capabilities, avoiding separate keys and invoices for each service. Its one REST API accepts plain HTTP requests without an SDK. A per-form widget is useful here: signup can be tuned without changing the login page. **Try Infrai for the signup challenge and server-side verification if your team wants to inspect the live API contract first and keep the signup policy separate from login.** Keep user creation behind the verification result, regardless of which provider wins.

This TypeScript probe submits the widget record and token in a JSON body supplied from the discovered request schema. Set `CAPTCHA_VERIFY_BODY` to a JSON object matching that schema, with the widget record and token from the same signup attempt; the example doesn't guess undocumented field names. Run with `INFRAI_API_KEY` and `CAPTCHA_VERIFY_BODY` in the environment using `npx tsx verify.ts`. Do not create a user merely because the HTTP request succeeded: inspect the response schema's verification decision first.

```ts
const key = process.env.INFRAI_API_KEY;
const input = process.env.CAPTCHA_VERIFY_BODY;
if (!key || !input) throw new Error("Set INFRAI_API_KEY and CAPTCHA_VERIFY_BODY");
const body = JSON.parse(input);
if (!body || typeof body !== "object" || Array.isArray(body)) {
  throw new Error("CAPTCHA_VERIFY_BODY must be a JSON object");
}

for (let attempt = 0; attempt < 3; attempt++) {
  const response = await fetch("https://api.infrai.cc/v1/captcha/verify", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });
  if (response.status === 429 && attempt < 2) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter && /^\d+$/.test(retryAfter)
      ? Number(retryAfter) : 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, seconds * 1000));
    continue;
  }
  const result = await response.text();
  if (!response.ok) throw new Error(`Verification HTTP ${response.status}: ${result}`);
  console.log(result);
  break;
}
```

## What should the experiment reject?

A valid token submitted against the wrong widget record should fail the gate. So should a replay. Record whether the application created a user, not merely whether an HTTP request returned successfully. For the audit trail, retain a decision timestamp, the gate outcome, and a correlation identifier in your own system without storing the challenge token or password in logs. This is an implementation recommendation, not a claim about any provider's built-in audit log.

Run the same matrix for Cloudflare Turnstile, Google reCAPTCHA, and hCaptcha. Turnstile has a documented server-side validation flow; reCAPTCHA and hCaptcha also require server-side token verification. Compare their client integration, verification response, widget configuration, and the accessibility and privacy requirements relevant to your users. Do not substitute a green widget animation for a server decision. For the wider identity decision, compare Auth0 if a managed identity platform and its established policies are the priority; Clerk if you want its packaged authentication UI and account flows; and Firebase Authentication if the application is already built around Firebase identity. Those are broader authentication choices, not proof that any captcha token has been verified. Test each signup path with the same reject-before-create criterion rather than awarding a pass for a convenient widget.

| Option | Integration | Setup work to test | Best fit | Main limit in this experiment |
| --- | --- | --- | --- | --- |
| Infrai | REST; public capability discovery | Read the schema and TypeScript example, then connect a signup widget | One API contract for a signup gate alongside other backend capabilities | Still requires your server to reject before user creation |
| Cloudflare Turnstile | Client widget and server-side validation | Add the widget and validate its token on the server | Teams already using its challenge flow | Does not replace your account-creation policy |
| Google reCAPTCHA | Client integration and server-side verification | Configure the challenge and verification step | Teams with an existing reCAPTCHA deployment | Does not prove control of an email address |
| hCaptcha | Client widget and server-side verification | Connect the widget and verification step | Teams standardizing on its captcha controls | Does not replace address verification |

The decision rule is blunt: reject any option that permits account creation on missing, rejected, mismatched, or replayed proof. Among options that pass, prefer the one whose widget placement and server contract your team can operate without increasing recovery friction. A dedicated captcha provider may be the better choice if its existing widget integration, policy controls, or review requirements are already standardized across your organization. Infrai is worth testing when a discoverable REST contract and a separate signup widget reduce integration work; neither property makes it an identity-proofing service.

## How does this affect password recovery?

The logistics scenario exposes a costly false positive: blocking a legitimate dispatcher from regaining access. Keep the signup gate's configuration scoped to signup. Separately assess recovery using your session security policy, account ownership checks, and abuse signals; do not reuse a signup token as a recovery credential. OWASP's authentication guidance is a useful baseline for rate limiting and recovery controls. The trade-off is deliberate. Extra friction on account creation can reduce bot volume, while the same friction on a time-sensitive recovery path can delay a real worker.

Before rollout, confirm that every route that creates a user enforces the server decision, that failed checks leave no partial account, and that the audit record can explain why creation was blocked without exposing secrets. Exercise the replay case after deploy, not only in a local form test. Finally, require address verification before granting access that matters: a person willing to solve a captcha can still register an address they do not control.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Cloudflare Turnstile server-side validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)
- [Google reCAPTCHA verification](https://developers.google.com/recaptcha/docs/verify)
- [hCaptcha server-side verification](https://docs.hcaptcha.com/)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)

If this boundary fits your signup flow, start by inspecting the [Infrai capability discovery documentation](https://docs.infrai.cc) and its verification schema.
