#Monitoring transactor process with monit

You can use [monit](http://mmonit.com/monit/) to monitor transactor process.
Monit also can take actions such as start/stop/restart process by following a rules you write.
So monit can act like startup system and make sure process is running.

Here is an example of `monitrc` that monitor a transactor and an application process.

```
set daemon 60
set logfile syslog facility log_daemon
set idfile /var/.monit.id
set statefile /var/.monit.state
set mailserver smtp.gmail.com port 587
    username "myalert@gmail.com" password "mypassword"
    using tlsv1
    with timeout 30 seconds
set alert mymonitor@gmail.com
set httpd port 2813 address localhost
    allow localhost
    allow admin:mypassword
  
check process datomic with pidfile "/home/myuser/.datomic.pid"
      start program = "/bin/su - myuser -c '/home/myuser/datomic-free-0.8.3848/bin/transactor /home/myuser/transactor.properties'"
      stop program = "/bin/kill `cat /home/myuser/.datomic.pid`"
      if cpu > 60% for 3 cycles then alert
      if cpu > 90% for 5 cycles then restart
      if totalmem > 500.0 MB for 5 cycles then restart
      if 3 restarts within 4 cycles then alert
 
check process myapp with pidfile "/home/myuser/.myapp.pid"
      start program = "/bin/su - myuser -c '/home/myuser/myapp/bin/daemon.sh start'"
      stop program = "/bin/kill `cat /home/myuser/.myapp.pid`"
      if failed port 8080 and protocol http
      and request "/" then restart
      if cpu > 60% for 3 cycles then alert
      if cpu > 90% for 5 cycles then restart
      if totalmem > 800.0 MB for 5 cycles then restart
      if 3 restarts within 4 cycles then alert
      depends on datomic
```

You should specify path of process id file into `transactor.properties`.

```
pid-file=/home/myuser/.datomic.pid
```

If you use [datomic-free wrapper](https://github.com/cldwalker/datomic-free), change `start program = ...` line as following.

```
      start program = "/bin/su - myuser -c '/home/myuser/.datomic-free/bin/datomic-free start /home/myuser/.datomic-free/transactor.properties'"
```

The reason why I use `su` instead of `as uid myuser and gid mygroup` [option](http://mmonit.com/monit/documentation/monit.html#configuration_examples) is that monit restricts environment variables such as HOME and PATH when invoking commands. 

You can check syntax validity as following.

```
$ monit -tc monitrc
Control file syntax OK
```

This `monitrc` is expected to run as root.
You can run monit as following.

```
$ chmod 0600 monitrc
$ sudo cp -p monitrc /etc/monitrc
$ sudo chown root:root /etc/monitrc
$ sudo monit -Ic /etc/monitrc
```

By the way, as you need to specify password of your email account into `monitrc`, I recommend you to create a disposable email account.
