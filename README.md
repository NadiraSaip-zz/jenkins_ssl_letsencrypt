# jenkins_ssl_letsencrypt
# Set up SSL on Jenkins using Let's Encrypt
1. Install Let’s Encrypt certbot and generate certificate
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
# -Congratulations! Your certificate and chain has been saved at :....

The generated cert and private key path is on the following file format which you will need in next step.
```
Cert: /etc/letsencrypt/live/jenkins.example.comenkins/fullchain.pem
PrivKey: /etc/letsencrypt/live/jenkins.example.com/privkey.pem
```
Jenkins can be an integral piece of software used to speed up development and deliver production services to customers. Because of this, security in Jenkins must be carefully implemented. This brief guide will walk a Jenkins sysadmin through setting up SSL for Jenkins.
- Prior knowledge of public key infrastructure (PKI).
- RedHat or CentOS GNU/Linux for file paths. Jenkins specific knowledge translates across distros.
- The hosting server has a fully qualified domain name.
- You installed Jenkins via a system package.
- You’re using Jenkins Winstone (the Jenkins built-in web server) which is the default.
- You’re running Jenkins as a normal user (e.g. jenkins). Normal users can’t listen on ports less than 1024 in Linux. Note: If  - you wish to use 443 instead of 8443 I recommend using iptables to forward ports.
