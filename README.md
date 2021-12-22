#### Instructions on a Fresh install of Ubuntu 18 or 20 :

##### Local Package :

1. Go to the directory where yaml file is present, and execute `snapcraft --use-lxd` 
2. Next, trigger `snap install --devmode snap_name.snap`
3. Execute the command specified in your snapcraft.yaml


Snapstore:

1. Create a repo and push your snapcraft.yaml into that repo
2. Register a new snap in snapcraft.io (dev-mode) having the name matching with that of the one present in the snapcraft.yaml
3. Go to Builds, and connect the github repo we have created in Step -1
4. Build shall start and newer version of package shall be released under edge. In case your package is a stable one, we can push this manually from edge to stable using the UI.
5. To install the stable package present in the snap-store, `sudo snap install snap-name`
6. If the package is present in the devmode, we should something like `snap install --edge snap-name --devmode`


### Key observations over past two months:

1. Initially deployed a package into the snapstore setting the grade: devel and confinement: devmode. The `logdir` and `metadir` needs to be outside of the /snap/snapname/current directory. Had to tweak few changes in `test/server-test.js.` I was able to run the tests as well as start the server as expected.
The snapraft.yaml associated this setup :

Link : [Click Here](https://github.com/KSR4599/hapi-server-snap/blob/master/snapcraft.yaml)
Usage : 

    Installation : `sudo snap install --edge hapiserver --devmode`
    For Starting Server : `sudo hapiserver`
    For Running Tests : `sudo hapiserver.hapiserver-test`

2. Next up, changed the settings to grade: stable and confinement: strict. Started facing the issue : `tar_extract_all() failed: Operation not permitted.`

The forum thread for this issue : https://forum.snapcraft.io/t/tar-extract-all-failed-operation-not-permitted/27497

3. To find the root cause, went ahead and tried out testing with simple binaries of Node.js and Python. Created a simple snapcraft.yaml for each of the binary. Looks something like this :


 

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
            source: https://github.com/**/run-simplepy/releases/download/v1.0.0/py.tar.gz
            plugin: dump


The py.tar.gz contains the python binary. Usage : `run-simplepy.runpyapp sample.py`. 

The error still persisted. Tried out the same with Node.js binary and nothing changed. But when if I cd to 

> /snap/run-simplepy/current

, I can execute .`/python-bin sample.py` as expected. So the problem seems to appear only when we trigger it from outside the /snap dir.

4. Next up tried the pre-load method. The snapcraft.yaml shall have the following parts :

       python:
	        plugin: python
	       python-version: python2 

  And following app:

    python:
	    command: python
    runhello:
	     command: python


I was able to execute simple python file using `snap_name.runhello sample.py` successfully.  Tried doing the same with Node.js as well. It seemed to work as well. Yet for Node.js, the interfaces had to be tweaked inorder to give additional access for the snaps confined to strict. These interfaces needs to be listed under plugs for each repective app. I had to add `system-files` interface inorder to get node up and working.

5. Also, tried getting some information about the opened file descriptors when executing the `./python-bin sample.py` from inside the `/snap` inorder to see what the snap is using as "**tmp**" dir inorder to write the data to. I have used a bunch of commands and techniques. Yet, none of them were fruitful.

6. Since testing out the package in the strict mode has become cumbersome since each of the change required us to push new package version into snapstore after the successful build process, tried using 

> --dangerous flag

 inorder to see if we can test the strict packages locally without deploying them to the snap store. The flag did not seem to work as issues related to authorization/signatures had started occuring.
 
7. Also faced an issue with manual revew of the packages in the strict mode. Usually, the packages confined to classic zone shall be reviewed manually before released into the snapstore, after the successful build process. Yet, seems like usage `system-files` interface flagged our packaged for the manual review even in the strict mode.

The Forum Thread for this Issue : https://forum.snapcraft.io/t/snap-is-being-queued-for-manual-review-in-strict-confinement/27756

Documentation Reference : https://snapcraft.io/docs/system-files-interface

Todo :

The package (devmode and devel) still needs sudo access to be installed. The hard-wiring of path inside the commands needs to be fixed.




