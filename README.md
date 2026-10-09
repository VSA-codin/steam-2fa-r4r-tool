> **PROJECT MOVED:** Steam Guard 2FA is now integrated into [VSA rep4rep-bot](https://github.com/VSA-codin/rep4rep-bot/tree/main/steam-2fa). This standalone repository is no longer maintained.

# steam-2fa-r4r-tool

Steam 2FA helper for **R4R-bot**. It enrolls Steam Guard Mobile, finishes interrupted enrollments, stores per-account 2FA secrets locally on a VPS, and generates Steam Guard codes for R4R-bot when Steam asks for `SteamGuardMobile`.

**Made by VSA**  
**Discord:** `vacsecuredapproved`

> **Important:** this repository contains code only. Never commit Steam passwords, cookies, `shared_secret`, `identity_secret`, `revocation_code`, SDA/maFiles, or your local 2FA store.

## Recommended VPS layout

```text
~/apps/R4R-bot
~/apps/steam-2fa-r4r-tool

~/.config/r4r/
├── steam-2fa.json
└── ACCOUNT.2fa-pending.json
```

The repository and the secret store are intentionally separate.

## Features

- Steam Guard Mobile enrollment
- safe continuation of an interrupted enrollment
- per-account `shared_secret` storage outside the repository
- automatic Steam TOTP generation for R4R-bot
- `0700` config directory and `0600` secret files
- manual Steam Guard fallback if an account has no stored secret
- no Steam passwords stored by this tool

## Requirements

- Linux VPS
- Node.js 20+
- npm
- Steam account credentials for accounts you own/control

The versions in `package.json` match the setup this tool was built around.

## Install

```bash
cd ~/apps
git clone https://github.com/VSA-codin/steam-2fa-r4r-tool.git
cd steam-2fa-r4r-tool
npm install
npm run check
```

## Enroll one Steam account

Do not put your Steam password directly into shell history.

```bash
cd ~/apps/steam-2fa-r4r-tool

read -rp "Steam login: " STEAM_USER
read -rsp "Steam password: " STEAM_PASS
echo

export STEAM_USER STEAM_PASS
node scripts/enroll.js
unset STEAM_USER STEAM_PASS
```

Depending on the account, Steam may ask for an email code, phone/SMS activation code, or another confirmation step.

On success, the account is stored locally in:

```text
~/.config/r4r/steam-2fa.json
```

The file is created with mode `0600`.

### Interrupted enrollment

If Steam accepted the start of enrollment but finalization did not finish, the tool preserves:

```text
~/.config/r4r/ACCOUNT.2fa-pending.json
```

Do **not** immediately start a new enrollment if a valid pending file already exists.

Finish it with:

```bash
cd ~/apps/steam-2fa-r4r-tool

ACCOUNT="your_steam_login"

read -rsp "Steam password: " STEAM_PASS
echo

export STEAM_PASS
node scripts/finalize.js "$ACCOUNT"
unset STEAM_PASS
```

The pending file is deleted only after successful finalization.

## Connect it to the current R4R-bot

Assuming:

```text
~/apps/R4R-bot
~/apps/steam-2fa-r4r-tool
```

### Option A — direct sibling import

In `~/apps/R4R-bot/index.js` add near the top:

```js
const {
    generateCodeForAccount
} = require('../steam-2fa-r4r-tool');
```

Then use this `SteamGuardMobile` handler inside your existing `doLogin(...)`:

```js
if (err.message === 'SteamGuardMobile') {
    try {
        const code = generateCodeForAccount(accountName);

        console.log(
            '[2FA] Generated Steam Guard code automatically.'
        );

        doLogin(accountName, password, null, code);
    } catch (secretErr) {
        console.log('[2FA] ' + secretErr.message);

        rl.question(
            'Steam Authenticator Code: ',
            function(code) {
                doLogin(
                    accountName,
                    password,
                    null,
                    code.trim()
                );
            }
        );
    }

    return;
}
```

This is the simplest integration for your current VPS folder layout.

### Option B — install it as a local package

```bash
cd ~/apps/R4R-bot
npm install ../steam-2fa-r4r-tool
```

Then:

```js
const {
    generateCodeForAccount
} = require('steam-2fa-r4r-tool');
```

A standalone example is in `examples/r4r-integration.js`.

## What to remove from the old R4R integration

If R4R already contains its own copy of:

```js
const SteamTotp = require('steam-totp');
```

and a custom function such as:

```js
getSteam2FASecret(accountName)
```

you can remove those **after** the new helper is tested successfully. The new repository owns that logic.

Before editing R4R:

```bash
cd ~/apps/R4R-bot
cp index.js index.js.before-steam-2fa-tool.bak
```

After editing:

```bash
node --check index.js
```

Do not restart R4R in the middle of an active farming run.

## Test the helper without printing secrets

```bash
cd ~/apps/steam-2fa-r4r-tool

STEAM_USER="your_steam_login" node - <<'NODE'
const {
    hasAccountSecrets
} = require('./');

console.log(
    hasAccountSecrets(process.env.STEAM_USER)
        ? '2FA READY'
        : '2FA MISSING'
);
NODE
```

To generate a code from application code:

```js
const {
    generateCodeForAccount
} = require('../steam-2fa-r4r-tool');

const code = generateCodeForAccount(accountName);
```

Do not print generated codes to logs unless you are debugging locally.

## Updating

```bash
cd ~/apps/steam-2fa-r4r-tool
git pull
npm install
npm run check
```

If you used Option B:

```bash
cd ~/apps/R4R-bot
npm install ../steam-2fa-r4r-tool
```

## systemd and unattended R4R runs

This tool handles the **Steam Guard code** part of a login.

It intentionally does **not** store Steam account passwords and therefore does not, by itself, make R4R automatically re-login after a saved Steam session expires.

If R4R is later extended to perform unattended re-login, keep passwords in a separate system secret mechanism such as systemd credentials. Do not put passwords in this repository or in `steam-2fa.json`.

## HTTP 429

Steam can rate-limit enrollment requests.

If `enableTwoFactor()` returns HTTP 429:

1. stop repeated enrollment attempts;
2. check whether an `ACCOUNT.2fa-pending.json` file already exists;
3. if a valid pending enrollment exists, use `scripts/finalize.js` instead of starting a new enrollment;
4. do not delete pending enrollment data until you know it is no longer needed.

## Security

Read [SECURITY.md](SECURITY.md) before using this with real accounts.

---

Made by **VSA**  
Discord: **vacsecuredapproved**
