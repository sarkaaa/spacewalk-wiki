# Change CA certificate



Please note!  following these steps will render all clients unable to connect until you redeploy the new ca cert ( RHN-ORG-TRUSTED-SSL-CERT) to EVERY client currently connection to the spacewalk/satellite server.


Recreate CA cert:


 1.  mv /root/ssl-build /root/ssl-build.bak
 2.  cd /root
 3.  rhn-ssl-tool --gen-ca
 4.  rhn-ssl-tool --gen-server
 5.  cp ./ssl-build/RHN-ORG-TRUSTED-SSL-CERT ./ssl-build/rhn-org-trusted-ssl-cert-1.0-1.noarch.rpm  /var/www/html/pub
 6.  rpm -e rhn-org-httpd-ssl-key-pair-HOSTNAME
 7.  rpm -ivh ./ssl-build/HOSTNAME/rhn-org-httpd-ssl-key-pair-HOSTNAME-1.0-1.noarch.rpm
 8.  rhn-ssl-dbstore -vvv --ca-cert /root/ssl-build/RHN-ORG-TRUSTED-SSL-CERT
 9.  spacewalk-service restart
 10. Redeploy RHN-ORG-TRUSTED-SSL-CERT to clients (if needed)
