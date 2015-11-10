# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# docker is running on a separate partition
# /var/lib/docker -> /dev/sdb1 (.persistent/var-lib-docker.vdi)
# in this way pulled images are preserved between vagrant-destroy
# and vagrant-up cycle

VAGRANTFILE_API_VERSION = "2"
VAR_LIB_DOCKER = File.join(File.dirname(File.expand_path(__FILE__)), '.persistent', 'var-lib-docker.vdi')

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.hostname = "curl-sh"
  config.ssh.insert_key = false

  config.vm.provider :virtualbox do |v|
    v.name = "curl-sh"
    v.cpus = 2
    v.memory = 1024
    unless File.exist?(VAR_LIB_DOCKER)
      v.customize ['createhd', '--filename', VAR_LIB_DOCKER, '--size', 40 * 1024]
    end
    FileUtils.ln_sf VAR_LIB_DOCKER, VAR_LIB_DOCKER + '.link'
    v.customize ['storageattach', :id, '--storagectl', 'SATAController', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', VAR_LIB_DOCKER + '.link']
  end

  config.vm.provision "install", type: "shell", inline: <<-SHELL
    function tzz() {
      "$@" >/dev/null 2>&1
    }

    echo "update and upgrade"
    [ `expr $(date +%s) - $(stat -c %Y /var/cache/apt/)` -gt 3600 ] && { # last update older than 1 hour
      echo "  ... updating"
      tzz apt-get update
      tzz apt-get upgrade
    }

    echo "install git"
    tzz command -v git || {
      echo "  ... installing"
      tzz apt-get install -y git
    }

    echo "prepare docker"
    df | tzz grep /dev/sdb1 || {
      echo "  ... /var/lib/docker -> /dev/sdb1 (.persistent/var-lib-docker.vdi)"
      tzz echo ",,,," | sfdisk -uS /dev/sdb
      tzz mkfs.ext4 /dev/sdb1 -L var-lib-docker
      echo "LABEL=var-lib-docker /var/lib/docker ext4 defaults 0 0" >> /etc/fstab
      tzz mkdir -p /var/lib/docker
      tzz mount -L var-lib-docker
    }

    echo "install docker"
    tzz command -v docker || {
      echo "  ... curl -sL https://get.docker.com | sh"
      tzz curl -sL https://get.docker.com | sh
      tzz usermod -aG docker vagrant
    }

  SHELL

end