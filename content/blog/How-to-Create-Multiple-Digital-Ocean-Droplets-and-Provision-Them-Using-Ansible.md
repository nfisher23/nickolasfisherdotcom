---
title: "How to Create Multiple Digital Ocean Droplets and Provision Them Using Ansible"
date: 2019-06-16T17:14:49
draft: false
tags: [ansible, DevOps, digital ocean]
---

In a previous post, we saw [how to create a digital ocean droplet and provision it with Ansible](https://nickolasfisher.com/blog/How-To-Create-a-Digital-Ocean-Droplet-and-Provision-It-Using-Ansible). Creating multiple droplets is very similar, you mostly just have to pay attention to the response object that you get back, which is different in the single vs. the many case.

### Creating Multiple Droplets

If you&#39;re using Ansible &lt; 2.8, to create multiple droplets you will first have to set your digital ocean api token as an environment variable:

```bash
$ export DO_API_TOKEN=1234YourTokenHere4321

```

We can then structure our droplet creation playbook like so:

```yaml
---
- hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: create two droplets
      digital_ocean:
        unique_name: yes
        region_id: ams3
        image_id: ubuntu-18-10-x64
        wait_timeout: 100
        wait: yes
        name: &#34;{{ item }}&#34;
        size_id: s-1vcpu-1gb
        state: present
        command: droplet
        ssh_key_ids: [ &#39;&#39; ] # &lt;---- put your numeric ssh key in here
      register: created_droplets
      with_items:
        - tmp-droplet-1
        - tmp-droplet-2

    - name: add to dynamic inventory
      add_host:
        name: &#34;{{ item.droplet.ip_address }}&#34;
        group: do
      with_items: &#34;{{ created_droplets.results }}&#34;

```

If you&#39;re using ansible 2.8 or greater, that playbook beginning can instead look like:

```yaml
---
- hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: create two droplets
      digital_ocean_droplet:
        unique_name: yes
        region: ams3
        image: ubuntu-18-10-x64
        wait_timeout: 100
        name: &#34;{{ item }}&#34;
        size_id: s-1vcpu-1gb
        state: present
        ssh_keys: [ &#39;&#39; ] # &lt;---- put your numeric ssh key in here
      register: created_droplets
      with_items:
        - tmp-droplet-1
        - tmp-droplet-2

    - name: add to dynamic inventory
      add_host:
        name: &#34;{{ item.data.ip_address }}&#34;
        group: do
      with_items: &#34;{{ created_droplets.results }}&#34;

```

Before you run this, you&#39;ll want to make sure to include an **ansible.cfg** file that looks like this:

```
[defaults]
host_key_checking = False
```

This will not prompt you before connecting via SSH to your newly created droplets. In a development environment where servers are ephemeral, this is preferred, and can further aid automation so that new droplets can be created without human intervention.

To prove this out, as we did in the last post, we can provision each server with nginx and a custom index page.
Create an **index.html.j2** Jinja2 template in the same directory as your playbook and fill it with:

```html
&lt;h1&gt; On a digital ocean droplet now &lt;/h1&gt;

&lt;p&gt; The ip address where we&#39;re at is: {{ ansible_default_ipv4.address }} &lt;/p&gt;
```

And round out the playbook with:

```yaml
- hosts: do
  remote_user: root
  gather_facts: false

  vars:
    ansible_python_interpreter: /usr/bin/python3

  tasks:
    - name: wait for port 22 to become available
      wait_for:
        host: &#34;{{ inventory_hostname }}&#34;
        port: 22
      delegate_to: localhost

    - name: Gather Facts
      setup:

    - name: install nginx
      apt:
        name: nginx

    - name: modify html file
      template:
        src: ./index.html.j2
        dest: /var/www/html/index.html

```

You should now be able to navigate to your IP addresses and see the home page with the IP address displayed. Go get &#39;em, tiger.