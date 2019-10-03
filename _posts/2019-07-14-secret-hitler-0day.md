This is a zero-day I discovered on the open source [Secret Hitler](https://www.secrethitler.com/) game.

By submitting crafted parameters to the `/password-reset` endpoint, attackers are able to takeover arbitrary non-staff accounts.

**This vulnerability can be mitigated by disabling JSON parsing.**

We control all of the parameters passed through `req.body`.

```javascript
const { username, password, password2, tok } = req.body;
```

The code that searches for a token is as follows.

```javascript
ResetPassword.findOneAndDelete({ username, token: tok, expirationDate: { $gte: now } })
```

In order to satisfy this, we can set 

```javascript
tok = {
  $ne: 1
}
```

The `$ne` specifier will return any object where the validation token is not equal to 1.

The code that loads the user profile to reset the password for is as follows.

```javascript
Account.findOne({ username: req.body.username })
```

Note that the crafted username will have to satisfy both the token search and the user search. In other words, the specifier we use must both have a reset token, and the account of the user we want to takeover.

This last constraint requires a bit more creativity to satisfy. We have two options.

1. Takeover an account that currently has an reset token active. This would severely impact the scope of our exploit however.
2. Use the `username` specifier to specify a large set of accounts (e.g. `{$ne: 1}`), and hope `Account.findOne` chooses the account we want.

Option 2 offers slightly more promise, and eventually lead me to the arbitrary account takeover.

By creating an account with username `z`, I could ensure that in terms of string comparisons, my account was very high on the list.

Then, setting

```javascript
username = {
  $gte: "account_to_takeover"
}
```

finishes the exploit. As long as there is a valid reset token from account `z`, we can takeover any account with name less than `z`. Note that by extending the username to `zzzzz...`, we can ensure arbitrary account takeover.

The `ResetPassword` constraint is satisfied because it will find the reset token generated by `z`. The `Account` constraint is also satisfied because `findOne` will return the first object according to the natural sort order (alphabetically), which would be the account with name equal to the `$gte$` condition.

Aside from staff accounts which are hardcoded to never accept any password resets

```javascript
if (!account || account.staffRole) {
  // Error
}
```

this method suffices.