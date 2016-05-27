# -*- mode: ruby -*-
# vi: set ft=ruby :
# based on project localserver on vagrant http://ilyar.github.io/localserver/

# Project settings for customize

custom_vm_os         = false # default 'precise' or 'trusty'
custom_project_name  = false # default from project name folder
custom_project_host  = false # default `project_name.local`
custom_project_alias = false # default `www.project_name.local`
custom_ip_address    = false # IP Address IPv4 private network range
custom_vm_arch       = false # default 32  or VM_ARCH
custom_vm_memory     = false # default 512 or VM_MEMORY
custom_vm_cores      = false # default 1   or VM_CORES

# Development script any command for run `vagrant provision`
$server_script = <<SCRIPT
SCRIPT

# Check script
$check = <<SCRIPT
    lsb_release -a
    apachectl -V
     mysql --version
    php -v
    php -m
SCRIPT

# General project settings for customize

project_name = $custom_project_name || File.basename(File.dirname(__FILE__))
project_host = $custom_project_host || "#{project_name}.local"
project_host_alias = $custom_project_host_alias || "www.#{project_name}.local"
ip_address = $custom_ip_address || "10.10.10.10"
hostmanager_aliases = ["#{project_host_alias}"]

# Environment variables

OS = $custom_vm_os || ENV['VM_OS'] || 'precise'
ARCH = $custom_vm_arch || ENV['VM_ARCH'] || 32
MEMORY = $custom_vm_memory || ENV['VM_MEMORY'] || 512
CORES = $custom_vm_cores || ENV['VM_CORES'] || 1

# Virtual host setup

$virtual_host = <<SCRIPT

    echo "Setup: virtual host..."
    echo "<VirtualHost *:80>
      ServerName #{project_host}
      ServerAlias #{$project_host_alias}

      DocumentRoot /var/www/#{project_name}

      <Directory /var/www/#{project_name}>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
      </Directory>

      # Available loglevels: trace8, ..., trace1, debug, info, notice, warn, error, crit, alert, emerg.
      LogLevel info
      ErrorLog  /var/log/apache2/#{project_name}-error.log
      CustomLog /var/log/apache2/#{project_name}-access.log combined
    </VirtualHost>" > /etc/apache2/sites-available/#{project_name}.conf

    a2ensite #{project_name}

    # Restart Services
    service apache2 restart

SCRIPT

# Setup script
$script = <<SCRIPT

    # Update the server
    echo "Update server..."
    apt-get update -qq && apt-get upgrade -qq -y --fix-missing

    # Install Apache
    apt-get install -qq -y apache2

    # Configure Apache
    echo "ServerName #{project_host}" | sudo tee /etc/apache2/conf-available/fqdn.conf
    sudo ln -s /etc/apache2/conf-available/fqdn.conf /etc/apache2/conf-enabled/fqdn.conf
    a2enmod rewrite
    a2dissite 000-default

    # Install PHP
    apt-get install -qq -y php5 php5-cli php5-curl php5-gd php5-mysql php5-xsl php5-xdebug

    # Configure xdebug
    echo "xdebug.remote_enable=on
    xdebug.remote_connect_back=on
    xdebug.remote_autostart=on
    xdebug.remote_handler=dbgp
    xdebug.remote_host=localhost
    xdebug.remote_port=9001
    xdebug.remote_mode=req
    xdebug.idekey=vagrant" > /etc/php5/apache2/conf.d/21-xdebug.ini > /etc/php5/cli/conf.d/21-xdebug.ini

    # Set timezone
    echo date.timezone = Etc/GMT > /etc/php5/apache2/conf.d/20-date.ini > /etc/php5/cli/conf.d/20-date.ini

    # Install MySQL
    echo mysql-server     mysql-server/root_password        password local | debconf-set-selections
    echo mysql-server     mysql-server/root_password_again  password local | debconf-set-selections
    apt-get -yq install mysql-client mysql-server

    # Create database
    mysql -u root --password=local -e "create database if not exists umi CHARACTER SET utf8 COLLATE utf8_general_ci"
    mysql -u root --password=local -e "grant all on umi.* to 'umi'@'%' identified by 'umi'"

    # Allow MySQL root access from any host
    sed -i 's/^bind-address[[:blank:]]*=[[:blank:]]*127.0.0.1$/bind-address = 0.0.0.0/g' /etc/mysql/my.cnf
    echo "GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'local' WITH GRANT OPTION" | mysql -u root --password=local
    echo "GRANT PROXY ON ''@'' TO 'root'@'%' WITH GRANT OPTION" | mysql -u root --password=local

    # Install phpmyadmin
    echo phpmyadmin       phpmyadmin/reconfigure-webserver  text     apache2     | debconf-set-selections
    echo phpmyadmin       phpmyadmin/dbconfig-install       boolean  true        | debconf-set-selections
    echo phpmyadmin       phpmyadmin/app-password-confirm   password local       | debconf-set-selections
    echo phpmyadmin       phpmyadmin/mysql/admin-pass       password local       | debconf-set-selections
    echo phpmyadmin       phpmyadmin/password-confirm       password local       | debconf-set-selections
    echo phpmyadmin       phpmyadmin/setup-password         password local       | debconf-set-selections
    echo phpmyadmin       phpmyadmin/mysql/app-pass         password local       | debconf-set-selections
    apt-get -yq install phpmyadmin

    # Restart Services
    service mysql restart

    # Configure VM user
    adduser vagrant www-data

SCRIPT

# Vagrant configuration
Vagrant.configure("2") do |config|

  # Define VM box to use
  config.vm.box = "ubuntu/#{OS}#{ARCH}"
  # tested on ubuntu/precise32 20160122.0.1 - OK
  # tested on ubuntu/trusty32 20160122.0.1 - ERROR - ISSUE #4

  # Set share folder
  config.vm.synced_folder "./" , "/var/www/#{project_name}", :mount_options => ["dmode=777", "fmode=666"]

  # Config vagrant-cachier
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end
  
  # Provider-specific configuration so you can fine-tune VirtualBox for Vagrant.
  # These expose provider-specific options.
  config.vm.provider :virtualbox do |vm|
    # Use VBoxManage to customize the VM. For example to change memory:
    vm.customize ["modifyvm", :id, "--memory", MEMORY.to_i]
    vm.customize ["modifyvm", :id, "--cpus", CORES.to_i]

    if CORES.to_i > 1
      vm.customize ["modifyvm", :id, "--ioapic", "on"]
    end
  end
  
  # Config vagrant-hostmanager
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.vm.define project_name do |node|
      node.vm.hostname = project_host
      node.vm.network :private_network, ip: ip_address
      node.hostmanager.aliases = hostmanager_aliases
    end
    config.vm.provision :hostmanager
  end

  config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

  if Dir.glob("#{File.dirname(__FILE__)}/.vagrant/machines/#{project_name}/*/id").empty?
    config.vm.provision :shell, :inline => $script
  end

  config.vm.provision :shell, :inline => $virtual_host
  config.vm.provision :shell, :inline => $server_script
  config.vm.provision :shell, :inline => $check

end
