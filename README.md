# yubikey-agent

yubikey-agent is a seamless ssh-agent for YubiKeys.

* **Easy to use.** A one-command setup, one environment variable, and it just runs in the background.
* **Indestructible.** Tolerates unplugging, sleep, and suspend. Never needs restarting.
* **Compatible.** Provides a public key that works with all services and servers.
* **Secure.** The key is generated on the YubiKey and can't be extracted. Every session requires the PIN, every login requires a touch. Setup takes care of PUK and management key.

Written in pure Go, it's based on [github.com/go-piv/piv-go](https://github.com/go-piv/piv-go) and [golang.org/x/crypto/ssh](https://golang.org/x/crypto/ssh).

## Installation

### macOS

```
brew tap filippo.io/yubikey-agent https://filippo.io/yubikey-agent
brew install yubikey-agent
brew services start yubikey-agent

yubikey-agent -setup # generate a new key on the YubiKey
```

Then add the following line to your `~/.zshrc` and restart the shell.

```
export SSH_AUTH_SOCK="/usr/local/var/run/yubikey-agent.sock"
```

## Advanced topics

### Conflicts with `gpg-agent` and Yubikey Manager

`yubikey-agent` takes a persistent transaction so the YubiKey will cache the PIN after first use. Unfortunately, this makes the YubiKey PIV and PGP applets unavailable to any other applications, like `gpg-agent` and Yubikey Manager. Our upstream [is investigating solutions to this annoyance](https://github.com/go-piv/piv-go/issues/47).

If you need `yubikey-agent` to release its lock on the YubiKey, send it a hangup signal. Likewise, you might have to kill `gpg-agent` after use for it to release its own lock.

```
killall -HUP yubikey-agent
```

This does not affect the FIDO2 functionality.

### Unblocking the PIN with the PUK

If the wrong PIN is entered incorrectly three times in a row, YubiKey Manager can be used to unlock it.

`yubikey-agent -setup` sets the PUK to the same value as the PIN.

```
ykman piv unblock-pin
```

If the PUK is also entered incorrectly three times, the key is permanently irrecoverable. The YubiKey PIV applet can be reset with `yubikey-agent -setup --really-delete-all-piv-keys`.

### Manual setup and technical details

`yubikey-agent` only officially supports YubiKeys set up with `yubikey-agent -setup`.

In practice, any PIV token with an RSA or ECDSA P-256 key and certificate in the Authentication slot should work, with any PIN and touch policy.

`yubikey-agent -setup` generates a random Management Key and [stores it in PIN-protected metadata](https://pkg.go.dev/github.com/go-piv/piv-go/piv?tab=doc#YubiKey.SetMetadata). Note that this is a different scheme from the `ykman` one, which enables PIN bruteforcing.

### Alternatives

#### Native FIDO2

Recent versions of OpenSSH [support using FIDO2 tokens as keys](https://buttondown.email/cryptography-dispatches/archive/cryptography-dispatches-openssh-82-just-works/). Since those are their own key type, they require server-side support, which is currently not available in Debian stable or on GitHub.

FIDO2 keys also usually don't require a PIN, but depending on the token can require a private key file. `yubikey-agent` keys can be ported to a different machine simply by plugging in the YubiKey.

#### `gpg-agent`

`gpg-agent` can act as an `ssh-agent`, and it can use keys stored on the PGP applet of a YubiKey.

This requires a finicky setup process dealing with PGP keys and the `gpg` UX, and seems to lose track of the YubiKey and require restarting all the time. Frankly, I had enough of PGP and GnuPG.

#### `ssh-agent` and PKCS#11

`ssh-agent` can load PKCS#11 applets to interact with PIV tokens directly. There are two PKCS#11 providers for YubiKeys: OpenSC and ykcs11.

The UX of this solution is poor: it requires calling `ssh-add` to load the PKCS#11 module and to unlock it with the PIN (as the agent has no way of requesting input from the client during use, a limitation that `yubikey-agent` handles with `pinentry`), and needs manual reloading every time the YubiKey is unplugged or the machine goes to sleep.

The ssh-agent that ships with macOS (which is pretty cool, as it starts on demand and is preconfigured in the environment) also has restrictions on where the `.so` modules can be loaded from. It can see through symlinks, so a Homebrew-installed `/usr/local/lib/libykcs11.dylib` won't work, while a hard copy at `/usr/local/lib/libykcs11.copy.dylib` will.

#### Secure Enclave

On macOS systems with a Secure Enclave, it would make even more sense to generate the keys on there, use Touch ID for confirmation, and maybe even show the host being authenticated to on the Touch Bar.

There is a project experimenting with that at [github.com/rolandshoemaker/sesa](https://github.com/rolandshoemaker/sesa), but unfortunately compiling software for the Secure Enclave requires specific entitlements from Apple, which complicates development.

#### `pivy-agent`

[`pivy-agent`](https://github.com/joyent/pivy#using-pivy-agent) is part of a suite of tools to work with PIV tokens. It's similar to `yubikey-agent`, and inspired its design.

The main difference is that it requires unlocking via `ssh-add -X` rather than using a graphical pinentry, and it caches the PIN in memory rather than relying on the device PIN policy. It's also written in C.

`yubikey-agent` also aims to provide an even smoother setup process.
