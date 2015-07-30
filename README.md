How to ACTUALLY Install edX
===========================

0 make sure your computer has at least 5 GB free space

1 download and install Vagrant (>= 1.6.5)

2 download and install VirtualBox (>= 4.3.12)

3 create the folder and config vagrant

```
mkdir devstack
cd devstack
curl -L https://raw.githubusercontent.com/edx/configuration/master/vagrant/release/devstack/Vagrantfile > Vagrantfile
vagrant plugin install vagrant-vbguest
vagrant up
```

4 wait for vagrant to download a 4 GB virtual machine "lavash-devstack"

5 vagrant is going to run a BUNCH of configuration and setup script. It takes a LONG time and it's probably gonna FAIL. You can rerun the script with ```vagrant provision```, but if it fails at first it's probably gonna FAIL again.

```
==> default: TASK: [edxapp | install python base-requirements] *****************************
==> default: failed: [localhost] => {"changed": true, "cmd": "/edx/app/edxapp/venvs/edxapp/bin/pip install -i https://pypi.python.org/simple --exists-action w --use-mirrors -r /edx/app/edxapp/edx-platform/requirements/edx/base.txt ", "delta": "0:00:42.542346", "end": "2015-07-30 22:01:03.052959", "item": "", "rc": 1, "start": "2015-07-30 22:00:20.510613"}
...
... (A whole bunch of nonsense)
...
==> default:   Downloading dm.xmlsec.binding-1.3.2.tar.gz (119kB)
==> default:     Error: cannot get XMLSec1 pre-processor and compiler flags; do you have the `libxmlsec1` development package installed?
==> default:     Complete output from command python setup.py egg_info:
==> default:     Error: cannot get XMLSec1 pre-processor and compiler flags; do you have the `libxmlsec1` development package installed?
==> default:
==> default:     ----------------------------------------
==> default:     Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-build-sUv02Y/dm.xmlsec.binding
==> default:
==> default: FATAL: all hosts have already failed -- aborting
==> default:
==> default: PLAY RECAP ********************************************************************
==> default:            to retry, use: --limit @/root/vagrant-devstack.retry
==> default: localhost                  : ok=171  changed=41   unreachable=0    failed=1
The SSH command responded with a non-zero exit status. Vagrant
assumes that this means the command failed. The output for this command
should be in the log above. Please read the output to determine what
went wrong.
```

6 hmmm... Do we have libxmlsec1 installed? I don't think so... Let's do that manually first.

```
vagrant up
sudo apt-get install libxmlsec1
sudo apt-get install libxmlsec1-dev
```

7 good, we have that. Let's try the command that's running during configuration again. Trust me, use sudo.

```
sudo /edx/app/edxapp/venvs/edxapp/bin/pip install -i https://pypi.python.org/simple --exists-action w --use-mirrors -r /edx/app/edxapp/edx-platform/requirements/edx/base.txt
```

8 looking good so far... Except you get another error...

```
swig -python -I/usr/include/python2.7 -I/usr/include/x86_64-linux-gnu -I/usr/include -I/usr/include/openssl -includeall -modern -o SWIG/_m2crypto_wrap.c SWIG/_m2crypto.i

unable to execute swig: No such file or directory

error: command 'swig' failed with exit status 1
```

9 okay... same trick... just manually install it

```
sudo apt-get install swig
```

10 let's try the command in #7 again, and it looks good... it takes quite a while to run... and... DONE!

11 to run the edx studio application, do the following

```
sudo su edxapp
source /edx/app/edxapp/edxapp_env
cd /edx/app/edxapp/edx-platform
sudo paver devstack studio
```

12 and this is what you get

```
/edx/app/edxapp/.rbenv/versions/1.9.3-p374/lib/ruby/1.9.1/rubygems/dependency.rb:247:in `to_specs': Could not find bundler (>= 0) amongst [bigdecimal-1.1.0, io-console-0.3, json-1.5.4, minitest-2.5.1, rake-0.9.2.2, rdoc-3.9.4] (Gem::LoadError)
...
... (a whole bunch of nonsense)
...
Build failed running pavelib.servers.devstack: Subprocess return code: 1
```

13 okay... ```sudo gem install bundler``` remember to use sudo

14 let's try the command in #11 again. You will see a bunch of "Could not find a tag or branch", just ignore it. After 27 such messages, we find out that the edx studio instance is finally running! Yay!!!

15 You are so excited that you open up your browser, go to localhost:8001 and start registering an account, and you get this in your log:

```
DatabaseError: (1054, "Unknown column 'auth_userprofile.bio' in 'field list'")
```

16 You google this and find only one edx article about that and you find out how to solve in the edx wiki [here](https://github.com/edx/configuration/wiki/edX-Developer-Stack) by searching "migration"

```
sudo paver update_db -s devstack
```

17 A lot of things are going on... once it's done, it's all done.

18 Congratulations, you now have an edX-platform running on your vagrant machine with port 8001

19 [Note] to verify your email while registering an account, pay special attention to your vagrant console

20 [Note] if you ever shutdown your computer without ```vagrant halt```, the mongodb files will be corrupted, and you will receive an error next time you start the server saying: ```[Errno 111] Connection refused```. In this case, don't panic, just do the following:

```
vagrant halt
vagrant up
vagrant ssh
# then in the vagrant shell
sudo rm /edx/var/mongo/mongodb/mongod.lock
sudo mongod -repair --config /etc/mongod.conf
sudo chown -R mongodb:mongodb /edx/var/mongo/.
sudo /etc/init.d/mongod start
```