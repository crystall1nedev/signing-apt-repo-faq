# How to sign a Cydia repository... correctly.

To prevent your users from possibly facing man-in-the-middle attacks and secure your repository, you should follow these steps. To get started, we'll need you to have a GPG key to sign with and export the public key (it's easier than you might think). I provide instructions for Linux and macOS because these are the two operating systems I use--this should be doable in WSL or Cygwin.

1. **Get the GnuPG utilities.** This is most likely included on your distro of Linux and can be installed with `sudo apt install gnupg` or `brew install gnupg` on macOS.

2. **Generate a new key.** Open a terminal and run `gpg --full-generate-key`. Pick `(9) ECC (sign and encrypt)` as the type, pick `(1) Curve 25519 *default*` for your curve, and `0` to keep the key from expiring (if you don't have the option to use ECC, choose `(1) RSA and RSA`, then select `4096` as your key length). Then enter your real name, email address, and an optional comment to fill out the signature details.
![Screen Shot 2021-07-20 at 3 19 37 PM](https://user-images.githubusercontent.com/55281754/126382964-d7e483ae-89ef-4161-a807-16d411f5336e.png)
Once you've done this, hit `O` and enter to save your settings, enter a password (you can leave it blank if you desire) and there you go! You now have a new key to sign with.

3. **Find the key's signature.** `gpg --list-secret-keys --keyid-format=long` will list the keys you've created on your system. What you're after is the `sec` of the key, which is expanded in full on the second line (what the blue arrow is pointing to).
![Screen Shot 2021-07-20 at 3 34 09 PM](https://user-images.githubusercontent.com/55281754/126384855-76c90f0f-c5c4-45ab-b93a-44190bf6b616.png)


4. **Create Release.gpg.** With this signature copied, you can now run the next command: `gpg -abs -u <what you copied> -o Release.gpg Release`. This signs your Release and creates Release.gpg, which marks the first part of signing properly complete.
(example: `gpg -abs -u 0EA1CD919AF64DDE44F2AF3EAEE9BA5226A6631B -o Release.gpg Release`)
5. **Export your public key.** Now, the final step is to export the public key. This allows your users to download and trust your repo is really yours. Remember that picture up above with the two arrows? This time, you need to copy what the yellow arrow is pointing to (all of the numbers and letters to the right of `rsa4096/`. Once copied, you can type `gpg --export <what you copied> > <yourname/username>-repo.gpg`.
(example: `gpg --export AEE9BA5226A6631B > doregon-repo.gpg`)

6. **Upload the public key to a location where your users can access it!** You can upload the key to the README of your repository on GitHub, for example. Alternatively, you can open a pull request to add your key to Procursus's keyring, so all your users need to do is look for your repo's name in the Keyrings section. If you send your public key to me, I will handle packaging and adding keys to the Procursus keyring for you.
