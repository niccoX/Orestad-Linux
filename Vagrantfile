# -*- mode: ruby -*-
# vi: set ft=ruby :#

# Install required plugin(s) and re-execute
required_plugins = %w( vagrant-vbguest vagrant-hostmanager)
required_plugins.each do |plugin|
    exec "vagrant plugin install #{plugin};vagrant #{ARGV.join(" ")}" unless Vagrant.has_plugin? plugin || ARGV[0] == 'plugin'
end

Vagrant.configure("2") do |config|

  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  config.vm.synced_folder "./www", "/var/www/html", type: "virtualbox", owner: "www-data", group: "www-data"

  config.vm.define "wordpress", primary: true do |wordpress|

    wordpress.vm.box = "xcoo/bionic64"
    wordpress.vm.hostname = "www.malmonaringslivsgala.se"
    wordpress.vm.network "private_network", type: "dhcp"

    wordpress.trigger.after [:up, :reload] do |trigger|
      trigger.run = {inline: "vagrant hostmanager"}
    end

    # Fix me
    #wordpress.trigger.after [:up] do |trigger|
    #  trigger.run = {inline: "rsync -a --exclude '*.zip' --exclude '*.gz' -e 'ssh -o StrictHostKeyChecking=no' malmonlg@indra.nmugroup.com:/home/malmonlg/public_html/ ./www/ --progress"}
    #end

    wordpress.hostmanager.ip_resolver = proc do
      `vagrant ssh -c 'hostname -I'`.split()[1]
    end

    wordpress.vm.provider :virtualbox do |vb|
      vb.name = "mnlg.local"
      vb.cpus = 2
      vb.memory = 2048
    end
    wordpress.vbguest.auto_update = false
    wordpress.vbguest.no_remote   = true

    wordpress.hostmanager.aliases = %w(malmonaringslivsgala.se)

    wordpress.vm.provision "shell", run: "once" do |shell|
      shell.inline = <<-SHELL
        echo 'Installing pre-requisites...'
        apt-get -qq update > /dev/null && apt-get -qq install -y software-properties-common > /dev/null
        add-apt-repository -y ppa:ondrej/php && apt-get -qq update > /dev/null
        apt-get -qq install -y mariadb-server-10.1 mariadb-client-10.1 apache2 php7.0-fpm php7.0-mysql php7.0-mbstring libapache2-mod-php7.0 > /dev/null
        a2enmod proxy_fcgi setenvif rewrite > /dev/null && a2enconf php7.0-fpm > /dev/null
        a2enmod ssl > /dev/null
	sed -i s/AllowOverride\ None/AllowOverride\ All/g /etc/apache2/apache2.conf
        systemctl restart apache2 > /dev/null
        a2ensite default-ssl.conf
        systemctl reload apache2
        #rm -rf /var/www/html
	#mkdir /var/www/html
	#mount -o bind /vagrant/www /var/www/html
        #mkdir -p /home/malmonlg/public_html
        #mount -o bind /vagrant/www /home/malmonlg/public_html
        curl -sSL https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar -o /usr/local/bin/wp && chmod +x /usr/local/bin/wp
        echo "Importing database..."
        ARRAY=
        mapfile -t ARRAY < <(grep -e DB_NAME -e DB_USER -e DB_PASSWORD -e DOMAIN_CURRENT_SITE /vagrant/www/wp-config.php | cut -d ',' -f 2 | sed s/\\ \\'/\\`/ | sed s/\\'.*/\\`/)
        DBPASSWORD=$(echo ${ARRAY[2]} | sed s/\\`/\\'/g)
        mysql -sNe "grant all privileges on ${ARRAY[0]}.* to ${ARRAY[1]} identified by ${DBPASSWORD};"
        if [ ! /vagrant/${ARRAY[0]}.sql ] ; then
          echo "Need database dump file ${ARRAY[0]}.sql in the same directory as Vagrantfile to continue."
          exit 1
        fi
        DB=$(echo ${ARRAY[0]} | sed s/\\`/\\/g)
        cd /vagrant/www && $(wp db create --allow-root > /dev/null 2>&1 || sleep 1) && wp db import /vagrant/${DB}.sql --allow-root
        # Fixme
	#echo "Replacing URLs..\nThis will take a while.." 
	#cd /vagrant/www && wp search-replace --url=www.malmonaringslivsgala.se '^(https?:\/\/)(www\.)?(malmonaringslivsgala\.se)(.*)$' 'http://$2mnlg.local$4' --regex --all-tables --precise --network --allow-root
        #echo "Creating a backup of wp-config.php before changes..."
        #cp /vagrant/www/wp-config.php /vagrant/wp-config.php.bkp
        #sed -i s/www.malmonaringslivsgala.se/www.mnlg.local/g /vagrant/www/wp-config.php
      SHELL
    end
  end
end
