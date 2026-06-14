Test repository for encryption/decryption with git clean/smudge filters using OpenSSL CMS.

First create a key and certificate with:
```sh
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out req.pem -nodes
```
(the -nodes option makes it so the key will not be encrypted,
otherwise it will prompt you for a password to encrypt it with, and
you will have to provide this password when decrypting)

Included is a POSIX shell script called `crypt` to encrypt/decrypt with OpenSSL CMS AES-GCM. The certificate and key are hardcoded respectively as `req.pem` and `key.pem`.

To use as a git smudge/clean filter:
```sh
git config --global filter.crypt.clean 'crypt encrypt'
git config --global filter.crypt.smudge 'crypt decrypt'
git config --global filter.crypt.required true
# ^ makes it not stage the file if the filter errors
```
and create a .gitattributes with:
```conf
*.txt filter=crypt
```
or a different pattern instead of the `*.txt` to match the files you
want to affect.

Keep key.pem or its contents safe! Without it, you will not be able
to decrypt. The certificate is not as important, it's possible to
decrypt without it; the ciphertext is a CMS structure that contains
encryptedKey, encryptedContent, the IV, and the MAC. You can decrypt
encryptedKey with your RSA key in key.pem (with RSA-OAEP SHA256),
and decrypt encryptedContent+MAC with it and the IV (with AES-GCM).

([script that demonstrates this process of decrypting without certificate](https://github.com/plu5/dotfiles/blob/main/pm/scripts/popenssl))
