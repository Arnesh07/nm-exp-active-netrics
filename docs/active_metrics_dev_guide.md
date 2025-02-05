# Extending or adding tests to nm-exp-active-netrics

## Developing code locally on a Raspberry Pi

The best place to start developing new code for the `nm-exp-active-netrics` package is on a
Raspberry Pi device that you have connected to your home network.

If you need a Raspberry Pi device to use for development, email mtrichardson@uchicago.edu.
Follow the steps below to develop code locally.

1. SSH into your local device or a device that you have set up in the lab:
   ```
   ssh ubuntu@{device_ip_address}
   ```
   Ask a member of the team for the password.
   If you are using a "managed" device, this _should_ also show up as `netrics.local`.
   If that does not work or you are using an unmanaged devce, you'll have to do some `nmap` sleuthing.
   
   * If developing on a device at home, you will need to be connected to your home network to ssh to the device.
   * Developing in the IoT lab is not recommended, because we have measurements running on those devices.
     However, you can _do_ testing on that device. You will need to first `ssh` to tigerteam and then `ssh` to the testing device in the lab using tigerteam's auxiliary ethernet interface. Ask us about IP addresses.
2. Next, pull the `nm-exp-active-netrics` source code from github by your preferred method.
   This repo is public and does not require special access to pull, but you'll have to ask Marc (mtrichardson) for commit rights.
3. After pulling the `nm-exp-active-netrics` repository from github, create a new branch for your test. You should name the branch based on the test that you are developing. So, for example, if you are developing a new test for encrypted dns, you could name the branch `add-encrypted-dns`.
4. If you need to add a binary for your test that you _cannot_ install via apt-get, you must:

      (a) Create a script that installs the binary and adds the directory housing the binary to $PATH. Add this script to the `/scripts` directory using the naming convention `make_{your_binary}.sh`. See the script [here](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/main/scripts/make_oplat.sh) for an example.

      (b) After creating the script to install the binary and add it to $PATH, edit the [Makefile](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/main/Makefile) to include your binary, as follows:
      ```
      {your_binary}:
               ./scripts/make_{your_binary}.sh
      ```

      (c) Add your new make command to the `help` [section](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/main/Makefile#L7) of the Makefile as an additional step for installation.

      (d) Run the command `make_{your_binary}.sh`.
   
   _Note_: Dependencies that you can install through apt-get can be added to the Makefile under the `deps` [section](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/main/Makefile#L34) of the file. There is no need to do anything else additional for dependencies installed through apt-get. (That is, the above steps for creating a make script and adding it to the makefile are unnecessary.) Once you've added the new dependency, you will need to run `sudo make deps` to install your new dependency.

   If you need to add a new Python library, add your library and version to the requirements.txt [file](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/main/requirements.txt) and rerun `sudo make venv` to install the new package in the virtual environment.
5. Most development for this package involves three files:
   * **[netrics.py](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/main/src/netrics.py)**: You need to
     (a) add arguments to the [parser](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/9ff6291c783f93c30dd6eaf1855ed19c5a71845f/src/netrics.py#L51)
     (b) ensure that, later on, the test actually runs.  Note that if there is no connectivity / the device is offline,
         then you or we may not want the test to run.  If the code takes too long to attempt, it can muck up other tests
         that we _do_ want to maintain on schedule.
   * **[netson.py](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/main/src/netrics/netson.py)**:
     Here, you must create a function that actually runs your test.  Have a look at
     [ping_latency](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/9ff6291c783f93c30dd6eaf1855ed19c5a71845f/src/netrics/netson.py#L309)
     for inspiration.
     Most of these are simply calling a command line tool and parsing its output. 
     The functions set two things:

     (a) The raw output of the tool is returned, and passed into a dictionary, that gets zipped up, so that we retain all raw files.

     (b) The processed output is loaded into the `self.results` dictionary, which is shared across all tests in that instance,
         and dumped as a json, for upload by the management code.

   * **[nm-exp-active-netrics.toml](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/main/conf/nm-exp-active-netrics.toml)**:
     This file controls all configurations that you may need for your tests.  For example, 
     [reference_site_dict](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/83b135f528dce3eaa48a54eb9800b85e4092b096/conf/nm-exp-active-netrics.toml#L32)
     defines the list of `ping` destinations.
     If, for instance, you wish to test encrypted DNS, you may want to specify your targets; 
     if you're checking page loads, you'll list your pages.
     You will also need to edit the `cron` [schedule](https://github.com/chicago-cdac/nm-exp-active-netrics/blob/83b135f528dce3eaa48a54eb9800b85e4092b096/conf/nm-exp-active-netrics.toml#L20)
     so that your test runs at all.
     Ultimately, we recommend adding a line like the following:
     ```
     {MY_NEW_TEST_SCHEDULE} netrics env USER=$LOGNAME /usr/local/bin/netrics --my_new_test >{log} 2>&1
     ```
6. Once you have your changes, you **must** (!!) test locally on the device where you are developing. You should test one-off, edge-cases, and continuous performance.

   For initial tests, you may want to just do `./netrics --your_test`.
   If all you've changed is `netrics.py` and `netson.py`, I would recommend rebuilding
   (see `make -h` and probably `make install`)
   or simply copy the two files to the installed version (if you're using the managed install), at
   ```
   /usr/local/src/nm-exp-active-netrics/src/
   ```
   You could also directly edit the cron file, 
   ```
   /etc/cron.d/cron-nm-exp-active-netrics
   ```
   to run a very vigorous test schedule.
   Once this has run for 24-48 hours on your device as expected, move to the next step. You can monitor performance via logs at 
   ```
   /var/nm/nm-exp-active-netrics/log
   ```
   and via your grafana dashboard.
7. Commit your code (via a pull request for the branch on which you are developing) and interface with the team for testing on the beta group.

## Deploying to beta testers

Your work may be done; ours continues.  Unless you are a member of the team, the links in the follow part of the document will not be accessible to you.

1. Merge the pull request branch with the main branch for `nm-exp-active-netrics`.
2. Build (`make build`) the active netrics package and copy it to `tigerteam.io:/scratch/databases/saltstack/saltstack1/srv/salt/files/dev/deb/`.
3. Each change to the cron job above in (3), i.e., `{MY_NEW_TEST_SCHEDULE}`,
   requires a change in _each_ of the management [templates](https://github.com/chicago-cdac/nm-mgmt-cms/tree/main/templates).
   For example, to _turn off_ the tests that you have just written, you would add a line like
   ```
   MY_NEW_TEST_SCHEDULE=#
   ```
   to the files like [ssx.template.conf](https://github.com/chicago-cdac/nm-mgmt-cms/blob/main/templates/default/ssx.template.conf).
   This will get replaced, by [this function](https://github.com/chicago-cdac/nm-mgmt-cms/blob/a30bf836ee298dc98b0ad7894132199fad8b80db/scripts/generate_pillar/generate_pillar.py#L87) 
   in `generate_pillar.py`.
4. Set the git and deb packages, and copy the toml into your directory, per [deploy.tigerteam.sh](https://github.com/chicago-cdac/nm-mgmt-cms/blob/main/deploy.tigerteam.sh#L13). 
5. Next, run `generate_pllar` (this is the last step of the `deploy.tigerteam.sh` script),
   and then verify that the files at
   ```
   /srv/salt/devices/dev/*/etc/nm-exp-active-metrics/nm-exp-active-netrics.toml
   ```
   reflect your intent.
6. Deploy to the devices, via [update_netrics.sh](https://github.com/chicago-cdac/nm-mgmt-cms/blob/main/scripts/update_netrics.sh).

