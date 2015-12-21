+++
date = "2015-03-17T20:56:46+05:30"
draft = false
title = "Vagrant & Saltstack Quickstart Tutorial"

+++
This post is translated / inspired from very well written [Vagrant & Ansible Quickstart Tutorial](http://adamcod.es/2014/09/23/vagrant-ansible-quickstart-tutorial.html). I was very much surprised that I couldn't find a similar article which created some thing use full to get started with saltstack. The style of the article is very interesting in that it starts setting up a simple LAMP stack. As the article progresses the requirements for the stack are also changed and more features of ansible are exposed. This is the style I will try to follow.

I professionally use puppet, but am in general more inclined to the python ecosystem. This is one of the reasons that got me interested in Ansible and Saltstack. As I was researching both the frameworks this article [Moving away from Puppet: SaltStack or Ansible?](http://ryandlane.com/blog/2014/08/04/moving-away-from-puppet-saltstack-or-ansible/) comparing them got me leaning towards Saltstack. Since the above article explain why you might want to use Slatstack better than I possibly could, lets just get our hands dirty.

## Salt and Vagrant

Ensure that a recent version(> 1.3.0) of Vagrant is installed, in older version you will have to install salt support to vagrant as a separate plugin. Vagrant will automatically download and install salt on a given machine so you don't need to install it separately.

## Basics

We will start by creating a new directory to hold our project.

    mkdir -p ~/Projects/vagrant-salt
    cd ~/Projects/vagrant-salt

Next, we can use Vagrant to create a new vagrant file based on the latest ubuntu image.

    vagrant init ubuntu/trusty64

You should now have a file called Vagrantfile in the root of the directory. This contains some basic information about the box you want to provision, and then a whole bunch of commented out stuff you don't need to worry about now. Remove all of the commented lines, so you're left with the bare minimum:

```ruby
Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"
end
```
We'll need a way to access our webserver once it's provisioned, so we'll tell Vagrant to forward port 80 from our box to port 8080 on localhost. To do that, add the following line just before end:

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8080
```

Now, there is one last thing we need to do to configure Vagrant, and then we're finished with it. We need to tell Vagrant that we want to use Ansible as its provisioner, and where to find the commands to run. To do this, add the following lines to your `Vagrantfile`, again, just before `end`:

```ruby
  config.vm.synced_folder "salt/roots/", "/srv/salt/"

  config.vm.provision :salt do |salt|
    salt.minion_config = "salt/minion.yml"
    salt.run_highstate = true
    salt.colorize = true
    salt.log_level = 'info'
  end
```

Once you've done that, the entire contents of your `Vagrantfile` should look like this:

```ruby
Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.network "forwarded_port", guest: 80, host: 8080

  config.vm.synced_folder "salt/roots/", "/srv/salt/"

  config.vm.provision :salt do |salt|
    salt.minion_config = "salt/minion.yml"
    salt.run_highstate = true
    salt.colorize = true
    salt.log_level = 'info'
  end
end
```
## Basic Terms

Salt is more similar to *Puppet* than *Ansible* in that it tries to get the server to a specified state that you have described in your configuration. This is usually divided into smaller *SaltStates* that describe a single thing. A *SaltState* tell Salt how to achieve a state like setting up Apache as a service. Inside the state you will be able to specify how to install it using aptitude or what configuration to use etc.

Normally salt runs in *master slave* mode. But here we are going to do master less setup. `Minion` is the terminology used for `slaves` in salt.

## First State

Create a new file `salt/minion.yml` with the following content:

```yaml
master: localhost
file_client: local
```

To install a LAMP stack, there are four basic steps we need to take:

1. Update Apt Cache
2. Install Apache
3. Install MySQL
4. Install PHP
5. Setup A PHP file to serve.

That's kind of all we need to do. We're using an ubuntu box, so all of that can be done via apt. To do that, we need to use SaltStack's `pkg` module. `pkg` module abstracts common package related actions like `install` and `remove` between different system package managers. Any specific commands for a package manager will still be in a specialized salt-state. We also don't need to explicitly call `apt-get update`, that's also internally taken care of by `pkg` module. We will also use `file` module to create a simple `php` file for testing.

Add a file `salt/roots/top.sls`

```yaml
base:
  '*':
  - base
```
Since we are using vagrant and we only have one host its ok to put star `*` on where to run. `- base` is what we are going to run on those servers.

Add a file `salt/roots/base.sls`

```yaml
apache:
  pkg.installed:
    - name: apache2
mysql:
  pkg.installed:
    - name: mysql-server
php:
  pkg.installed:
    - name: php5

/var/www/html/info.php:
  file.managed:
    contents: '<?php phpinfo();'
```

In the terminal `vagarant up` and you should see output similar to the following.

That's it. If all you wanted was a working LAMP server, you now have it.

And load it in your browser at http://localhost:8080/info.php and you should see everything working as expected.

## Refactoring

The next setup for us here is to reduce the repetitiveness of this code. `pkg` and many other salt modules allow you to give a list of item to apply this module to. For example in case of `pkg` we can rewrite the piece like this.

``` yaml
install required packages:
  pkg.installed:
    - names:
      - apache2
      - mysql-server
      - php5

/var/www/html/info.php:
  file.managed:
    contents: '<?php phpinfo();'
```


We reduced three separate states to just one. You can test it works by running `vagrant destroy` followed by another `vagrant up` in the root of your project.

## Modularize

Usually, when you're installing packages for a new server you want to do more than just install the packages - you probably want to configure them too. You might want to tell apache to use `/vargrant` instead of `/var/www/html` (which is the default location for vagrant to mount the current directory), or install `php_mysql` and `php_pdo` so you can access your MySQL Server from PHP.

There are two ways you can split this base module.

## Conculsion

Unlike many other infra automation tools you don't need to know many salt specific terminologies. Salt has very minimal concepts. Unlike `Ansible` there is no special roles. You just split it into modules and arrange the files as you see fit.
