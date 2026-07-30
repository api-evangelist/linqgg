---
name: linqgg-onboard-player
description: Register a game player in LinQ Wallet services and optionally link them to a real wallet account, so they can hold a balance and move money.
api: LinQ Wallet Public API
generated: '2026-07-19'
method: generated
source: https://docs.linq.gg/modules/auth
operations:
  - linq.geo.restrictions.v2.RestrictionsService.IsAccessAllowed
  - linq.auth.game.v1.AuthGameService.SignInGame
  - linq.auth.user.v1.AuthUserService.SignUpGameUser
  - linq.auth.user.v1.AuthUserService.SignInGameUser
  - linq.auth.user.v1.AuthUserService.SaveGameUser
  - linq.money.accounts.v1.AccountsService.GetActualBalance
---

# Onboard a player into LinQ Wallet

LinQ has two player modes. **Conditionally anonymous** players can hold a game
balance, top it up, pay entry fees and win prizes, but cannot move money out.
**Authorized** players have linked a real LinQ wallet account and can do
everything, including KYC and withdrawal. Onboard every player anonymously
first, then offer linking.

## 1. Confirm the region is allowed

Call `RestrictionsService.IsAccessAllowed` (package `linq.geo.restrictions.v2`).
It takes `google.protobuf.Empty` and needs **no authorization** — it is the
correct first smoke test of any integration.

Read `allowed`. If false, the player's region is restricted and you must not
proceed to account creation or any money operation. `location` gives country,
region and city.

Do not use `linq.geo.restrictions.v1.RestrictionsService.isAccessAllowed` — the
whole v1 package is marked `option deprecated = true` in the contract.

## 2. Create the anonymous account

Immediately after the player registers or first logs into your game, call
`AuthGameService.SignInGame` from your **backend** with:

- `game_token` — your private secret key, supplied as an environment variable.
  Never ship it in client code.
- `game_user_id` — your internal, immutable user identifier. Use an incremental
  id or UUID. Do not use a username or email, because the player can change those.

Store the returned `access_token` on the user profile. It has unlimited lifetime.
Send it on every later call as gRPC metadata `authorization: Bearer <token>`.

## 3. Verify the account works

Call `AccountsService.GetActualBalance` with the game's internal currency code
and the access token. A successful response confirms the internal account exists.

Balances are integers in coins. Divide by 100 before showing them to the player.

## 4. Offer wallet linking

When the player wants to withdraw, link them to a real wallet.

Call `AuthUserService.SignUpGameUser` with the player's access token. It returns
`user_token`, a short-lived token for the linking handshake.

You may pass a `UserData` message (`phone`, `email`, `first_name`, `last_name`,
`dob`) to pre-create the wallet profile. If you do, you **must** warn the player
in plain language that an account will be created for them in a third-party
application, and the docs recommend an explicit checkbox or confirmation dialog.
Check `mismatched_user_fields` on the response — it reports supplied values that
did not match an existing wallet profile, and it arrives alongside a valid token
rather than as an error.

## 5. Hand off to the LinQ app

Send `user_token` to the LinQ Wallet app by either route:

- Deep link — `linq://?user_token={token}` in production,
  `linq-stg://?user_token={token}` in stage.
- Associated domain — `https://a.linq.gg?user_token={token}` in production,
  `https://a.stg.linq.gg?user_token={token}` in stage. This opens the app if it
  is installed and redirects to the App Store (or TestFlight in stage) if not.

If the app is not installed, write the token to the shared Keychain access group
with `LinqUnity.Keychain.setAuthUserToken(token, accessGroup)` using
`$(AppIdentifierPrefix)games.galactica.linq.shared` (or `.stg.shared`). The LinQ
app reads it after install and authorizes the game. This works on Apple platforms
only and requires both apps to share one App Store account.

## 6. Complete the link

After the player confirms in the LinQ app and returns to the game, call
`AuthUserService.SignInGameUser` with the `user_token`. Set
`ignore_game_payment_accounts_duplicates` when you want the existing wallet
account to win over a previously created anonymous account.

Store the returned `access_token` as the player's **Wallet Token**. Keep both
tokens and prefer the wallet token from then on:

```
getAuthorization(user.walletToken ?? user.accessToken)
```

## 7. Lifting limits without full linking

If a player will not link a wallet, call `AuthUserService.SaveGameUser` with
their `UserData` to attach details to the anonymous profile. This is what raises
the anonymous money-operation limits, and it is required before generating a Pix
code for an anonymous Brazilian player.

## Testing

Work against `services.stage.galactica.games`. To skip the real in-app
confirmation, call `HelpersService.VerifyPlayerToken` with the player's wallet
alias and the token. It exists only in stage — calling it in production errors
with "not implemented method".

## Anonymous mode limits

- Cannot transfer to the wallet account or to other games.
- Total deposit amount capped at 20.00.
- Check headroom with `PaymentsService.IsLimitReached`.
