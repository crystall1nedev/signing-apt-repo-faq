# How to sign an APT repository... correctly.

To prevent your users from possibly facing man-in-the-middle attacks and secure your repository, you should follow these steps. To get started, we'll need you to have a GPG key to sign with and export the public key (it's easier than you might think). I provide instructions for Linux and macOS because these are the two operating systems I use--this should be doable in WSL or Cygwin.

## Generating keys

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

6. **Verify your key works.** Upload Release.gpg to your repo's server, and move the exported key to `/etc/apt/trusted.gpg.d/` on iOS or `/opt/procursus/etc/apt/trusted.gpg.d` on macOS. If you're successful, you should only see errors about an incorrect Date and missing Hashes (we solve this in the next section). Getting `BADSIG`, `NO_PUBKEY`, or any other GPG related errors means that you should check again and possbily regenerate your keys. 

## Making use of the keys

7. **Generate and sign your files.** When you or your package manager downloads repo files, in this case Packages\*, Contents\* and Release, we need to certify that these files you download are exactly the ones you're supposed to get. It's possible to do so by checking the hash of a file, thanks to protocols like MD5, SHA1, SHA256...  
Each time you're doing a modification to your `deb`s, you'll need to run this script from your repo's root folder:
```sh
apt-ftparchive packages ./debs > Packages
bzip2 -c9 Packages > Packages.bz2
xz -c9 Packages > Packages.xz
xz -5fkev --format=lzma Packages > Packages.lzma
gzip -c9 Packages > Packages.gz
zstd -c19 Packages > Packages.zst

# While we are at it, also generate Contents files
apt-ftparchive contents ./debs > Contents-iphoneos-arm
bzip2 -c9 Contents-iphoneos-arm > Contents-iphoneos-arm.bz2
xz -c9 Contents-iphoneos-arm > Contents-iphoneos-arm.xz
xz -5fkev --format=lzma Contents-iphoneos-arm > Contents-iphoneos-arm.lzma
gzip -c9 Contents-iphoneos-arm > Contents-iphoneos-arm.gz
zstd -c19 Contents-iphoneos-arm > Contents-iphoneos-arm.zst

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
	- Generates Conetnts file which allows previewing package filenames before downloading
	- Also compresses it with xz, gzip, zstd, bzip2, and lzma, this is for the widest compatibility.
	- Saves temporarily key values of Release file in Base file
	- Generates hashes and date of Packages files in the Release file
	- Adds the content of Base _then_ Release in Release
		- Note that you shouldn't push Base file, it's temporary. Remove it by adding `rm Base` in the script or don't push it by doing `git add Packages* Conetnts* Release*` instead of `git add .`.
		- Even better, you could add it to .gitignore
	- Signs Release in Release.gpg

## Conclusion

8. **Upload your new changes!**. The files changed by the above shell script can be uploaded to your repo server. To your users, nothing will have changed and the repo will still work as usual. For your GPG key, you can put it somewhere public, like the README of a GitHub Pages repository, the main page for a private one, or another location where your users can easily get at. If you want to include your key in your README in a GitHub repo, you can drag it into the README while editing it _from the web editor_. I've been working closely with the Procursus Team to add keys to their keyring, so in a future update of Sileo adding your repo automatically installs your keyring (or users can manually install it), and you can email me at `tunnic.adam@gmail.com` to see about getting your keys onto Procursus.
