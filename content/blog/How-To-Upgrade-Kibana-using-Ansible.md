---
title: "How To Upgrade Kibana using Ansible"
date: 2019-03-23T21:14:22
draft: false
tags: [vagrant, ansible, the elastic stack, DevOps, molecule]
---

You can view the sample code associated with this post [on GitHub](https://github.com/nfisher23/some-ansible-examples).

In a previous post on [Provisioning a Server with Kibana](https://nickolasfisher.com/blog/How-to-Provision-a-Linux-VM-With-Kibana-Using-Ansible), we saw that it&#39;s very straightforward to get kibana on a box.

Upgrading Kibana is also very straightforward (and nowhere near as complicated as [upgrading elasticsearch](https://nickolasfisher.com/blog/How-to-do-a-Rolling-Upgrade-of-an-Elasticsearch-Cluster-Using-Ansible)). That will be the subject of this post.

First, initialize the ansible role using molecule, with vagrant as the VM provider:

```bash
$ molecule init role -r upgrade-kibana -d vagrant
```

Then modify your **molecule/default/molecule.yml** file to look like:

```yaml
platforms:
  - name: kibana
    box: ubuntu/xenial64
    memory: 4096
    provider_raw_config_args:
    - &#34;customize [&#39;modifyvm&#39;, :id, &#39;--uartmode1&#39;, &#39;disconnected&#39;]&#34;
    interfaces:
    - auto_config: true
      network_name: private_network
      ip: 192.168.56.121
      type: static
```

We can bring in work from the previous post and include a dependency on our previous role to ensure Kibana is already there. Modify your **meta/main.yml** file to look like:

```yaml
---
dependencies:
  - role: install-kibana
```

You should now be able to enter:

```bash
$ molecule create &amp;&amp; molecule converge
```

And see kibana come up at 192.168.56.121:5601.

### Upgrading

We can now begin the upgrade process. We will follow a similar pattern to [upgrading logstash](https://nickolasfisher.com/blog/How-to-do-a-Rolling-Upgrade-of-Multiple-Logstash-Instances-Using-Ansible) and [upgrading elasticsearch](https://nickolasfisher.com/blog/How-to-do-a-Rolling-Upgrade-of-an-Elasticsearch-Cluster-Using-Ansible) by adding another collection of tasks to perform the upgrade when we see fit. Change your **tasks/main.yml** file to look like:

```yaml
---
# tasks file for upgrade-kibana
- include: upgrade_kibana.yml
  when: upgrade_kibana
```

Then add your **tasks/upgrade\_kibana.yml** file:

```yaml
- name: ensure kibana already up and running
  service:
    name: kibana
    state: started
  become: yes

- name: get kibana deb to dl
  get_url:
    dest: &#34;/etc/{{ kibana_deb_to_upgrade_to }}&#34;
    url: &#34;https://artifacts.elastic.co/downloads/kibana/{{ kibana_deb_to_upgrade_to }}&#34;
    checksum: &#34;sha512:https://artifacts.elastic.co/downloads/kibana/{{ kibana_deb_to_upgrade_to }}.sha512&#34;
  become: yes

- name: stop kibana
  service:
    name: kibana
    state: stopped
  become: yes

- name: upgrade
  apt:
    deb: &#34;/etc/{{ kibana_deb_to_upgrade_to }}&#34;
  become: yes

- name: start upgraded kibana
  service:
    name: kibana
    state: started
  become: yes
```

We will need a couple of parameters, which I&#39;ve elected to keep in the **defaults/main.yml** file:

```yaml
---
# defaults file for upgrade-kibana
upgrade_kibana: true
kibana_deb_to_upgrade_to: kibana-6.5.3-amd64.deb
```

With this, you should be able to run:

```bash
$ molecule converge
```

Note that [there are some breaking changes to the upgrade to 6.5.](https://www.elastic.co/guide/en/kibana/current/release-notes-6.5.0.html#known-issues-6.5.0) Given that this is an Elastic product, I&#39;m not surprised one bit. You will have to address those if, like this example, you upgrade to 6.5 or greater from a &lt; 6.5 version.