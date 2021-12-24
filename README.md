# Instructions on a Fresh install of Ubuntu 18 or 20:

## Local Package:

1. Go to the directory of the yaml file and execute `snapcraft --use-lxd`.
2. Next, trigger `snap install --devmode snap_name.snap`, where `snap_name.snap` is the file generated as a result of the above command.
3. Execute the command specified as the `name` value in `snapcraft.yaml`.

## Snapstore:

1. Put `snapcraft.yaml` into a GitHub repository
2. Register a new snap in snapcraft.io (`devmode`) having the `name` value in `snapcraft.yaml`.
3. Go to Builds, and connect the GitHub repo created in Step 1.
4. A build should start and the package will released under edge. In case your package is a stable one, we can push this manually from edge to stable using the Snapstore UI.

To install, test, and run a **stable** package from snapcraft.io

```
sudo snap install hapiserver
sudo hapiserver-test 
sudo hapiserver
```

Note : sudo was not required for installing other packages like sublime, VLC etc. Yet, hapi-server required me to use sudo.

To install, test, and run a **devmode** package from snapcraft.io

```
sudo snap install --edge snap-name --devmode 
sudo hapiserver.hapiserver-test 
sudo hapiserver
```

Note : sudo is needed again for the same above mentioned reasons.

# TODO

1. The package (devmode and devel) still needs sudo access to be installed.
2. The hard-wiring of path inside the commands needs to be fixed.

# Issues:

1. Initially deployed a package into the snapstore setting the `grade: devel` and `confinement: devmode`. The `logdir` and `metadir` needs to be outside of the /snap/snapname/current directory. Had to make few modifications to `test/server-test.js.` (Package having those changes : https://github.com/drunkenlord/snapp-happ/releases/download/v1.0.11/hapi-server-v1.0.11.tgz). Since this code change is done as part of packaging github code diff is not available. To be precide, I just had to add server_args = server_args + " --logdir="+argv.metadir + " --metadir="+argv.logdir to server-test.js file

I was able to run the tests as well as start the server as expected given the instructions above.

2. Next up, changed the settings to grade: stable and confinement: strict. Started facing the issue : `tar_extract_all() failed: Operation not permitted.`

   The forum thread for this issue : https://forum.snapcraft.io/t/tar-extract-all-failed-operation-not-permitted/27497

3. To find the root cause, went ahead and tried out testing with simple binaries of Node.js and Python. Created a simple snapcraft.yaml for each of the binary. For Python, the `yaml` file was:

   ```
   name: run-simplepy
   base: core18
   version: '1.0.0'
   summary: Simple Py
   description: |
      Simple Py
   grade: stable
   confinement: strict
   architectures:
      - build-on: amd64
      
   apps:
      runpyapp:
          command: /snap/run-simplepy/current/python-bin
          
   parts:
      cprog:
          source: https://github.com/drunkenlord/run-simplepy/releases/download/v1.0.0/py.tar.gz
          plugin: dump
   ```

The py.tar.gz contains the python binary. Usage: `run-simplepy.runpyapp sample.py`. 

The error still persisted. Tried out the same with Node.js binary and nothing changed. But  

```
cd /snap/run-simplepy/current
./python-bin sample.py`
```

works as expected. So the problem seems to appear only when we trigger it from outside the `/snap` dir.

4. Next up tried the pre-load method. The snapcraft.yaml will look something like this:

```
name: run-simp-py
base: core18
version: '1.0.0'
summary: Simple Py
description: |
  Simple Py
grade: stable
confinement: strict
architectures:
  - build-on: amd64


  apps:
  python:
    command: python


runhello:
     command: python

parts:
python:
    plugin: python
    python-version: python2 
    python-packages:
      - argparse==1.2.2
      
   ```

I was able to execute simple python file using `run-simp-py.runhello sample.py` successfully.  
Tried doing the same with Node.js as well. It seemed to work as well. Yet for Node.js, the interfaces had to be tweaked inorder to give additional access for the snaps confined to strict. 
These interfaces needs to be listed under plugs for each repective app. 
I had to add `system-files` interface inorder to get node up and working. 
Link to the complete YAML file : https://github.com/drunkenlord/run-simplepy/blob/master/snapcraft.yaml

5. Also, tried getting some information about the opened file descriptors when executing the `./python-bin sample.py` from inside the `/snap` inorder to see what the snap is using as "**tmp**" dir inorder to write the data to. I have used a bunch of commands and techniques. Yet, none of them were fruitful.

6. Since testing out the package in the strict mode has become cumbersome since each of the change required us to push new package version into snapstore after the successful build process, tried using 

> --dangerous flag

inorder to see if we can test the strict packages locally without deploying them to the snap store. The flag did not seem to work as issues related to authorization/signatures had started occuring.
 
7. Also faced an issue with manual revew of the packages in the strict mode. Usually, the packages confined to classic zone shall be reviewed manually before released into the snapstore, after the successful build process. Yet, seems like usage `system-files` interface flagged our packaged for the manual review even in the strict mode.

The Forum Thread for this Issue: https://forum.snapcraft.io/t/snap-is-being-queued-for-manual-review-in-strict-confinement/27756

Documentation Reference: https://snapcraft.io/docs/system-files-interface

Link for repo which has files and archives based on various experiments performed :  https://github.com/KSR4599/hapi-server-snapcraft-files-archive
TODO : Will create a readME.md for this repo as well.
