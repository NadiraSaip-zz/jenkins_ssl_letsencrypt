# jenkins_ssl_letsencrypt
# Set up SSL on Jenkins using Let's Encrypt

Jenkins can be an integral piece of software used to speed up development and deliver production services to customers. Because of this, security in Jenkins must be carefully implemented. This brief guide will walk a Jenkins sysadmin through setting up SSL for Jenkins.
1. Prior knowledge of public key infrastructure (PKI).
2. RedHat or CentOS GNU/Linux for file paths. Jenkins specific knowledge translates across distros.
3. The hosting server has a fully qualified domain name.
4. You installed Jenkins via a system package.
5. You’re using Jenkins Winstone (the Jenkins built-in web server) which is the default.
6. You’re running Jenkins as a normal user (e.g. jenkins). Normal users can’t listen on ports less than 1024 in Linux. Note: If  - you wish to use 443 instead of 8443 I recommend using iptables to forward ports.

# Install Let’s Encrypt certbot and generate certificate
Certbot is popular client for fetching and configuring Let’s Encrypt SSL certificate in no time along with easy renewal. Its well integrated with popular servers (nginx, apache, haproxy etc.). I will install it on centos instance. 
```
yum install certbot -y
```
Considering jenkins.example.com is our jenkins endpoint, we can generate the certificates with this simple command:
```
sudo certbot certonly --standalone -d jenkins.example.com
```
It will ask for few confirmations and email. If thee certification is generated you will see the message which will start with 
# IMPORTANT NOTES:
-Congratulations! Your certificate and chain has been saved at :....

The generated cert and private key path is on the following file format which you will need in next step.
```
Cert: /etc/letsencrypt/live/jenkins.example.comenkins/fullchain.pem
PrivKey: /etc/letsencrypt/live/jenkins.example.com/privkey.pem
```
# Create a Java Keystore
1. Reference your SSL certificates and key (listed above).
2. Convert the SSL certificates into an intermediate format (PKCS12 keystore).
3. Convert the intermediate format into a Java Keystore (JKS keystore).

The following command will convert SSL certificates into the intermediate format (PKCS12).
```
openssl pkcs12 -export -out jenkins_keystore.p12 \
  -passout 'pass:changeit' -inkey /etc/letsencrypt/live/jenkins.example.com/privkey.pem \
  -in /etc/letsencrypt/live/jenkins.example.com/fullchain.pem -certfile /etc/letsencrypt/live/jenkins.example.com/cert.pem -name jenkins.example.com
```
Then convert the intermediate format (PKCS12) to Java Keystore (JKS). This way Jenkins can use the JKS.
```
keytool -importkeystore -srckeystore jenkins_keystore.p12 \
  -srcstorepass 'changeit' -srcstoretype PKCS12 \
  -srcalias example.com -deststoretype JKS \
  -destkeystore jenkins_keystore.jks -deststorepass 'changeit' \
  -destalias example.com
```
Make sure the followings are correct:
1. The -name, -srcalias, and -destalias must be the same as the domain name (e.g. example.com).
2. The password for the PKCS12 store and the JKS store must be exactly the same. Jenkins will fail to start if it is not the case.
3. You should probably use a different password than changeit (which is conventional knowledge for Java keystores). It will protect you if the keystore is compromised and downloaded but the password is not known. Though, in a system breach, it is wise to revoke and request    new certificates.
4. The final keystore file is jenkins_keystore.jks.
Now,  move the keystore to a place which can be referenced in Jenkins configuration.
```
mkdir -p /etc/jenkins
cp jenkins_keystore.jks /etc/jenkins/

#configure permissions to secure your keystore
chown -R jenkins: /etc/jenkins
chmod 700 /etc/jenkins
chmod 600 /etc/jenkins/jenkins_keystore.jks
```
# Configure Jenkins to use SSL

Edit /etc/sysconfig/jenkins and set the following variables.
```
#disable HTTP
JENKINS_PORT="-1"

#configure HTTPS
JENKINS_HTTPS_PORT="8443"
JENKINS_HTTPS_KEYSTORE="/etc/jenkins/jenkins_keystore.jks"
JENKINS_HTTPS_KEYSTORE_PASSWORD="changeit"
JENKINS_HTTPS_LISTEN_ADDRESS="127.0.0.1" #remove it if you can't access the page on web browser
```
# Verify SSL is running
Ensure a correct certificate chain order. The following command also accounts for Server Name Indication (SNI).
```
openssl s_client -connect example.com:443 -servername example.com < /dev/null
```

View certificate information of the remote service.
```
openssl s_client -connect example.com:443 -servername example.com < /dev/null | openssl x509 -text | less
```


# Congratulations your website is now SSL Certified! 
If you can't have problems and can't access it follow the below troubleshooting solutions. 

To learn more about these options you can execute (while Jenkins is running):
```
java -jar /usr/lib/jenkins/jenkins.war --help
ps aux | grep 'jenkins\.war'
```
See /var/log/jenkins/jenkins.log for potential troubleshooting.

# Configure iptables to port forward
Iptables can be used to forward port 443 to 8443. This provides a better user experience since this is the standard port for HTTPS connections. To do this on a RedHat system a few things must occur:

- The kernel must allow forwarding.
- iptables needs rules for forwarding.
- The recommendations are relevant to interface eth0. Double check you’re using the correct interface for your system.

#Configure kernel to allow port forwarding  
I have done it on AWS Consul but you can follow the below steps on CLI.
To enable forwarding during runtime:
```
#allow forwarding
sysctl -w net.ipv4.ip_forward=1
#allow forwarding to localhost on eth0
sysctl -w net.ipv4.conf.eth0.route_localnet=1
```
For forwarding settings to persist on reboot, add or change the following settings in /etc/sysctl.conf.

```
net.ipv4.ip_forward = 1
net.ipv4.conf.eth0.route_localnet = 1
```
# Configure iptables

Depending on how iptables is configured you may have to adapt how you insert the rules into the runtime firewall. It may be simpler to write your rules in /etc/sysconfig/iptables and reload the firewall.
```
#for remote connections
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j DNAT --to-destination 127.0.0.1:8443
iptables -A FORWARD -i eth0 -m state --state NEW -m tcp -p tcp -d 127.0.0.1 --dport 8443 -j ACCEPT

#for localhost connections
iptables -t nat -A OUTPUT -p tcp --dport 443 -d 127.0.0.1 -j DNAT --to-destination 127.0.0.1:8443
```
Use iptables-save to update /etc/sysconfig/iptables rules.

# Repeat the steps to Verify SSL is running

If you visit your secured Jenkins and find your browser is not showing a secure connection, then you’re likely missing the certificate authority certificate in your browser trust store.

