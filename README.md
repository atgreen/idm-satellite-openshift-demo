# idm-satellite-openshift-demo
Notes on installing and configuring a OpenShift Container Platform in a disconnected environment with IdM and Satellite 6

# Step 1: Install Satellite 6

In the disconnected environment, Red Hat Satellite becomes the conduit
through which all content will be presented to the demo systems.  This
includes, among other things, hosting RPM content and all docker
container images.  Of course, this assumes that Satellite is able to
proxy out to the internet, which is typically the case.

We will not be using Satellite to provision systems.  It will just be
acting as the trusted repository for RPM and container content.

1. Create a VM with 12GB of RAM and 400GB of storage.  Name it
   `sat6.atgreen.org`.

2. Run "`subscription-manager register`" with the appropriate
   --proxy=... and related arguments.

4. Run "`subscription-manager list --all --available`" to find the Pool ID of the Satellite subscription. 

5. Run "`subscription-manager attach --pool=`"... with the appropriate Pool ID.

6. Run "`subscription-manager repos --disable=\*`" to disable all repos.

7. Selectively enable repos with...
```
for r in rhel-7-server-rpms \
         rhel-server-rhscl-7-rpms \
         rhel-7-server-satellite-6.2-rpms; do
  subscription-manager repos --enable=$r;
done;
```
  
8. Install bits and reboot with...
```
yum install -y satellite-installer katello ipa-client \
&& yum update -y \
&& sync && reboot
```
  
9. Log back into the server, and run the installer like so:
```
satellite-installer --foreman-initial-organization "OSCP PoC" \
                    --foreman-initial-location "Innovation Zone" \
                    --foreman-admin-username=admin \
                    --foreman-admin-password=Redhat1! \
                    --scenario=satellite
```
   Be sure to provide your own values for initial org, initial
   location, and admin passord.  Also, use one or more of the
   following installer options to configure access through the web
   proxy:
```
    --katello-proxy-password Proxy password for authentication (default: nil)
    --katello-proxy-port Port the proxy is running on (default: nil)
    --katello-proxy-url URL of the proxy server (default: nil)
    --katello-proxy-username Proxy username for authentication (default: nil)
```
  
10. 


