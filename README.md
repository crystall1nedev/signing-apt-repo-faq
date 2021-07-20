# How to sign a Cydia repository... correctly.

To prevent your users from possibly facing man-in-the-middle attacks and secure your repository, you should follow these steps. To get started, we'll need you to have a GPG key to sign with and export the public key (it's easier than you might think). I provide instructions for Linux and macOS because these are the two operating systems I use--this should be doable in WSL or Cygwin.

1. **Get the GnuPG utilities.** This is most likely included on your distro of Linux and can be installed with `sudo apt install gnupg` or `brew install gnupg` on macOS.

2. **Generate a new key.** Open a terminal and run `gpg --full-generate-key`. Pick `(9) ECC (sign and encrypt)` as the type, pick `(1) Curve 25519 *default*` for your curve, and `0` to keep the key from expiring (if you don't have the option to use ECC, choose `(1) RSA and RSA`, then select `4096` as your key length). Then enter your real name, email address, and an optional comment to fill out the signature details.
```
gpg (GnuPG) 2.4.9; Copyright (C) 2025 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (14) Existing key from card
Your selection? 9
Please select which elliptic curve you want:
   (1) Curve 25519 *default*
   (4) NIST P-384
   (6) Brainpool P-256
Your selection? 1
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: Eva Isabella Luna
Email address: me@crystall1ne.dev
Comment: crystll1ne
You selected this USER-ID:
    "Eva Isabella Luna (crystll1ne) <me@crystall1ne.dev>"
```

Once you've done this, hit `O` and enter to save your settings, enter a password (you can leave it blank if you desire) and there you go! You now have a new key to sign with.

3. **Find the key's signature.** `gpg --list-secret-keys --keyid-format=long` will list the keys you've created on your system. What you're after is the `sec` of the key, which is expanded in full on the second line (what the blue arrow is pointing to).
```
sec   ed25519/A9950BA7CB3F3C03 2026-05-04 [SC]
      FA9417C900DE745D12939638A9950BA7CB3F3C03
uid                 [ultimate] Eva Isabella Luna (crystll1ne) <me@crystall1ne.dev>
ssb   cv25519/663F59A2DD7C4C27 2026-05-04 [E]
```


4. **Create Release.gpg.** With this signature copied, you can now run the next command: `gpg -abs -u <what you copied> -o Release.gpg Release`. This signs your Release and creates Release.gpg, which marks the first part of signing properly complete.
(example: `gpg -abs -u 0EA1CD919AF64DDE44F2AF3EAEE9BA5226A6631B -o Release.gpg Release`)
5. **Export your public key.** Now, the final step is to export the public key. This allows your users to download and trust your repo is really yours. Remember that picture up above with the two arrows? This time, you need to copy what the yellow arrow is pointing to (all of the numbers and letters to the right of `rsa4096/`. Once copied, you can type `gpg --export <what you copied> > <yourname/username>-repo.gpg`.
(example: `gpg --export AEE9BA5226A6631B > doregon-repo.gpg`)

6. **Upload the public key to a location where your users can access it!** You can upload the key to the README of your repository on GitHub, for example. Alternatively, you can open a pull request to add your key to Procursus's keyring, so all your users need to do is look for your repo's name in the Keyrings section. If you send your public key to me, I will handle packaging and adding keys to the Procursus keyring for you.
