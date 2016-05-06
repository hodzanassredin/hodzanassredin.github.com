--- 
published: true 
layout: post 
title: Simple cassandra cluster deployment with fabric and azure cli.
tags : [cassandra, azure, fabric] 
--- 
 
 
## {{page.title}} 
 
 
 
 
<p class="meta">05 May 2016 &#8211; Karelia</p> 
 
In our project, deployed to azure, we used single node cassandra cluster during test phase. It was deployed manually as described in some blog posts. But after some time we decided to move to a real single datacenter cluster with 3 nodes for more realistic tests and use version 3 with secondary indexes and materialized views. We already have scripts for farm creation from vm images but unfortunately cassandra config is a little bit harder and some time ago new "Resource management" deployment mode for azure was introduced. After some investigation I found existing, ready to use, templates for cassandra, unfortunately some of them are broken and some of them uses Datastax Enterprise version, but we want to use open source version. And from my point of view all that templates are too hard to write and understand (if you don’t have a time) and this time we decided not to use vm images at all, because in case of changes you need to maintain them. So we decided to write some simple scripts which will use azure cli and Fabric for deployment.
First of all you need to install [azure cli](https://azure.microsoft.com/ru-ru/documentation/articles/xplat-cli-install/) and [fabric](http://www.fabfile.org/installing.html).
First thing to do is to [login](https://azure.microsoft.com/en-us/documentation/articles/xplat-cli-connect/) into your azure account and select subscription and set resource mode.
{% highlight bash %} 
azure login
azure account set YourSubscriptionName
azure config mode arm
{% endhighlight %} 
if you don’t see your subscription then you need to execute 
{% highlight bash %} 
azure account download
azure account import PathToDownloadedFile...
{% endhighlight %} 
So everything is ready and we can start. First step is to create a resource group. In our case we will use "West Europe" region.
{% highlight bash %} 
azure group create cassandra-group "West Europe"
{% endhighlight %} 
We will put all vms into subnet of a virtual network. Let’s create them.
{% highlight bash %} 
azure network vnet create cassandra-group cassandra-net "West Europe" 
azure network vnet subnet create cassandra-group cassandra-net cass-sub -a 10.0.0.0/24
{% endhighlight %} 
Next step is optional for you and do it only in case if want to access your nodes from internet. We will create a three (number of nodes) public ips in our group. 
{% highlight bash %} 
azure network public-ip create "cassandra-group" "cass-ip-1" "West Europe" -a Static
azure network public-ip create "cassandra-group" "cass-ip-2" "West Europe" -a Static
azure network public-ip create "cassandra-group" "cass-ip-3" "West Europe" -a Static
{% endhighlight %} 
Now it is time for network interfaces. We will create them in our network and bind them to public ips.
{% highlight bash %} 
azure network nic create "cassandra-group" "cass-nic-1" "West Europe" --subnet-name "cass-sub" --subnet-vnet-name "cassandra-net" -p "cass-ip-1" -a "10.0.0.4"  
azure network nic create "cassandra-group" "cass-nic-2" "West Europe" --subnet-name "cass-sub" --subnet-vnet-name "cassandra-net" -p "cass-ip-2" -a "10.0.0.6"  
azure network nic create "cassandra-group" "cass-nic-3" "West Europe" --subnet-name "cass-sub" --subnet-vnet-name "cassandra-net" -p "cass-ip-3" -a "10.0.0.8"  
{% endhighlight %} 

Our nics are hidden by azure’s firewall so we need to configure it and add allow rules for ssh and cassandra ports.
{% highlight bash %} 
azure network nsg create "cassandra-group" "cass-nsg" "West Europe" 
azure network nsg rule create -g cassandra-group -a cass-nsg -n cass-rule -c Allow -p Tcp -r Inbound -y 100 -f Internet -o * -e * -u 9042
azure network nsg rule create -g cassandra-group -a cass-nsg -n ssh-rule -c Allow -p Tcp -r Inbound -y 200 -f Internet -o * -e * -u 22
azure network nic set cassandra-group cass-nic-1 -o cass-nsg
azure network nic set cassandra-group cass-nic-2 -o cass-nsg
azure network nic set cassandra-group cass-nic-3 -o cass-nsg
{% endhighlight %} 
Main thing is to create vms and attach data disks to them. Every vm will use a separate nic, a ssh key file and ubuntu 15 disk image. How to find a correct image URN for vm described [here](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-cli-ps-findimage/).
{% highlight bash %} 
azure vm create cassandra-group cass01 "West Europe" --nic-name "cass-nic-1" -y linux -Q Canonical:UbuntuServer:15.10:15.10.201604050 -u username -M id_rsa.pub -z Standard_D2_v2 --vnet-name cassandra-net --vnet-subnet-name cass-sub
azure vm disk attach-new "cassandra-group" cass01 100 cass-data-01

azure vm create cassandra-group cass02 "West Europe" --nic-name "cass-nic-2" -y linux -Q Canonical:UbuntuServer:15.10:15.10.201604050 -u username -M id_rsa.pub -z Standard_D2_v2 --vnet-name cassandra-net --vnet-subnet-name cass-sub
azure vm disk attach-new "cassandra-group" cass02 100 cass-data-02

azure vm create cassandra-group cass03 "West Europe" --nic-name "cass-nic-3" -y linux -Q Canonical:UbuntuServer:15.10:15.10.201604050 -u username -M id_rsa.pub -z Standard_D2_v2 --vnet-name cassandra-net --vnet-subnet-name cass-sub
azure vm disk attach-new "cassandra-group" cass03 100 cass-data-03
{% endhighlight %} 

Our vms are ready to use, but we need to use ssh to install all software and configure it.
We will use fabric automation and to use it we have to create fabfile.py. Our tasks from fabfile need to know internal and external ips of created vms. Thanks to azure cli json output it is easy to do. 
{% highlight bash %} 
azure network public-ip list "cassandra-group" --json > nics.json
azure network nic list cassandra-group --json > adapters.json
{% endhighlight %} 
Fabfile.py is a python script. And we need to install pyyaml lib for cassandra's config file edit.
{% highlight bash %} 
pip install pyyaml
{% endhighlight %} 
Now we need to import all needed libs and load info from json files
{% highlight python %} 
from fabric.api import env, run, sudo, put, local
import json
from os.path import expanduser, join
import time
import yaml

home = expanduser("~")
with open('nics.json') as data_file:    
    nics = json.load(data_file)

with open('adapters.json') as data_file:    
    adapters = json.load(data_file)

cluster_ips = [n["ipAddress"].encode("utf8") for n in nics]

public_id_to_ip = {nic["id"]: nic["ipAddress"].encode("utf8") for nic in nics}
public_to_private = {}
for adapter in adapters:
    c = adapter["ipConfigurations"][0]
    private = c["privateIpAddress"].encode("utf8")
    public_id = c["publicIpAddress"]["id"]
    public = public_id_to_ip[public_id]
    public_to_private[public] = private

cluster_private_ips = [public_to_private[ip] for ip in cluster_ips]

print "public_to_private", public_to_private
print "public", ",".join(cluster_ips)
{% endhighlight %} 
Fabric uses roledefs as an identity of a group of vms for a task execution, we need two: one for all cluster nodes and second one for only one node of a cluster for cqlsh commands. Also we need to specify ssh user and key file.
{% highlight python %} 
env.roledefs.update({
    'cassandra': cluster_ips,
    'cassandra_any' : [cluster_ips[0]],
})
env.key_filename = [join(home, '.ssh/id_rsa')] 
env.user = 'username'  
env.shell = '/bin/bash -c'
{% endhighlight %} 
We are ready to a first tasks: java, jna installation and java version check.
{% highlight python %} 
def install_java():
    sudo('apt-add-repository ppa:webupd8team/java -y ') 
    sudo('apt-get update')  
    sudo("echo debconf shared/accepted-oracle-license-v1-1 select true | \
  sudo debconf-set-selections")
    sudo("echo debconf shared/accepted-oracle-license-v1-1 seen true | \
  sudo debconf-set-selections")
    sudo('apt-get install oracle-java8-installer -y')

def install_jna():
    sudo('apt-get install libjna-java -y ')

def java_version():
    run('java -version')
{% endhighlight %} 
Next step is pyyaml and pip installation.
{% highlight bash %} 
def install_pyyaml():
    sudo("apt-get install python-pip -y")
    sudo('pip install pyyaml')
{% endhighlight %} 
We have attached disk to every vm and we have to partition it, format and mount.
{% highlight python %} 
def add_disk():
    sudo('(echo o; echo n; echo p; echo 1; echo ; echo; echo p; echo w) | fdisk /dev/sdc')
    sudo('mkfs -t ext4 /dev/sdc1')
    sudo('mkdir /mnt/datadrive')
    sudo('mount /dev/sdc1 /mnt/datadrive')
    sudo('echo "UUID=$(blkid -s UUID -o value /dev/sdc1) /mnt/datadrive ext4 defaults,noatime,nodiratime,nobootwait 0 0" | sudo tee -a /etc/fstab')
    sudo('e2label /dev/sdc1 /mnt/datadrive')
    sudo('mkdir -p /mnt/datadrive/lib/cassandra')
    sudo('chown -R cassandra:cassandra /mnt/datadrive/lib/cassandra')
    sudo('mkdir -p /var/lib/cassandra')
    sudo('chown -R cassandra:cassandra /var/lib/cassandra')
    sudo('mkdir -p /var/log/cassandra')
    sudo('chown -R cassandra:cassandra /var/log/cassandra')
{% endhighlight %} 
install cassandra
{% highlight python %} 
def install_cassandra():
    sudo('echo "deb http://www.apache.org/dist/cassandra/debian 30x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list')
    sudo('echo "deb-src http://www.apache.org/dist/cassandra/debian 30x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list')
    sudo("gpg --keyserver pgp.mit.edu --recv-keys F758CE318D77295D")
    sudo("gpg --export --armor F758CE318D77295D | sudo apt-key add -")
    sudo("gpg --keyserver pgp.mit.edu --recv-keys 2B5C1B00")
    sudo("gpg --export --armor 2B5C1B00 | sudo apt-key add -")
    sudo("gpg --keyserver pgp.mit.edu --recv-keys 0353B12C")
    sudo("gpg --export --armor 0353B12C | sudo apt-key add -")
    sudo("apt-get update")
    sudo("apt-get install cassandra -y")
    
def install_cassandra_dsc():
    sudo('echo "deb http://debian.datastax.com/community stable main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list')
    sudo('curl -L http://debian.datastax.com/debian/repo_key | sudo apt-key add -')
    sudo("apt-get update")
    sudo("apt-get install dsc30 -y")
{% endhighlight %} 
For cassandra configuration we have to write a simple script which will allow us to read and write cassandra settings in /etc/cassandra/cassandra.yaml

Let’s name it yaml_option.py
{% highlight python %} 
#!/usr/bin/python
import sys
import yaml

def read_val(cpath, setting):
    with open(cpath, 'r') as stream:
        y = yaml.load(stream)
        if setting in y:
            return y[setting]
        else:
            return ""

def write_val(cpath, setting, v):
    with open(cpath, 'r') as stream:
        y = yaml.load(stream)
    if v is None:
        if setting in y:
            print "deleting", setting
            del y[setting]
    else:
        print "setting: ", v
        y[setting] = v
    with open(cpath, 'w') as stream:
        yaml.dump(y, stream)

if __name__ == "__main__":
    if len(sys.argv) == 4:
        cpath, opt, val = sys.argv[1:]
        print "current: %s" % read_val(cpath, opt)
        #print "setting raw: ", val
        if val == "''\n":
            write_val(cpath, opt, None)
        else:
            write_val(cpath, opt, yaml.load(val))
    else:
        cpath, opt = sys.argv[1:]
        print "current: %s" % read_val(cpath, opt)
{% endhighlight %} 
Now in fabfile.py we will add tasks for this script.
{% highlight python %} 
def copy_yaml_opt():
    put('yaml_option.py', '/home/username/yaml_option.py')

def set_cassandra_option(option, value):
    if isinstance(value, unicode):
        value = value.encode('utf-8')
    value = yaml.dump(value)
    sudo('python /home/username/yaml_option.py /etc/cassandra/cassandra.yaml "%s" "%s"' % (option, value))
{% endhighlight %} 
Just some helper tasks for cassandra
{% highlight python %} 
def cassandra_status():
    #sudo('service cassandra status')
    sudo('nodetool status')

def cassandra_start():
    sudo("service cassandra start")

def cassandra_stop():
    sudo("service cassandra stop")

def cassandra_restart():
    sudo("service cassandra restart")

def cassandra_log():
    sudo("tail /var/log/cassandra/system.log")
{% endhighlight %} 
Andour cassandra configuration.
{% highlight python %} 
seed_provider = [{'class_name': 'org.apache.cassandra.locator.SimpleSeedProvider', 
                  'parameters': [{'seeds': ",".join(cluster_private_ips[:3])}]}]

def set_cluster_config():
    sudo("cp /etc/cassandra/cassandra.yaml /home/username/cassandra.yaml.%d.bkp" % time.time())
    set_cassandra_option("cluster_name", "ClusterName")
    set_cassandra_option("seed_provider", seed_provider)
    set_cassandra_option("listen_interface", "eth0")
    set_cassandra_option("start_rpc", True)
    set_cassandra_option("rpc_interface", "eth0")
    set_cassandra_option("broadcast_rpc_address", public_to_private[env.host_string])#
    set_cassandra_option("endpoint_snitch", "GossipingPropertyFileSnitch")
    set_cassandra_option("data_file_directories", ["/mnt/datadrive/lib/cassandra/data"])
    set_cassandra_option("rpc_server_type", "hsha")
    set_cassandra_option("rpc_address", "")#remove
    set_cassandra_option("listen_address", "")#remove
    set_cassandra_option("rpc_max_threads", 2048)
{% endhighlight %} 
Also after cassandra config we have to setup keyspace, schema, authorization.
{% highlight python %} 
def cassandra_cqlsh(cmd, user = None, password = None):
    ip = public_to_private[env.host_string]
    cmd = "cqlsh -e \"%s\"" % cmd
    if not user is None:
        cmd += " -u %s -p %s" % (user, password)
    cmd += " %s" % ip
    run(cmd)

def cassandra_run_cql_file(path, rc, user = None, password = None):
    ip = public_to_private[env.host_string]
    cmd = "cqlsh -f \"%s\"" % path
    if not user is None:
        cmd += " -u %s -p %s" % (user, password)
    cmd += " --cqlshrc=%s" % rc

    cmd += " %s" % ip
    run(cmd)

def snapshot_keyspace(name):
    run("nodetool -h localhost -p 7199 snapshot %s" % keyspace)

def cassandra_create_user(user,password, id_admin = False):
    if id_admin:
        print "creating super user"
        cassandra_cqlsh("CREATE USER %s WITH PASSWORD '%s' SUPERUSER" % (user,password))
    else:
        cassandra_cqlsh("CREATE USER %s WITH PASSWORD '%s'" % (user,password))

def cassandra_set_pass_auth():
    set_cassandra_option("authenticator", "PasswordAuthenticator")
    set_cassandra_option("authorizer", "CassandraAuthorizer")
    cassandra_restart()
    
def cassandra_create_admin(user, password):    
    cassandra_cqlsh("CREATE USER %s WITH PASSWORD '%s' SUPERUSER" % (user, password), "cassandra", "cassandra")
    cassandra_cqlsh("DROP USER cassandra", user, password)
    
def drop_schema(user, password, keyspace):    
    cassandra_cqlsh("DROP KEYSPACE IF EXISTS %s" % keyspace, user, password)

def cassandra_set_schema(user, password):
    put('cqlshrc', '/home/username/.cassandra/cqlshrc')
    put('schema.cql', '/home/username/schema.cql')
    cassandra_run_cql_file("/home/username/schema.cql", "/home/username/.cassandra/cqlshrc", user, password)
{% endhighlight %} 
As you can see for cqlsh we use rc file cqlshrc.
{% highlight bash %} 
[connection]
client_timeout = 30
{% endhighlight %} 
And our schema.cql is just a cql file with a keyspace and tables creation. 
And final deploy task. 
{% highlight python %}
def deploy():
    install_java()
    install_jna()
    install_pyyaml()
    install_cassandra()
    add_disk()
    copy_yaml_opt()
    set_cluster_config()
    cassandra_fix_owner()
    cassandra_restart() 
{% endhighlight %} 
now we could deploy everything from a command line.
{% highlight bash %}
fab -R cassandra deploy
fab -R cassandra casssandra_status
fab -R cassandra cassandra_set_pass_auth
fab -R cassandra_any cassandra_create_admin:user,password
fab -R cassandra_any cassandra_set_schema:user,password
{% endhighlight %} 

Now our cluster is ready to use. You could use public ips as the cluster address.


# Recommended reading: 
1. [Setting up a Load-Balanced Cassandra Cluster v3](http://blog.z-proj.com/setting-up-a-cassandra-cluster-on-azure/) 
2. [Running Cassandra with Linux on Azure](https://github.com/Azure/azure-content/blob/master/articles/virtual-machines/virtual-machines-linux-classic-cassandra-nodejs.md)
