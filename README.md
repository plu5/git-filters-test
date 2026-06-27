Test repository for encryption/decryption with git clean/smudge filters using OpenSSL CMS.

## Set up
First create a key and certificate with:
```sh
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out req.pem -nodes
```
(the -nodes option makes it so the key will not be encrypted,
otherwise it will prompt you for a password to encrypt it with, and
you will have to provide this password when decrypting)

To use as a git smudge/clean filter:
```sh
git config --global filter.crypt.clean 'openssl cms -encrypt -aes-256-gcm -recip req.pem -keyopt rsa_padding_mode:oaep -keyopt rsa_oaep_md:sha256'
git config --global filter.crypt.smudge 'openssl cms -decrypt -recip req.pem -inkey key.pem'
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

## Cloning
Create a new folder and copy your req.pem and key.pem to it. Then while inside that folder:
```sh
git clone --bare https://github.com/plu5/git-filters-test .git
git config --unset core.bare
git reset --hard
```

Alternatively you can clone normally, which will error due to the absence of req.pem and key.pem, then add these files to the folder and restore.
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
