# idm-satellite-openshift-demo
Notes on installing and configuring a OpenShift Container Platform in a disconnected environment with IdM and Satellite 6

The following configurations of IdM, Satellite and OpenShift Container
Platform (OSCP) assume that we're working in a lab environment that is
disconnected from typical enterprise services.  For the most part, we
also assume no access to the internet.  The one exception would be for
the Satellite server, which is allowed to access the internet via web
proxy.

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

1. Run "`subscription-manager register`" with the appropriate
   --proxy=... and related arguments.

1. Run "`subscription-manager list --all --available`" to find the Pool ID of the Satellite subscription. 

1. Run "`subscription-manager attach --pool=`"... with the appropriate Pool ID.

1. Run "`subscription-manager repos --disable=\*`" to disable all repos.

1. Selectively enable repos with...
<pre><code>for r in rhel-7-server-rpms \
         rhel-server-rhscl-7-rpms \
         rhel-7-server-satellite-6.2-rpms; do
    subscription-manager repos --enable=$r;
done;</code></pre>
  
1. Install bits and reboot with...
<pre><code>yum install -y satellite-installer katello ipa-client \
&& yum update -y \
&& sync && reboot</code></pre>
  
1. Log back into the server, and run the installer like so:
<pre><code>satellite-installer --foreman-initial-organization "OSCP PoC" \
                        --foreman-initial-location "Innovation Zone" \
                        --foreman-admin-username=admin \
                        --foreman-admin-password=Redhat1! \
                        --scenario=satellite</code></pre>
   Be sure to provide your own values for initial org, initial
   location, and admin password.  Also, use one or more of the
   following installer options to configure access through the web
   proxy:
<pre><code>--katello-proxy-password Proxy password for authentication (default: nil)
--katello-proxy-port Port the proxy is running on (default: nil)
--katello-proxy-url URL of the proxy server (default: nil)
--katello-proxy-username Proxy username for authentication (default: nil)</code></pre>
  
1. Log into access.redhat.com and generate a manifest for your
   Satellite with all required subscriptions.  You'll need a RHEL
   Server subscription for the IdM server, and some number of
   OpenShift subs for the OSCP cluster.  Additional specific details
   on creating and downloading the manifest file are found here:
   https://access.redhat.com/solutions/118573

1. Log into the Satellite web UI, and navigate to Content>Red Hat
   Subscriptions.  Click on the "Browse..." button to select your
   freshly downloaded manifest zip file, and the "Upload" to send it
   to the Satellite.  Wait until Satellite tells you that it has
   successfully imported the manifest.

1. On the satellite server, create ~/.hammer/cli_config.yml in order
   to provide credentials to the hammer cli:
<code><pre>mkdir ~/.hammer
cat > ~/.hammer/cli_config.yml <<\EOF
    :foreman:
        :enable_module: true
        :host: 'https://localhost/'
        :username: 'admin'
        :password: 'Redhat1!'
EOF</pre></code>

1. Run the following commands to enable the RPM repos that we need:
<pre><code>hammer repository-set enable --name "Red Hat Enterprise Linux 7 Server (Kickstart)" --product "Red Hat Enterprise Linux Server" --organization-id 1 --basearch x86_64 --releasever 7Server
hammer repository-set enable --name "Red Hat Enterprise Linux 7 Server (RPMs)" --product "Red Hat Enterprise Linux Server" --organization-id 1 --basearch x86_64 --releasever 7Server
hammer repository-set enable --name "Red Hat Satellite Tools 6.2 (for RHEL 7 Server) (RPMs)" --product "Red Hat Enterprise Linux Server" --organization-id 1 --basearch x86_64 
hammer repository-set enable --name "Red Hat Enterprise Linux 7 Server - Extras (RPMs)" --product "Red Hat Enterprise Linux Server" --organization-id 1 --basearch x86_64 
hammer repository-set enable --name "Red Hat OpenShift Enterprise 3.2 (RPMs)" --product "Red Hat OpenShift Enterprise" --organization-id 1 --basearch x86_64</code></pre>

