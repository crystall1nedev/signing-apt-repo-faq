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

## Making use of the key

For our next act, we need to make APT release when our repo updated, and verify the shas of our Packages.\* files. Here, I've copied a snippet from my repository (a bash shell script):
```bash
rm -rf Packages*
dpkg-scanpackages --multiversion debs > Packages
gzip -c9 Packages > Packages.gz
xz -c9 Packages > Packages.xz
xz -5fkev --format=lzma Packages > Packages.lzma
zstd -c19 Packages > Packages.zst
bzip2 -c9 Packages > Packages.bz2
packages_size=$(ls -l Packages | awk '{print $5,$9}')
packages_md5=$(md5sum Packages | awk '{print $1}')
packages_sha256=$(sha256sum Packages | awk '{print $1}')
packagesgz_size=$(ls -l Packages.gz | awk '{print $5,$9}')
packagesgz_md5=$(md5sum Packages.gz | awk '{print $1}')
packagesgz_sha256=$(sha256sum Packages.gz | awk '{print $1}')
packagesbz2_size=$(ls -l Packages.bz2 | awk '{print $5,$9}')
packagesbz2_md5=$(md5sum Packages.bz2 | awk '{print $1}')
packagesbz2_sha256=$(sha256sum Packages.bz2 | awk '{print $1}')
packagesxz_size=$(ls -l Packages.xz | awk '{print $5,$9}')
packagesxz_md5=$(md5sum Packages.xz | awk '{print $1}')
packagesxz_sha256=$(sha256sum Packages.xz | awk '{print $1}')
packageszst_size=$(ls -l Packages.zst | awk '{print $5,$9}')
packageszst_md5=$(md5sum Packages.zst | awk '{print $1}')
packageszst_sha256=$(sha256sum Packages.zst | awk '{print $1}')
date=$(date -R -u)
cp Base Release
sed -i "s/Date: .*/Date: $date/" Release
echo "MD5Sum:" >> Release
echo " $packages_md5 $packages_size" >> Release
echo " $packagesgz_md5 $packagesgz_size" >> Release
echo " $packagesbz2_md5 $packagesbz2_size" >> Release
echo " $packagesxz_md5 $packagesxz_size" >> Release
echo " $packageszst_md5 $packageszst_size" >> Release
echo "SHA256:" >> Release
echo " $packages_sha256 $packages_size" >> Release
echo " $packagesgz_sha256 $packagesgz_size" >> Release
echo " $packagesbz2_sha256 $packagesbz2_size" >> Release
echo " $packagesxz_sha256 $packagesxz_size" >> Release
echo " $packageszst_sha256 $packageszst_size" >> Release
```
*NOTE:* macOS users may need to use the GNU versions of md5sum and sha256sum.

1. **Rename Release to Base.** Don't panic--if you run the above shell script it will generate the Release file every time you run it. You can also append this script to your repo.sh, update.sh, or whatever you use (if you have one).

2. **Run the script.** A summary of what it does:
   - Generates Packages from the debs/ folder, change this if your debs are not in debs/
   - Compresses it with xz, gzip, zstd, bzip2, and lzma, this is for the widest compatibility. Change if you don't see the need for it.
   - Calculates the UTC Date.
   - Calculates the SHA256 and MD5's of the Packages file, and its compressed variants.
   - Copies the Base template over `Release`.
   - Imports the SHA256s, MD5s, and Date into the fresh Release file, using sed
       - macOS users should use gsed from Homebrew or Procursus, or modify the above to use the default sed.

3. **Upload your new changes!**. The files changed by the above shell script can be uploaded to your repo server. To your users, nothing will have changed and the repo will still work as usual. For your GPG key, you can put it somewhere public, like the README of a GitHub Pages repository, the main page for a private one, or another location where your users can easily get at. I've been working closely with the Procursus Team to add keys to their keyring, so in a future update of Sileo adding your repo automatically installs your keyring (or users can manually install it), and you can email me at `tunnic.adam@gmail.com` to see about getting your keys onto Procursus.
