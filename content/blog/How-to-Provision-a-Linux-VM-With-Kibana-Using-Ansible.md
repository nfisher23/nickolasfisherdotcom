---
title: "How to Provision a Linux VM With Kibana Using Ansible"
date: 2019-03-16T15:37:40
draft: false
tags: [distributed systems, vagrant, ansible, the elastic stack, DevOps, molecule]
---

The corresponding source code for this post is available [on GitHub](https://github.com/nfisher23/some-ansible-examples).

[Kibana](https://www.elastic.co/products/kibana) is a fancy pants web application that tries to make data in Elasticsearch user-friendly. Rounding out the previous two posts on [how to install an elasticsearch cluster](https://nickolasfisher.com/blog/how-to-provision-a-multi-node-elasticsearch-cluster-using-ansible) and [how to install multiple logstash hosts](https://nickolasfisher.com/blog/how-to-install-multiple-logstash-hosts-using-ansible), I will now show you how to stack kibana on top of them.

### Create the Ansible Role

Navigate to the directory you want the ansible role to reside and type:

```bash
$ molecule init role -d vagrant -r install-kibana
```

I'm using molecule to wrap vagrant and I'm calling this role install-kibana.

We will put kibana on a single host, 192.168.56.121. To make this happen, adjust the _platforms_ section of your **molecule/default/molecule.yml** file to look like:

```yaml
platforms:
  - name: kibana
    box: ubuntu/xenial64
    memory: 4096
    provider_raw_config_args:
    - "customize ['modifyvm', :id, '--uartmode1', 'disconnected']"
    interfaces:
    - auto_config: true
      network_name: private_network
      ip: 192.168.56.121
      type: static

```

We provision this local VM with 4GB of RAM. You can bring up this VM at this point with:

```bash
$ molecule create
```

First, we'll decide on the version of kibana we want to provision. To keep this compatible with the two posts mentioned above, we'll choose version 6.4.0, and update the **vars/main.yml** file to reflect the full name of the deb file we'll be grabbing in our playbook like so:

```yaml
---
# vars file for install-kibana
kibana_deb_file: kibana-6.4.0-amd64.deb
```

Our **tasks/main.yml** file can now look like:

```yaml
---
# tasks file for install-kibana
- name: download deb file
  get_url:
    dest: "/etc/{{ kibana_deb_file }}"
    url: "https://artifacts.elastic.co/downloads/kibana/{{ kibana_deb_file }}"
    checksum: "sha512:https://artifacts.elastic.co/downloads/kibana/{{ kibana_deb_file }}.sha512"
  become: yes

- name: install kibana from deb file
  apt:
    deb: "/etc/{{ kibana_deb_file }}"
    update_cache: yes
  become: yes

- name: send kibana config file
  template:
    dest: /etc/kibana/kibana.yml
    src: kibana.yml.j2
  become: yes
  notify: restart kibana
```

You can see we're using a handler that restarts kibana, which is in the **handlers/main.yml** file like so:

```yaml
---
# handlers file for install-kibana
- name: restart kibana
  service:
    name: kibana
    state: restarted
  become: yes

```

Finally, we will have to create a **templates/kibana.yml.j2** template, which as of right now is just a simple file:

```yaml
elasticsearch.url: "http://192.168.56.102:9200"
server.host: 0.0.0.0

```

Keeping it as a template makes extending it and extracting variables as the role gets more involved possible.

You should now be able to run:

```bash
$ molecule converge
```

And the playbook will install kibana successfully. Once it comes up, you should navigate to [http://192.168.56.121:5601,](http://192.168.56.121:5601,) and you'll see kibana's home page. If you don't have any backing elasticsearch database, then you'll see an error, but bring on up at the port in the kibana.yml.j2 template and that error will go away.
