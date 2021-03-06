Release Process for Tor Messenger
=================================

You are ready to release Tor Messenger when you have performed a substantial
development effort, or you have to patch a security issue.

The release process is divided into two parts: building Tor Messenger, and
then signing the MAR files for secure automatic updates; follow this guide
step by step to complete both the steps.

Building
========

- If not already done, bump the version number in `ChangeLog`, `rbm.conf`,
  `tools/update-responses/config.yml`

- Ensure `HEAD` on the build machines matches the `master` of
  `tor-messenger-build.git` repository.

- Run `make fetch` before the next step.

- Run `make tor-messenger-release`. The builds will be in the
  `release/$VERSION` directory, along with the MAR files. This will also
  output the `sha256sum` of the files.

- Compare the Linux builds with at least one other person -- preferably
  building on another machine -- to check if they are reproducible.

- Test the builds on all platforms:

    - Create an XMPP account and IRC account

        - Ensure that the corresponding OTR keys have been generated (Tools >
          OTR Preferences > Private Keys) 

        - Initiate an OTR conversation with another instance. Verify the
          various authentication mechanisms.

- If everything is fine, send the Windows and macOS builds (EXE and DMG) to
  the Tor Browser team for code signing.

**Wait to get the signed EXE and DMG back before proceeding to the next step**

Making Update MARs
==================

This step only works if you want to do an update build and have a base version
to diff against. So if you are upgrading from A to B, follow these steps; this
assumes that A is the older version and B is the newer one.

**These steps are not required if you are doing a build for just A or B**

- Navigate to the build directory on the build machine

- `cd tor-messenger-build/release/ && mkdir -p tor-messenger/signed/$VERSION`
  (where $VERSION is the version of the release, same as in rbm.conf).

- You are now in the tor-messenger/signed/$VERSION/ directory

