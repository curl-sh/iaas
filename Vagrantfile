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
    df | grep /dev/sdb1 || \
      echo ",,,," | sfdisk -uS /dev/sdb && \
      mkfs.ext4 /dev/sdb1 -L var-lib-docker && \
      echo "LABEL=var-lib-docker /var/lib/docker ext4 defaults 0 0" >> /etc/fstab && \
      mkdir -p /var/lib/docker && \
      mount -L var-lib-docker

    command -v docker || curl -sL https://get.docker.com/ | sh && \
    usermod -aG docker vagrant
  SHELL
end