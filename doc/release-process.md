Release Process
====================

Before every release candidate:

* Update translations (ping wumpus on IRC) see [translation_process.md](https://github.com/bitcoin/bitcoin/blob/master/doc/translation_process.md#synchronising-translations).

* Update manpages, see [gen-manpages.sh](https://github.com/upsidedoge-project/upsidedoge/blob/master/contrib/devtools/README.md#gen-manpagessh).
* Update release candidate version in `configure.ac` (`CLIENT_VERSION_RC`)

Before every minor and major release:

* Update [bips.md](bips.md) to account for changes since the last release.
* Update version in `configure.ac` (don't forget to set `CLIENT_VERSION_IS_RELEASE` to `true`) (don't forget to set `CLIENT_VERSION_RC` to `0`)
* Write release notes (see below)
* Update `src/chainparams.cpp` nMinimumChainWork with information from the getblockchaininfo rpc.
* Update `src/chainparams.cpp` defaultAssumeValid with information from the getblockhash rpc.
  - The selected value must not be orphaned so it may be useful to set the value two blocks back from the tip.
  - Testnet should be set some tens of thousands back from the tip due to reorgs there.
  - This update should be reviewed with a reindex-chainstate with assumevalid=0 to catch any defect
     that causes rejection of blocks in the past history.

Before every major release:

* Update hardcoded [seeds](/contrib/seeds/README.md), see [this pull request](https://github.com/bitcoin/bitcoin/pull/7415) for an example.
* Update [`src/chainparams.cpp`](/src/chainparams.cpp) m_assumed_blockchain_size and m_assumed_chain_state_size with the current size plus some overhead.
* Update `src/chainparams.cpp` chainTxData with statistics about the transaction count and rate. Use the output of the RPC `getchaintxstats`, see
  [this pull request](https://github.com/bitcoin/bitcoin/pull/12270) for an example. Reviewers can verify the results by running `getchaintxstats <window_block_count> <window_last_block_hash>` with the `window_block_count` and `window_last_block_hash` from your output.
* Update version of `contrib/gitian-descriptors/*.yml`: usually one'd want to do this on master after branching off the release - but be sure to at least do it before a new major release

### First time / New builders

If you're using the automated script (found in [contrib/gitian-build.py](/contrib/gitian-build.py)), then at this point you should run it with the "--setup" command. Otherwise ignore this.

Check out the source code in the following directory hierarchy.

    cd /path/to/your/toplevel/build
    git clone https://github.com/upsidedoge-project/gitian.sigs.updog.git
    git clone https://github.com/upsidedoge-project/upsidedoge-detached-sigs.git
    git clone https://github.com/devrandom/gitian-builder.git
    git clone https://github.com/upsidedoge-project/upsidedoge.git

### UpsideDoge maintainers/release engineers, suggestion for writing release notes

Write release notes. git shortlog helps a lot, for example:

    git shortlog --no-merges v(current version, e.g. 0.7.2)..v(new version, e.g. 0.8.0)

(or ping @wumpus on IRC, he has specific tooling to generate the list of merged pulls
and sort them into categories based on labels)

Generate list of authors:

    git log --format='- %aN' v(current version, e.g. 0.16.0)..v(new version, e.g. 0.16.1) | sort -fiu

Tag version (or release candidate) in git

    git tag -s v(new version, e.g. 0.8.0)

### Setup and perform Gitian builds

If you're using the automated script (found in [contrib/gitian-build.py](/contrib/gitian-build.py)), then at this point you should run it with the "--build" command. Otherwise ignore this.

Setup Gitian descriptors:

    pushd ./upsidedoge
    export SIGNER="(your Gitian key, ie bluematt, sipa, etc)"
    export VERSION=(new version, e.g. 0.8.0)
    git fetch
    git checkout v${VERSION}
    popd

Ensure your gitian.sigs.updog are up-to-date if you wish to gverify your builds against other Gitian signatures.

    pushd ./gitian.sigs.updog
    git pull
    popd

Ensure gitian-builder is up-to-date:

    pushd ./gitian-builder
    git pull
    popd

### Fetch and create inputs: (first time, or when dependency versions change)

    pushd ./gitian-builder
    mkdir -p inputs
    wget -P inputs https://bitcoincore.org/cfields/osslsigncode-Backports-to-1.7.1.patch
    echo 'a8c4e9cafba922f89de0df1f2152e7be286aba73f78505169bc351a7938dd911 inputs/osslsigncode-Backports-to-1.7.1.patch' | sha256sum -c
    wget -P inputs https://downloads.sourceforge.net/project/osslsigncode/osslsigncode/osslsigncode-1.7.1.tar.gz
    echo 'f9a8cdb38b9c309326764ebc937cba1523a3a751a7ab05df3ecc99d18ae466c9 inputs/osslsigncode-1.7.1.tar.gz' | sha256sum -c
    popd

Create the macOS SDK tarball, see the [macOS build instructions](build-osx.md#deterministic-macos-dmg-notes) for details, and copy it into the inputs directory.

### Optional: Seed the Gitian sources cache and offline git repositories

NOTE: Gitian is sometimes unable to download files. If you have errors, try the step below.

By default, Gitian will fetch source files as needed. To cache them ahead of time, make sure you have checked out the tag you want to build in upsidedoge, then:

    pushd ./gitian-builder
    make -C ../upsidedoge/depends download SOURCES_PATH=`pwd`/cache/common
    popd

Only missing files will be fetched, so this is safe to re-run for each build.

NOTE: Offline builds must use the --url flag to ensure Gitian fetches only from local URLs. For example:

    pushd ./gitian-builder
    ./bin/gbuild --url upsidedoge=/path/to/upsidedoge,signature=/path/to/sigs {rest of arguments}
    popd

The gbuild invocations below <b>DO NOT DO THIS</b> by default.

### Build and sign UpsideDoge Core for Linux, Windows, and macOS:

    export GITIAN_THREADS=2
    export GITIAN_MEMORY=3000
    
    pushd ./gitian-builder
    ./bin/gbuild --num-make $GITIAN_THREADS --memory $GITIAN_MEMORY --commit upsidedoge=v${VERSION} ../upsidedoge/contrib/gitian-descriptors/gitian-linux.yml
    ./bin/gsign --signer "$SIGNER" --release ${VERSION}-linux --destination ../gitian.sigs.updog/ ../upsidedoge/contrib/gitian-descriptors/gitian-linux.yml
    mv build/out/upsidedoge-*.tar.gz build/out/src/upsidedoge-*.tar.gz ../

    ./bin/gbuild --num-make $GITIAN_THREADS --memory $GITIAN_MEMORY --commit upsidedoge=v${VERSION} ../upsidedoge/contrib/gitian-descriptors/gitian-win.yml
    ./bin/gsign --signer "$SIGNER" --release ${VERSION}-win-unsigned --destination ../gitian.sigs.updog/ ../upsidedoge/contrib/gitian-descriptors/gitian-win.yml
    mv build/out/upsidedoge-*-win-unsigned.tar.gz inputs/upsidedoge-win-unsigned.tar.gz
    mv build/out/upsidedoge-*.zip build/out/upsidedoge-*.exe ../

    ./bin/gbuild --num-make $GITIAN_THREADS --memory $GITIAN_MEMORY --commit upsidedoge=v${VERSION} ../upsidedoge/contrib/gitian-descriptors/gitian-osx.yml
    ./bin/gsign --signer "$SIGNER" --release ${VERSION}-osx-unsigned --destination ../gitian.sigs.updog/ ../upsidedoge/contrib/gitian-descriptors/gitian-osx.yml
    mv build/out/upsidedoge-*-osx-unsigned.tar.gz inputs/upsidedoge-osx-unsigned.tar.gz
    mv build/out/upsidedoge-*.tar.gz build/out/upsidedoge-*.dmg ../
    popd

Build output expected:

  1. source tarball (`upsidedoge-${VERSION}.tar.gz`)
  2. linux 32-bit and 64-bit dist tarballs (`upsidedoge-${VERSION}-linux[32|64].tar.gz`)
  3. windows 32-bit and 64-bit unsigned installers and dist zips (`upsidedoge-${VERSION}-win[32|64]-setup-unsigned.exe`, `upsidedoge-${VERSION}-win[32|64].zip`)
  4. macOS unsigned installer and dist tarball (`upsidedoge-${VERSION}-osx-unsigned.dmg`, `upsidedoge-${VERSION}-osx64.tar.gz`)
  5. Gitian signatures (in `gitian.sigs.updog/${VERSION}-<linux|{win,osx}-unsigned>/(your Gitian key)/`)

### Verify other gitian builders signatures to your own. (Optional)

Add other gitian builders keys to your gpg keyring, and/or refresh keys: See `../upsidedoge/contrib/gitian-keys/README.md`.

Verify the signatures

    pushd ./gitian-builder
    ./bin/gverify -v -d ../gitian.sigs.updog/ -r ${VERSION}-linux ../upsidedoge/contrib/gitian-descriptors/gitian-linux.yml
    ./bin/gverify -v -d ../gitian.sigs.updog/ -r ${VERSION}-win-unsigned ../upsidedoge/contrib/gitian-descriptors/gitian-win.yml
    ./bin/gverify -v -d ../gitian.sigs.updog/ -r ${VERSION}-osx-unsigned ../upsidedoge/contrib/gitian-descriptors/gitian-osx.yml
    popd

### Next steps:

Commit your signature to gitian.sigs.updog:

    pushd gitian.sigs.updog
    git add ${VERSION}-linux/"${SIGNER}"
    git add ${VERSION}-win-unsigned/"${SIGNER}"
    git add ${VERSION}-osx-unsigned/"${SIGNER}"
    git commit -m "Add ${VERSION} unsigned sigs for ${SIGNER}"
    git push  # Assuming you can push to the gitian.sigs tree
    popd

Codesigner only: Create Windows/macOS detached signatures:
- Only one person handles codesigning. Everyone else should skip to the next step.
- Only once the Windows/macOS builds each have 3 matching signatures may they be signed with their respective release keys.

Codesigner only: Sign the macOS binary:

    transfer upsidedoge-osx-unsigned.tar.gz to macOS for signing
    tar xf upsidedoge-osx-unsigned.tar.gz
    ./detached-sig-create.sh -s "Key ID"
    Enter the keychain password and authorize the signature
    Move signature-osx.tar.gz back to the gitian host

Codesigner only: Sign the windows binaries:

    tar xf upsidedoge-win-unsigned.tar.gz
    ./detached-sig-create.sh -key /path/to/codesign.key
    Enter the passphrase for the key when prompted
    signature-win.tar.gz will be created

Codesigner only: Commit the detached codesign payloads:

    cd ~/upsidedoge-detached-sigs
    checkout the appropriate branch for this release series
    rm -rf *
    tar xf signature-osx.tar.gz
    tar xf signature-win.tar.gz
    git add -a
    git commit -m "point to ${VERSION}"
    git tag -s v${VERSION} HEAD
    git push the current branch and new tag

Non-codesigners: wait for Windows/macOS detached signatures:

- Once the Windows/macOS builds each have 3 matching signatures, they will be signed with their respective release keys.
- Detached signatures will then be committed to the [upsidedoge-detached-sigs](https://github.com/upsidedoge-project/upsidedoge-detached-sigs) repository, which can be combined with the unsigned apps to create signed binaries.

Create (and optionally verify) the signed macOS binary:

    pushd ./gitian-builder
    ./bin/gbuild -i --commit signature=v${VERSION} ../upsidedoge/contrib/gitian-descriptors/gitian-osx-signer.yml
    ./bin/gsign --signer "$SIGNER" --release ${VERSION}-osx-signed --destination ../gitian.sigs.updog/ ../upsidedoge/contrib/gitian-descriptors/gitian-osx-signer.yml
    ./bin/gverify -v -d ../gitian.sigs.updog/ -r ${VERSION}-osx-signed ../upsidedoge/contrib/gitian-descriptors/gitian-osx-signer.yml
    mv build/out/upsidedoge-osx-signed.dmg ../upsidedoge-${VERSION}-osx.dmg
    popd

Create (and optionally verify) the signed Windows binaries:

    pushd ./gitian-builder
    ./bin/gbuild -i --commit signature=v${VERSION} ../upsidedoge/contrib/gitian-descriptors/gitian-win-signer.yml
    ./bin/gsign --signer "$SIGNER" --release ${VERSION}-win-signed --destination ../gitian.sigs.updog/ ../upsidedoge/contrib/gitian-descriptors/gitian-win-signer.yml
    ./bin/gverify -v -d ../gitian.sigs.updog/ -r ${VERSION}-win-signed ../upsidedoge/contrib/gitian-descriptors/gitian-win-signer.yml
    mv build/out/upsidedoge-*win64-setup.exe ../upsidedoge-${VERSION}-win64-setup.exe
    mv build/out/upsidedoge-*win32-setup.exe ../upsidedoge-${VERSION}-win32-setup.exe
    popd

Commit your signature for the signed macOS/Windows binaries:

    pushd gitian.sigs.updog
    git add ${VERSION}-osx-signed/"${SIGNER}"
    git add ${VERSION}-win-signed/"${SIGNER}"
    git commit -a
    git push  # Assuming you can push to the gitian.sigs.updog tree
    popd

### After 3 or more people have gitian-built and their results match:

- Create `SHA256SUMS.asc` for the builds, and GPG-sign it:

```bash
sha256sum * > SHA256SUMS
```

The list of files should be:
```
upsidedoge-${VERSION}-aarch64-linux-gnu.tar.gz
upsidedoge-${VERSION}-arm-linux-gnueabihf.tar.gz
upsidedoge-${VERSION}-i686-pc-linux-gnu.tar.gz
upsidedoge-${VERSION}-x86_64-linux-gnu.tar.gz
upsidedoge-${VERSION}-osx64.tar.gz
upsidedoge-${VERSION}-osx.dmg
upsidedoge-${VERSION}.tar.gz
upsidedoge-${VERSION}-win32-setup.exe
upsidedoge-${VERSION}-win32.zip
upsidedoge-${VERSION}-win64-setup.exe
upsidedoge-${VERSION}-win64.zip
```
The `*-debug*` files generated by the gitian build contain debug symbols
for troubleshooting by developers. It is assumed that anyone that is interested
in debugging can run gitian to generate the files for themselves. To avoid
end-user confusion about which file to pick, as well as save storage
space *do not upload these to the upsidedoge.org server, nor put them in the torrent*.

- GPG-sign it, delete the unsigned file:
```
gpg --digest-algo sha256 --clearsign SHA256SUMS # outputs SHA256SUMS.asc
rm SHA256SUMS
```
(the digest algorithm is forced to sha256 to avoid confusion of the `Hash:` header that GPG adds with the SHA256 used for the files)
Note: check that SHA256SUMS itself doesn't end up in SHA256SUMS, which is a spurious/nonsensical entry.

- Upload zips and installers, as well as `SHA256SUMS.asc` from last step, to the upsidedoge.org server.

```
- Update upsidedoge.org version

- Update other repositories and websites for new version

- Announce the release:

  - upsidedoge-dev mailing list

  - blog.upsidedoge.org blog post

  - Update title of #upsidedoge and #upsidedoge-dev on Freenode IRC

  - Optionally twitter, reddit /r/UpsideDoge, ... but this will usually sort out itself

  - Archive release notes for the new version to `doc/release-notes/` (branch `master` and branch of the release)

  - Create a [new GitHub release](https://github.com/upsidedoge-project/upsidedoge/releases/new) with a link to the archived release notes.

  - Celebrate
