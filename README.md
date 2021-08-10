# How to sign an APT repository... correctly.

To prevent your users from possibly facing man-in-the-middle attacks and secure your repository, you should follow these steps. To get started, we'll need you to have a GPG key to sign with and export the public key (it's easier than you might think). I provide instructions for Linux and macOS because these are the two operating systems I use--this should be doable in WSL or Cygwin.

## Generating keys

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

6. **Verify your key works.** Upload Release.gpg to your repo's server, and move the exported key to `/etc/apt/trusted.gpg.d/` on iOS or `/opt/procursus/etc/apt/trusted.gpg.d` on macOS. If you're successful, you should only see errors about an incorrect Date and missing Hashes (we solve this in the next section). Getting `BADSIG`, `NO_PUBKEY`, or any other GPG related errors means that you should check again and possbily regenerate your keys. 

## Making use of the keys

7. **Generate and sign your files.** When you or your package manager downloads repo files, in this case Packages\* and Release, we need to certify that these files you download are exactly the ones you're supposed to get. It's possible to do so by checking the hash of a file, thanks to protocols like MD5, SHA1, SHA256...  
Each time you're doing a modification to your `deb`s, you'll need to run this script from your repo's root folder:
```sh
apt-ftparchive packages ./debs > Packages
bzip2 -c9 Packages > Packages.bz2
xz -c9 Packages > Packages.xz
xz -5fkev --format=lzma Packages > Packages.lzma
gzip -c9 Packages > Packages.gz
zstd -c19 Packages > Packages.zst

grep -E "Origin:|Label:|Suite:|Version:|Codename:|Architectures:|Components:|Description:" Release > Base
apt-ftparchive release . > Release
cat Base Release > out && mv out Release

# This line below will ask for your password defined on step 2. Replace what's on <> with the right value, it should look like step 4.
gpg -abs -u <blue-arrow-value> -o Release.gpg Release
```
(If you are hosting your repo on GitHub Pages, you can use GitHub Actions [like that](https://github.com/RedenticDev/Repo/blob/main/.github/workflows/automations.yml#L12-L87))

Some info:
- This script requires `zstd`, that can be installed using `brew install zstd` or `sudo apt install zstd`
	- Same thing for `xz` and `lzma` (both bundled in `xz` brew package)
- macOS users:
	- May need to use the GNU versions of `md5sum` and `sha256sum`.
	- **Do need** [Procursus fork of apt-ftparchive](https://apt.procurs.us/apt-ftparchive) as macOS doesn't features `apt`. You may need to add '`./`' before each `apt-ftparchive` in the script.
- What is does:
	- Generates Packages with hashes from the debs/ folder, change this if your debs are not in debs/
	- Compresses it with xz, gzip, zstd, bzip2, and lzma, this is for the widest compatibility. Change if you don't see the need for it.
	- Saves temporarily key values of Release file in Base file
	- Generates hashes and date of Packages files in the Release file
	- Adds the content of Base _then_ Release in Release
		- Note that you shouldn't push Base file, it's temporary. Remove it by adding `rm Base` in the script or don't push it by doing `git add Packages* Release*` instead of `git add .`.
	- Signs Release in Release.gpg

## Conclusion
8. **Upload your new changes!**. The files changed by the above shell script can be uploaded to your repo server. To your users, nothing will have changed and the repo will still work as usual. For your GPG key, you can put it somewhere public, like the README of a GitHub Pages repository, the main page for a private one, or another location where your users can easily get at. If you want to include your key in your README in a GitHub repo, you can drag it into the README while editing it _from the web editor_. I've been working closely with the Procursus Team to add keys to their keyring, so in a future update of Sileo adding your repo automatically installs your keyring (or users can manually install it), and you can email me at `me@crystall1ne.dev` to see about getting your keys onto Procursus.