2. Save the following script and run it.  It will create all of the
docker image repositories we'll need for OSCP.
<pre><code>#!/bin/bash
    
    ORG_ID=1
    PRODUCT_NAME="OSCP Docker Images"
    
    upstream_repos=( \
        dotnet/dotnetcore-10-rhel7 \
        jboss-amq-6/amq62-openshift \
        jboss-datagrid-6/datagrid65-openshift \
        jboss-decisionserver-6/decisionserver62-openshift \
        jboss-decisionserver-6/decisionserver63-openshift \
        jboss-eap-6/eap64-openshift \
        jboss-eap-7/eap70-openshift \
        jboss-fuse-6/fis-java-openshift \
        jboss-fuse-6/fis-karaf-openshift \
        jboss-processserver-6/processserver63-openshift \
        jboss-webserver-3/webserver30-tomcat7-openshift \
        jboss-webserver-3/webserver30-tomcat8-openshift \
        openshift3/image-inspector \
        openshift3/jenkins-1-rhel7 \
        openshift3/logging-auth-proxy \
        openshift3/logging-deployment \
        openshift3/logging-elasticsearch \
        openshift3/logging-fluentd \
        openshift3/logging-kibana \
        openshift3/metrics-cassandra \
        openshift3/metrics-deployer \
        openshift3/metrics-hawkular-metrics \
        openshift3/metrics-heapster \
        openshift3/mongodb-24-rhel7 \
        openshift3/mysql-55-rhel7 \
        openshift3/nodejs-010-rhel7 \
        openshift3/ose-deployer \
        openshift3/ose-docker-builder \
        openshift3/ose-docker-registry \
        openshift3/ose-haproxy-router \
        openshift3/ose-pod \
        openshift3/ose-recycler \
        openshift3/ose-sti-builder \
        openshift3/perl-516-rhel7 \
        openshift3/php-55-rhel7 \
        openshift3/postgresql-92-rhel7 \
        openshift3/python-33-rhel7 \
        openshift3/ruby-20-rhel7 \
        rhscl/mariadb-101-rhel7 \
        rhscl/mongodb-26-rhel7 \
        rhscl/mongodb-32-rhel7 \
        rhscl/mysql-56-rhel7 \
        rhscl/nodejs-4-rhel7 \
        rhscl/perl-520-rhel7 \
        rhscl/php-56-rhel7 \
        rhscl/postgresql-94-rhel7 \
        rhscl/postgresql-95-rhel7 \
        rhscl/python-27-rhel7 \
        rhscl/python-34-rhel7 \
        rhscl/python-35-rhel7 \
        rhscl/ruby-22-rhel7 \
        rhscl/ruby-23-rhel7 \
    )
    
    hammer product create --name "$PRODUCT_NAME" --organization-id $ORG_ID
    
    PRODUCT_LIST_FILE=`mktemp`
    hammer repository list --organization-id "$ORG_ID" \
        --product "$PRODUCT_NAME" > $PRODUCT_LIST_FILE
    
    for i in ${upstream_repos[@]}; do
      found=`grep "$i" $PRODUCT_LIST_FILE`
      if [[ -z $found ]]; then
          echo "Creating repository for $i"
          hammer repository create --name "$i" --organization-id $ORG_ID \
    	  --content-type docker --url "https://registry.access.redhat.com" \
    	  --docker-upstream-name "$i" --product "$PRODUCT_NAME";
      else
          echo "Found Satellite mirror for repository $i"
      fi
    done
    
    rm -f $PRODUCT_LIST_FILE > /dev/null</code></pre>
    
1. Synchronize all repositories with the following command:
<pre><code>for i in $(hammer --csv repository list --organization-id=1  | awk -F, {'print $1'} | grep -vi '^ID'); do hammer repository synchronize --id ${i} --organization-id=1 --async; done</code></pre>
   Take a well earned break while all repos sync!

1. Create the Content View for a regular RHEL server:
<pre><code>hammer content-view create --name "RHEL 7 Server" --organization-id "1" 

hammer content-view add-repository --name "RHEL 7 Server" --organization-id "1" \
--product "Red Hat Enterprise Linux Server" \
--repository "Red Hat Enterprise Linux 7 Server - Extras RPMs x86_64"

hammer content-view add-repository --name "RHEL 7 Server" \
--organization-id "1" --product "Red Hat Enterprise Linux Server" \
--repository "Red Hat Enterprise Linux 7 Server RPMs x86_64 7Server"

hammer content-view add-repository --name "RHEL 7 Server" --organization-id "1" \
--product "Red Hat Enterprise Linux Server" \
--repository "Red Hat Satellite Tools 6.2 for RHEL 7 Server RPMs x86_64"</code></pre>

1. Create server7-ak

1. Create the Content View for an OSCP server:
<pre><code>hammer content-view create --name "RHEL 7 Server" --organization-id "1" 

hammer content-view add-repository --name "RHEL 7 Server" --organization-id "1" \
--product "Red Hat Enterprise Linux Server" \
--repository "Red Hat Enterprise Linux 7 Server - Extras RPMs x86_64"

hammer content-view add-repository --name "RHEL 7 Server" \
--organization-id "1" --product "Red Hat Enterprise Linux Server" \
--repository "Red Hat Enterprise Linux 7 Server RPMs x86_64 7Server"

hammer content-view add-repository --name "RHEL 7 Server" --organization-id "1" \
--product "Red Hat Enterprise Linux Server" \
--repository "Red Hat Satellite Tools 6.2 for RHEL 7 Server RPMs x86_64"

hammer content-view add-repository --name "Openshift 3" --organization-id "1" \
--product "Red Hat OpenShift Enterprise" \
--repository "Red Hat OpenShift Enterprise 3.2 RPMs x86_64"</code></pre>

1. Create oscp-ak

# Step 2: Install IdM

The IdM server will host DNS and user authentication services.

1. Create a VM with 4GB of RAM and 12GB of storage.  Name it
   `idm.atgreen.org`.

1. Run "`rpm -ihv http://IP-OF-SATELLITE/pub/katello-ca-consumer-latest.noarch.rpm`"

1. Run "`subscription-manager register --organization='OSCP PoC' --activationkey='server7-ak'`"

1. Run "`yum -y install ipa-server && yum -y update && sync && reboot`"

1. Log back into the IdM server and run "`ipa-server-install --setup-dns --mkhomedir`"

1. Log back into the Satellite server and set /etc/resolv.conf to
   point at the IdM server, where DNS is now hosted.

1. On the Satellite, run "`ipa-client-install --mkhomedir`"

1. Set the nameserver on your browser host to point at the IdM server,
   and keep it there.

# Step 3: Install OpenShift Container Platform (OSCP)

