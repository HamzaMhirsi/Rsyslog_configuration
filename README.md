# Send log file over Rsyslog/TLS
# Rsyslog configuring with TLS (send log file)

Today we will share a small tutorial where we will talk about all the steps that we need to configure Rsyslog and to create certificates:

1. Rsyslog
2. TLS
3. Software needed 
4. Configuration
Summary


# 1. Rsyslog
Rsyslog is an Open Source software work on Unix, Rsyslog help to send messages over IP network, it's based on syslog protocol, and can help to filter traffic and flexible configuration.

# 2. TLS
Transport Layer Security is a cryptographic protocol that provide a secure connection over computer network, " Several versions of the protocols find widespread use in applications such as web browsing, email, instant messaging, and voice over IP (VoIP). Websites can use TLS to secure all communications between their servers and web browsers. " [Source Wikipedia]


# 3. Software needed
In this configuration we will need 2 Virtual machine, I prefer to work on Centos 7, you can find a link to Download the .iso LINK, and we will use Virtual Box (I support Open Source Products =p).


# 4. Configuration
In this part we will describe how we will configure our both machine step by step, there is always too different ways to configure our Rsyslog, by personally I prefer the way that follow:

+ After we install our machine "Client" and "Server", I choose to work on GUI version so it will be easier to test the traffic when we open our browser, and I will use "Putty" to connect to my machines, (Both machine don't need to be in the same network, the configuration will not defer, either we will use a public IP @ or a private one). For more details you can check this video:



+ After we install both machine we will need to add some line of code so we can ensure that our machine will work perfectly:

#sudo yum repolist 
#sudo yum update # sudo yum upgrade  

+ Now both machine are ready to be used, we need a tool that will help us to catch the traffic on the client machine, I will simply install wireshark and I will configure the machine to capture traffic when it boot:

By default crontab will be installed, else check this link, in this part will capture src/dest mac and src/dest IP @ going through our default interface, we will add a separator so it will be more clear, and we will finally save log in a file in our tshark directory (we can add more fields based on our needsfor example here is a list for ssl link):
#sudo yum install wireshark
#tshark -v (to check version in case)
#sudo crontab -e (that's will open a file with vi)
Add those line of code as indicated in the picture (hit "i" to insert):



I prefer to use ">>" in place of ">" so I can append traffic to the existing file, to quit and save just hit escape + ':qw'

When we will reboot our machine we will mention a new file created in the path indicated, when we generate a traffic we can check the file using:
#tail -f /home/client/tshark/traffic.txt
Saving data in a .txt file will not be a good idea especially if the traffic is so big, so I decided to configure a task that will be executed each 1min to delete the "tarrafic.txt" file, don't be aware it will be created again.
#sudo vi /etc/crontab 


+ We have a nice traffic on our file, we just need to connect to any web site to make sure that our configuration is working (you can run from terminal # firefox www.google.com &), logging our activity with a wrong clock mean that we are digging into water, so we need to set our time:

First we need fix time on our client and server machine:
#sudo yum install ntp -y
#sudo systemctl start ntpd
#sudo systemctl enable ntpd (start and enable ntpd daemon to satrt on system boot)
#sudo firewall-cmd --permanent --add-service=ntp
#sudo firewall-cmd --reload (allow udp traffic for ntp on our firewalls)
#sudo vi /etc/ntp.conf
In this file we need to set the ntp server that we will use to synchronize time, time is too important to get the right time of log files, so we just check this url to choose our servers



"We use the iburst option for each servers, per the NTP Pool recommendations. That way, if the server is unreachable, this will send a burst of eight packets instead of the usual one packet. Using the burst option in the NTP Pool Project is considered abuse as it will send those eight packets every poll interval, whereas iburst sends the eight packets only the first time.
Next, make sure the default configuration does not allow management queries. If you don't, your server could be used in NTP reflection attacks, or could be vulnerable to ntpq and ntpdc queries that attempt to modify the state of the server. Check that the noquery option is added to the default restrict lines. Also make sure you add the options kod and limited as they restrict too eagerly asking clients and enforce rate limiting." [Source]

Now we need to restart ntpd and check his status (we need to wait some time so our machine will connect to servers):

#sudo systemctl restart ntpd
#ntpq -p
The output will be like that:



+ Now it's time to create our certificate on our server machine that will be used to secure the traffic (encrypt the traffic between the client and the server) if you like to know more how TLS work check this link, first we need to create the certificate of the certification authority CA (all this configuration will be in the server machine as it's the authority that will create certificates for all the clients) :

#cd
#mkdir rsyslogTLS (create the directory where we will put our certification both in server and client machine)
#cd rsyslogTLS
#sudo yum install -y gnutls-utils (install needed package)
#sudo certtool --generate-privkey --outfile ca-key.pem (create the private key of CA) # sudo chmod 400 ca-key.pem (we need to secure our private key, so only root can check it) # sudo certtool --generate-self-signed --load-privkey ca-key.pem --outfile ca.pem  (we generate our self signed public key)
Rq: In this step there is some question to answer make sure that you give the right answer for the following certificates too.
#sudo certtool --generate-privkey --outfile client-key.pem --bits 2048 (we will create the private key of our client)
#sudo certtool --generate-request --load-privkey client-key.pem --outfile client-request.pem (we will generate now the certification request)
#sudo certtool --generate-certificate --load-request client-request.pem --outfile client-cert.pem --load-ca-certificate ca.pem --load-ca-privkey ca-key.pem (finally we will create the certification of our client)
- After this step we need to delete the request of the client certification(rm -rf client-request.pem) than we will send the (ca.pem, client-key.pem and client-cert.pem) that will be used to encrypt the traffic to the client) even we use "scp" or we just copy them on a flash disk, or just copy/past them from terminals, we will need to generate a certification of our server too (it's the same process creating a "key" and a "cert".

+ In this part I find too many difficulties on setting SELinux and Firewalld so I easily disabled them and I installed IPtables, check this link to stop SElinux and this link to disable firewalld and enable iptable.

+ Finally it's time to install rsyslog, we have certificates, we have traffic and we need to send it now, in this part I will talk about server configuration then client, the file will be a little bit complicated for those that use those configuration file for the first time, what I can say is; read it carefully and google every unclear tag or modules (^.^) :

#cd
#sudo mkdir /etc/ssl/rsyslog (create or directory)
#sudo yum install rsyslog-gnutls -y (install rsyslog)
The only file that I will use for configuration is /etc/rsyslog.conf, in some documentation you will see that they create other configuration file in "/etc/rsyslog.d/" directory that will be imported to rsyslog.conf file automatically when the service go up, I mainly like to use one file it's more easy, but the time you add too many lines of code you may regret (don't say that I didn't mention that =p ).

###### SERVER PART ######

#sudo vi /etc/rsyslog.conf
In the following line I will describe in comment what each line of code mean:

#In this part we will load tcp and we will mention the AuthMode that we will use ( we will send our traffic on TCP) module(load="imtcp" StreamDriver.AuthMode="x509/xxx" StreamDriver.Mode="1") # make gtls driver the default $DefaultNetstreamDriver gtls $DefaultNetstreamDriverCAFile /home/server/rsyslogTLS/ca.pem $DefaultNetstreamDriverCertFile /home/server/rsyslogTLS/server-cert.pem $DefaultNetstreamDriverKeyFile/home/rsys/rsyslogTLS/prsyslog-key.pem # Choose the port that we will listen on $InputTCPServerRun 6514  # Only allow the domain that we choose (optional)
$InputTCPServerStreamDriverPermittedPeer *.example.com

   + As we will accept traffic on port 6514 we need to activate it on our Iptable:
#sudo vi /etc/sysconfig/iptables
And we simply add the following line
-A INPUT -m state --state NEW -m tcp -p tcp --dport 6514 -j ACCEPT
#sudo /etc/init.d/iptables reload

###### CLIENT PART ######

In this part we will send to our server the file “traffic.txt” created:

#sudo vi /etc/rsyslog.conf
The file is open you just type “i” to insert, save and quit with “esp + :wq”:

#Load imfile module that will help us to send a file over rsyslog module(load=”imfile”) # Set gtls driver, authentication mode, stream driver mode to enable only tls and add certificates module(load=”imtcp” StreamDriver.AuthMode=”x509/yyy” StreamDriver.Mode=”1”) $DefaultNetstreamDriverCAFile /home/client/rsyslogTLS/ca.pem $DefaultNetstreamDriverCertFile /home/client/rsyslogTLS/client-cert.pem $DefaultNetstreamDriverKeyFile /home/client/rsyslogTLS/client-key.pem #read from the traffic log file and assign the tag "ssl" for its messages input(type="imfile "File="/home/client/rsyslog/traffic/txt" deleteStateOnFileDelete="on" reopenOnTruncate="on "Tag="ssl") #send ssl traffic to server on port 6514 if($syslogtag=="ssl") then{ action(name="sslLog" type="omfwd" protocol="tcp" target="IP@server" port="6514" template="ForwardToRsyslogServerFormatTemplate" StreamDriver="gtls" StreamDriverMode="1" StreamDriverAuthMode="x509/yyy")} #We can permit only to send traffic to a DNS name if we have (optional) $ActionSendStreamDriverPermittedPeer NAME.EXAMPLE.ORG #limit the message size (optional) $MaxMessageSize 10k # If you want to send every think typed in the client machine, If you use hostnames instead of IP configure DNS on /etc/hosts (optional) *.*@@ NAME.EXAMPLE.ORG:6514
+ To start we just need to restart the rsyslog service on both client and server machine:

#sudo service rsyslog restart
To track the traffic on our server we tcpdump:

#sudo yum install tcpdump -y
#sudo tcpdump -i eth0 tcp port 6514 -X -s 0 -nn
Finally you can see the logged traffic, you can follow the configuration in this video you may enjoy it ;):



+ To debug any problem with rsyslog, please check your log messages where the path is indicated in the configuration file, and else check you FW if blocking the traffic in both server/client machine.

Summary
In this article we discussed how to configure rsyslog on our server and our client to send a log file captured with tshark. Rsyslog can use TLS certificate to encrypt the traffic.

If you face any problem, just ask the question in the comment section and wait the Peerlyst community to respond ^^.

I hope that you enjoyed this article as always, if you have something to add don't hesitate to write a comment ^^ and follow :p

Read more
https://www.peerlyst.com/posts/send-log-file-over-rsyslog-tls-hamza-m-hirsi?trk=search_suggestion_query

References
+ https://www.digitalocean.com/community/tutorials/how-to-configure-ntp-for-use-in-the-ntp-pool-project-on-centos-7
+ https://www.youtube.com/watch?v=pgcdHmsfmJM