- Copy all the release files from tor-messenger-release into this directory

        cp -r ../../../$VERSION/* .

  **If this is the first time you are doing an update, make sure the older
  version is also present in the signed/ directory. For example if you are
  building B for the first time, you should have A/ in the signed/ directory.
  Repeat the above steps for A (if not already done):**

        mkdir -p tor-messenger/signed/A
        cd A/
        cp -r ../../../A/* .

  **The `gen_incrementals' script will complain about this.**

- Now copy the SIGNED Windows and macOS EXE and DMG files which the Tor
  Browser team has uploaded.

    - Check this step again to ensure you have the signed binaries. Compare
      the checksums!

- At this stage, you have the code signed binaries and the unsigned complete
  MARs.

- Now go to `tor-messenger-build/tools/update-responses`.

- Edit `config.yml`:
    - Check and update the version in the `channel` section
    - In `version`, add a new section (see existing sections for help)
      corresponding to the release version
    - Assume you are updating from A to B. Your `config.yml` should look like:

            channels:
                release: B
            B:
                platformVersion: 52.2.0
                detailsURL: https://blog.torproject.org/blog/tor-messenger-B-released
                incremental_from:
                  - A

    **Increment platformVersion if it has changed
    Update detailsURL which will point to the blog post detailing the release**

- Now run `./gen_incrementals`. In the `signed/$VERSION` directory, you should
  see incremental MARs from A->B along with the existing complete MARs.

- This completes the MAR generation step. The next step is signing, which
  takes place offline.

Signing Update MARs
===================

This step has to be performed offline and assumes you have the MAR signing
certificates and the private keys.

**DO NOT copy the certificate directory to ANY remote machine**

**This step works only on a Linux machine (32-bit or 64-bit)**

Offline Steps Start
-------------------

- Create a new local directory

- Copy `signmars.py` from the build repository (`tor-messenger/tools/update-responses/`)

- Copy the following files from the signed/ directory in the previous section
  to the current local directory:

        mar-tools-linux*.zip
        *.mar

        scp tor-messenger-build/release/tor-messenger/signed/$VERSION/{*.mar,mar-tools-linux*.zip} .

- Set `NSS_DB_DIR` to point to the directory with the certificate files. You
  should point to the directory with the `cert8.db` file.

- Run `signmars.py` and follow the steps. The signed MARs will be in the
  signed/ directory.

- `cd signed/`

- Upload the signed MARs back to the directory you copied them from.

        scp *.mar tor-messenger-build/release/tor-messenger/signed/$VERSION/

Offline Steps End
-----------------

- Back to the build machine: navigate to tools/update-responses/. Run
  `./update_responses` to generate the update manifest.

- Generate the checksums for the builds:

        sha256sum `ls -I "*.zip" -I "*.txt"` > sha256sums-signed-build.txt

- GPG sign the sha256sums-signed-build.txt file:

    - Copy the sha256sums-signed-build.txt to a local machine

            gpg -abs sha256sums-signed-build.txt

    - Upload the signature (*.asc) back to build machine signed/ directory.

- At this stage, you have the code signed DMG and EXE, Linux builds, signed
  MAR files, the update information, and the signed sha256sum of all files.

Testing Updates
===============

Before we push the update to users, we should test them first to make sure
that incremental (or complete updates) are working as intended. We do this by
pushing the updates to the `update_2.test` directory instead of `update_2`.

- Copy the `htdocs/release` directory from the last section to `aus2.torproject.org`

        staticiforme.torproject.org:/srv/aus2-master.torproject.org/htdocs/tormessenger/update_2.test/release

**Make sure the .htaccess file is copied as well.**

- ssh to `staticiforme.torproject.org:/srv/dist-master.torproject.org/htdocs/tormessenger`
  `mkdir $VERSION` to create the directory for the new version.

- Copy the contents* of the signed/ directory (MAR files) to dist.tormessenger.org/tormessenger/$VERSION

        staticiforme.torproject.org:/srv/dist-master.torproject.org/htdocs/tormessenger/$VERSION

  * - You can skip sha256sums-unsigned-build.txt since we don't use it.

- ssh to `staticiforme.torproject.org:/srv/dist-master.torproject.org/htdocs/tormessenger`

    - Run `ln -sfn $VERSION current`
        This helps us ensure that the `current` directory always refers to the
        latest release of Tor Messenger

    - Exit

- We need to finalize the changes. Run:

        ssh staticiforme.torproject.org static-update-component dist.torproject.org && ssh staticiforme.torproject.org static-update-component aus2.torproject.org 

- Now test the updates on ALL platforms as it is possible that updates may
  work on one but fail on the other.

    - Start Tor Messenger

    - Open the preferences editor and copy the value for preference `app.update.url`

    - Create a new string preference (Right click -> New -> String) and set
      the name to `app.update.url.override`. Set the value copied from the
      previous step REPLACING `update_2` with `update_2.test`. Your string
      should be:

            https://aus2.torproject.org/tormessenger/update_2.test/%CHANNEL%/%BUILD_TARGET%/%VERSION%/%LOCALE% 

    - Create a new boolean preference `app.update.log` and set it to `true`

    - Force an update by going to the about screen

    - Tor Messenger should update (incrementally) and then restart

    - The update should be applied on restart. If not, it should complain and
      that means something is broken. Since we set `app.update.log` to `true`,
      it's a good time to look at the error console

If everything went on fine with the testing, move on to the next step.

Finalizing Updates and Releasing
================================

- Publish the blog post on blog.torproject.org. The URL should follow the same
  format as described in the `config.yml` file for $VERSION. This is usually
  done automatically by the blog platform.

- ssh to `staticiforme.torproject.org:/srv/aus2-master.torproject.org/htdocs/tormessenger/update_2/release`

- Delete the existing files and then copy the changes from the `test` directory:

        cp -R ../../update_2.test/release/. .

- The update is still not live. To finalize it, exit, and then run:

        ssh staticiforme.torproject.org static-update-component aus2.torproject.org 

- The update is now live. Now is a good time to again test if the updates are
  being properly pushed to the users! It may be a good idea to repeat the Tor
  Messenger tests in the previous section. (Install older version, force
  update, check.)

- Add the date to the ChangeLog file and commit it with the message `Release
  version $VERSION`.

- Finalize the release process by tagging the version in
  `tor-messenger-build.git` (run the code below). Make sure that HEAD is the
  commit which adds the release date to the ChangeLog.

        VERSION=`awk '/tormessenger_version/ {print $2}' rbm.conf | cut -d "'" -f2` 
        git tag -s v$VERSION -m "version $VERSION"

  Verify the tag and then push it to the remote:

        git push --tags

- This completes the release process.

After a Release
===============

- Bump up the version number in ChangeLog, rbm.conf,
  tools/update-responses/config.yml for the next release

- Increment the version number and update the links on
  https://trac.torproject.org/projects/tor/wiki/doc/TorMessenger

- Administer the comments on the blog and reply to them. Open relevant tickets
  wherever necessary.

Troubleshooting
===============

- If you want to update add-ons like ctypes-otr or tor-launcher, make sure to
  bump the version number in their `install.rdf` file. Add-ons are only
  updated if the version number is incremented.

- Any changes you make on staticiforme.torproject.org have to be finalized
  with the `static-update-component $DIR` command. So if you have made changes
  to `dist.torproject.org`, you have to run:

   ssh staticiforme.torproject.org static-update-component dist.torproject.org 

- Make sure `app.update.log` is set to `true` before testing updates since you
  will get logging information as the update is applied, and if it fails.
