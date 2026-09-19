Test repository for encryption/decryption with git clean/smudge filters using JWE or OpenSSL CMS.

> [!CAUTION]
> There are issues with this approach. The filter isn't stable. Running clean twice on the same plaintext results in a different ciphertext. Git compares the content post-clean. Normally git doesn't compare the contents of the files if the metadata hasn't changed, so on the same local repository you would not at first notice a problem, but if there is some tool that changes metadata on files en masse, they will all be marked as changed despite not having changed. Merge will be broken too. Clone/checkout likewise resulting in unclean working directory without any real change; it can be made to work, but only by forcing git to accept the files as unmodified with `git -c filter.crypt.clean="git show HEAD:%f" update-index --really-refresh`.
>
> A bad situation would be something changing the metadata on files en masse provoking this problem without the user knowing which files really changed.
>
> It's also not idempotent; clean(clean(x)) ≠ clean(x), but I don't understand git well enough yet to understand where this would cause problems (merging?). It's not impossible to make the filter idempotent by just not encrypting again if it sees it's already in JWE/CMS format.
>
> [Discussion](https://github.com/plu5/plu5.github.io/blob/master/_devlog/17-git-clean-smudge.md#filter-stability)

## Contents

1. [Set up: JWE](#set-up-jwe)
1. [Set up: OpenSSL CMS](#set-up-openssl-cms)
1. [A note on cloning with the crypt filter active](#a-note-on-cloning-with-the-crypt-filter-active)

## Set up: JWE
Using [latchset/jose](https://github.com/latchset/jose), which is available in Arch Linux on extra/jwe.
There exist other JOSE/JWI CLI (as well as libraries for all popular programming languages) which you could use instead, and adapt the commands below according to their interface.

(1) Create a key with:
```sh
jose jwk gen -i '{"alg": "A256GCM"}' -o oct.jwk
```

(2) Add a git smudge/clean filter:
```sh
git config --global filter.crypt.clean 'jose jwe enc -I- -k oct.jwk --compact'
git config --global filter.crypt.smudge 'jose jwe dec -i- -k oct.jwk'
git config --global filter.crypt.required true
# ^ makes it not stage the file if the filter errors
```
Or without `--global` to only apply to current repository.

(3) Create a .gitattributes with:
```conf
*.txt filter=crypt
```
or a different pattern instead of the `*.txt` to match the files you
want to affect.

Encrypted file format ([JWE Compact Serialisation](https://datatracker.ietf.org/doc/html/rfc7516#section-3.1)):
```text
eyJhbGciOiJkaXIiLCJlbmMiOiJBMjU2R0NNIn0..RWwAina2Y6pFiKp3.Jj4j.QitJY2B1RBZ-8S0Dm_2yUA
```
`protectedHeader.encryptedKey.iv.encryptedContent.mac`. The first field, `eyJhbGciOiJkaXIiLCJlbmMiOiJBMjU2R0NNIn0`, is the protectedHeader `{alg: "dir", enc: "A256GCM"}`, and will always be this same value for all things encrypted with these settings. `alg` is the key encryption algorithm (key wrap). `dir` means no key wrap; directly encrypt the content. The `..` in the resulting structure is because the encryptedKey field is empty.

> [!NOTE]
> Key wrapping is useful for (1) multiple recipients, (2) not having to update encryptedContent to rotate the KEK (key encryption key), (3) JWE will encrypt each file with a different CEK (content encryption key), so if it leaks only one file is compromised (though in practice you can only decrypt the CEK if you have the KEK. In theory the CEK without the KEK could leak in logs or memory dumps).
>
> I think there's not much benefit wrapping the key in this use case (even if you have multiple developers, the encryptedKey field on each file will be the same for everyone, so everyone needs the same key in practice to be able to decrypt it), and "dir" is simpler and faster, so that's what I opt for.

Keep oct.jwk safe, or at least save the value in its k field. Without it, you will not be able
to decrypt.

## Set up: OpenSSL CMS
(1) Create a key and certificate with:
```sh
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out req.pem -nodes
```
(the -nodes option makes it so the key will not be encrypted,
otherwise it will prompt you for a password to encrypt it with, and
you will have to provide this password when decrypting)

(2) Add a git smudge/clean filter:
```sh
git config --global filter.crypt.clean 'openssl cms -encrypt -aes-256-gcm -recip req.pem -keyopt rsa_padding_mode:oaep -keyopt rsa_oaep_md:sha256'
git config --global filter.crypt.smudge 'openssl cms -decrypt -recip req.pem -inkey key.pem'
git config --global filter.crypt.required true
# ^ makes it not stage the file if the filter errors
```
Or without `--global` to only apply to current repository.

AES256-GCM and RSA-OAEP were chosen because they're state of the art and widely supported (including in WebCrypto).

(3) Create a .gitattributes with:
```conf
*.txt filter=crypt
```
or a different pattern instead of the `*.txt` to match the files you
want to affect.

Encrypted file format ([S/MIME](https://en.wikipedia.org/wiki/S/MIME) [CMS AuthEnvelopedData](https://datatracker.ietf.org/doc/html/rfc5083) DER):
```text
MIME-Version: 1.0
Content-Disposition: attachment; filename="smime.p7m"
Content-Type: application/pkcs7-mime; smime-type=authEnveloped-data; name="smime.p7m"
Content-Transfer-Encoding: base64

MIICBQYLKoZIhvcNAQkQARegggH0MIIB8AIBADGCAaQwggGgAgEAMF0wRTELMAkG
A1UEBhMCQVUxEzARBgNVBAgMClNvbWUtU3RhdGUxITAfBgNVBAoMGEludGVybmV0
IFdpZGdpdHMgUHR5IEx0ZAIUGvJxJ2ygSUUDAH3nkXc2GNLWPAYwOAYJKoZIhvcN
AQEHMCugDTALBglghkgBZQMEAgGhGjAYBgkqhkiG9w0BAQgwCwYJYIZIAWUDBAIB
BIIBACV9y8Ht8m82CjH4yXcDOwzbO4aipp/cOTuR434G5mL940/l2F2pjmlNOXs/
A0XgUK0lm8l1Dddmt1UEIWrVqsFWkNuVyIPITFVP4/SDjYuiD/5TYX2n7m6gF2qD
bHgKF3WeF5786OsFO79FUR46apjc1DRcZ2sqDE0UkGr4gzY1CAQOEEX1LlRVaQEQ
ZK5pdZBBzHlHOTjnqIOs5idN5WtR0zMVtyvzYtJ7IHLGl5tfBJyavnb8P32MSv6W
O7wvXLJvgmoQHVNgVEUtEMhxvuXsndgtRRTaNDdtPD3JTjFVgideJbIUwlYXosie
cI2Q4iWZrpH8n65xItGMw7OjBSYwMQYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjAR
BAyZmnZj1v8xz+D9ICQCARCABLS5GdoEEGteIMONmhZviZ6YHsSd+x0=
```
Unfortunately there is not wide support for AuthEnvelopedData. It's hard to work with it on the web, for example (I can't find any JavaScript libraries that support it, pki.js only supports EnvelopedData as of June 2026).

Keep key.pem safe. Without it, you will not be able
to decrypt. The certificate is not as important, it's possible to
decrypt without it; the ciphertext is a CMS structure that contains
encryptedKey, encryptedContent, the IV, and the MAC. You can decrypt
encryptedKey with your RSA key in key.pem (with RSA-OAEP SHA256),
and decrypt encryptedContent+MAC with it and the IV (with AES-GCM).

([script that demonstrates this process of decrypting without certificate](https://github.com/plu5/dotfiles/blob/main/pm/scripts/popenssl))

## A note on cloning with the crypt filter active
With the filter present in your global gitconfig and set as required, files that are should filter according to the gitattributes will error when cloning due to the absence of key/certificate in the folder.
You can copy them over (oct.jwk in the case of JWK, req.pem and key.pem in the case of OpenSSL CMS) after cloning, then run `git restore --source=HEAD :/`. Otherwise the files that should filter will be absent (hi.txt in this case).
```sh
$ git clone https://github.com/plu5/git-filters-test
Cloning into 'git-filters-test'...
remote: Enumerating objects: 7, done.
remote: Counting objects: 100% (7/7), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 7 (delta 1), reused 7 (delta 1), pack-reused 0 (from 0)
Receiving objects: 100% (7/7), done.
Resolving deltas: 100% (1/1), done.
Could not open file or uri for loading recipient certificate file from req.pem: No such file or directory
error: external filter 'crypt decrypt' failed 2
error: external filter 'crypt decrypt' failed
fatal: hi.txt: smudge filter crypt failed
warning: Clone succeeded, but checkout failed.
You can inspect what was checked out with 'git status'
and retry with 'git restore --source=HEAD :/'

$ cp ../req.pem req.pem
$ cp ../key.pem key.pem
$ ls
COPYING  crypt  key.pem  README.md  req.pem
$ git restore --source=HEAD :/
$ ls
COPYING  crypt  hi.txt  key.pem  README.md  req.pem
$ cat hi.txt
hi
```

Faster way:
Create a new folder and copy over your key/certificate. Then while inside that folder:
```sh
git clone --bare https://github.com/plu5/git-filters-test .git
git config --unset core.bare
git reset --hard
```
