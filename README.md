Ansible role that allows for quickly deploying  a redirector to an existing server with mod_rewrite proxy rules, 

Supports Debian and Ubuntu, tested in Digital ocean and Azure 

See threat.tevora.com/automating-redirector-deployment-with-ansible for a blog walking through redirectors, ansible, and a deep dive on this role

To get started, clone this repo, install ansible, and place this repo in your roles folder. See sample playbook below for an example of how to build your redirector instance config 

**Instance Playbook Sample provision_redirector_example.yml**
```language-yaml
- hosts: EnigmaticEmu
  gather_facts: False
  user: root
  pre_tasks:
  - name: Install python for Ansible
    raw: test -e /usr/bin/python || (apt -y update && apt install -y python-minimal)
    changed_when: False
  - setup: # aka gather_facts 
  tasks: 
  - include_role:
      name: redirectors
    vars:
      le_email: 'threat@tevora.com'
      hop_dir: hops/empire_hop
      vhosts: [
        {
          servername: 'fakeamazon.com',
          http_port: 80,
          https_port: 443,
          c2filters: [
            {
              rewritefilter: '^/orders/track/?$',
              host: '123.124.125.126'
            }
          ],
          configs: [ 
            'RewriteRule !\.php$ https://www.amazon.com/%{REQUEST_URI} [L,R=302]'
          ]
        },
 {
          servername: 'fakegoogle.com',
          http_port: 80,
          https_port: 443,
          config_files: [
            "redirectors.txt",
            "apache_tweaks.conf"
          ],
        }
      ] 
```

Breakdown of the example: 

 * `- hosts: EnigmaticEmu`
   This specifies the hosts the playbook will execute on.
   For this playbook to execute correclty, an inventory file must be used with the hostname or ip of engigmatic emu defined  
   
 *   `gather_facts: False`
   The ansible gather facts task breaks on newer versions of ubuntu with no python 2 installed
   we disable gather facts,  so we can make sure to install python 2 first. This is just some compatability bootstrap stuff

* `  user: root`
   this specifies what user to logon to the remote server with, ansible performs all its configurations over ssh
   Change this to whatever user you want to use
  *if you need to sudo, may need to add `become: true` and call ansible-playbook with `--ask-become-pass`

* `pre_tasks:` 
   * these lines install python 2 if it is not there and run gather facts
 `  tasks: ` Now we've reached the good part, our playbook tasks! 

* `- include_role:`
   as you may imagine, this specfies a role to include, and the next line `name: redirectors` specifies to include our redirectors role
 
 * `vars:`
   here is where we do the meat of our config 
   Our server contains three vars,le_email, hop_dir, and vhosts
   Le email is used to specify the email address we use for lets_encrypt, be sure to use one you control
   remember the hop_dir we used in the copy task of our role? Here is where it gets defined! The hop dir should located in ./files/ relative to the location of the playbook itself
  
 * `hosts: [...]`
   our vhosts variable is a list of vhost config dictionaires. This allows us to setup multiple vhosts with different domains and C2 profiles per server. 
   Remember that this variable is a list, as it will be important in how we template our config files and other ways we use these variables in our redirectors role
   
 * `servername: 'fakeamazon.com'` 
   this is the hostname of our server, and you MUST have a dns record for this hostname pointing at the IP of the server. 
  
 * `http_port` and `https_port` are pretty self explanatory. Choose what ports http and https will listen on
 
 * notice before we go on we have multiple ways to define mod rewrite rules in the config. This is largely due to experimentation, and plans to integrate this playbook with a python API
 * `c2filters: [...]`
   a list of c2filter dicts
   Any request whose URI matches `rewritefilter` will proxy the connection to `host`
  
 * `configs[...]`
   a list of config strings
   Each string will be added to the file.  
  
 * `config_files[...]` filename or path
   list of filenames or paths. The content of each file will be added to aache
   usefull for leveraging mod_rewrite automation tool output such as @Inspired-Secs awesome tool: https://blog.inspired-sec.com/archive/2017/04/17/Mod-Rewrite-Automatic-Setup.html 

We formatted this config mostly in JSON (YAML is a superset of JSON) but you can format it however you like as long as it matches up. 

Notice in the config how there are multiple vhosts, and each one can use one or more methods of specifying how it is injecting into the configuration templates 

 We create one configuration file per vhost is that Letsencrypt, specifically the certbot-apache component, does not support more than one vhost per config file. Because of this we will be provisioning multiple configuration files to the server. 
 
**Running the Playbook**

To run the role. Create your playbook in the form of the example we covered and run `ansible-playbook -i <your_hosts_file> <your_playbook>`. Ensure that this roles is in the roles directory in the same path of your playbook, and your hop and/or config files are placed correctly.  Your directory layout should look like: 
```
├── my_playbook.yml
├── files
│   └── empire_hop
│       └── news
│           └── login.php
└── roles
    ├── redirectors
    │   ├── files
    │   ├── handlers
    │   │   └── main.yaml
    │   ├── meta
    │   ├── tasks
    │   │   ├── apache.yml
    │   │   ├── letsencrypt.yml
    │   │   └── main.yml
    │   ├── templates
    │   │   ├── apache_sslvhost.conf.j2
    │   │   └── apache_vhost.conf.j2
    │   └── vars
```
