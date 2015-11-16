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
  config.ssh.insert_key = false

  config.vm.box = "ubuntu/trusty64"
  config.vm.hostname = "curl-sh"
  config.vm.boot_timeout = 30

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
    tzz() {
      "$@" >/dev/null
    }

    isOlder() {
      [ -r ${1} -a `expr $(date +%s) - $(stat -c %Y ${1})` -gt ${2:-0} ] && {
        return 0 # true
      }
      return 1 # false
    }

    gh() {
      git_user=$(echo $1 | cut -d '/' -f1)
      git_repo=$(echo $1 | cut -s -d '/' -f2 | sed 's/\.git$//g')
      isAlreadyCloned=0 # true

      [ -L "/usr/local/${git_user}" ] || {
        tzz ln -s /vagrant/.persistent/github.com/${git_user} /usr/local/${git_user}
      }

      src="https://github.com/${git_user}/${git_repo}.git"
      dst="/vagrant/.persistent/github.com/${git_user}/${git_repo}"

      echo "${src} > ${dst}"

      [ -d "${dst}" ] || {
        isAlreadyCloned=1 # false
        echo "  ... cloning"
        tzz mkdir -p ${dst}
        tzz git clone --recursive ${src} ${dst}
      }

      isOlder "${dst}/.git" 3600 && {
        echo "  ... updating"
        tzz git -C ${dst} pull --recurse-submodules
        tzz git -C ${dst} submodule update --recursive
      }
      return $isAlreadyCloned
    }

    echo "update and upgrade"
    isOlder /var/cache/apt 3600 && {
      echo "  ... updating"
      tzz apt-get update
      tzz apt-get -y upgrade
    }

    echo "install git"
    tzz command -v git || {
      echo "  ... installing"
      tzz apt-get install -y git
    }

    echo "install zsh"
    tzz command -v zsh || {
      echo "  ... installing zsh"
      tzz apt-get install -y zsh

      gh kaluzki/prezto || {
        echo "  ... installing prezto"
        echo 'PREZTO_DIR=/usr/local/kaluzki/prezto' >> /etc/zsh/zshenv
        echo 'PREZTORC="\${PREZTO_DIR}/runcoms/zpreztorc"' >> /etc/zsh/zshenv
        for file in zshenv zprofile zshrc zlogin; do
          echo '[ -s "\${PREZTO_DIR}/runcoms/zshenv" ] && source ${PREZTO_DIR}/runcoms/'${file} >> /etc/zsh/${file}
        done
      }

      for login in root vagrant; do
        echo "  ... set zsh as login shell for $login"
        chsh -s $(which zsh) $login
      done
    }

    echo "prepare ansible"
    tzz command -v pip || {
      echo "  ... installing python-setuptools python-pip"
      tzz apt-get install -y python-setuptools python-pip
      echo "  ... installing paramiko PyYAML Jinja2 httplib2 six"
      tzz pip install paramiko PyYAML Jinja2 httplib2 six
    }

    gh ansible/ansible

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