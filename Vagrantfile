# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

# These settings should remain unchanged
VAGRANT_PRIVATE_IP = "192.168.10.10"
VAGRANT_ENV = { "DEBIAN_FRONTEND" => "noninteractive" }

# You may want to change these ones ;)
VAGRANT_TIMEZONE = "Europe/Paris"
LOCAL_FOLDER = "/Users/yann/Projects"

# Inline shell script for the Ubuntu guest provisioning
$provisioning = <<SCRIPT
echo #{VAGRANT_TIMEZONE} > /etc/timezone
dpkg-reconfigure tzdata
apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
cat > /etc/apt/sources.list.d/docker.list <<EOF
deb https://apt.dockerproject.org/repo ubuntu-wily main
EOF
apt-get update -q
apt-get upgrade -qy
apt-get autoremove -qy
apt-get install -qy mitmproxy docker-engine
usermod -aG docker vagrant
SCRIPT

# We need this separate non privileged command to create
# mitmproxy's CA certificate in /home/vagrant/.mitmproxy
$mitmproxy = <<SCRIPT
mitmdump -nc /dev/null
SCRIPT

# Add mitmproxy's CA certificate to the docker
# build context, and build the docker image
$docker = <<SCRIPT
cp /home/vagrant/.mitmproxy/mitmproxy-ca-cert.pem /vagrant/docker
docker build -t docker_base /vagrant/docker
rm -f /vagrant/docker/mitmproxy-ca-cert.pem
SCRIPT

# Add a function to vagrant user's .bashrc. This
# is a convenient way to switch on / switch off
# redirection rules.
$iptables = <<SCRIPT
cat >> /home/vagrant/.bashrc <<EOF

function redirection {

  # Redirection rule template
  cmd='sudo iptables -t nat \\${action} PREROUTING -p tcp --dport \\${dport} -j REDIRECT --to 8080 -w'

  # Test if the redirection rules are already set
  sudo iptables -t nat -S PREROUTING |egrep 'dport (80|443) -j REDIRECT' >/dev/null
  is_active=\\$?

  case \\$1 in
    on)
      if (( is_active != 0 )) ; then
        action="-I"
        dport="80" eval \\$cmd
        dport="443" eval \\$cmd
      fi
      ;;
    off)
      if (( is_active == 0)) ; then
        action="-D"
        dport="80" eval \\$cmd
        dport="443" eval \\$cmd
      fi
      ;;
    status)
      (( is_active == 0)) && echo "Redirection is active." || echo "Redirection is *not* active."
      ;;
    *) echo "Usage: redirection on|off|status"
       ;;
  esac
}
EOF
SCRIPT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # VirtualBox specific config
  config.vm.provider "virtualbox" do |vb|
    vb.cpus = "2"
    vb.memory = "1024"
    vb.name = "streamshot"
  end

  # Set a private network between the box and the host
  config.vm.network "private_network", ip: VAGRANT_PRIVATE_IP

  # Install Ubuntu 15.10 on the new box
  config.vm.box = "ubuntu/wily64"

  # Shared folder
  config.vm.synced_folder LOCAL_FOLDER, "/home/vagrant/projects", create: true

  # Box default provisioning (executed once)
  config.vm.provision "shell", inline: $provisioning, env: VAGRANT_ENV

  # .bashrc (executed once)
  config.vm.provision "shell", inline: $iptables, privileged: false

  # Generate mitmproxy CA certificate (executed once)
  config.vm.provision "shell", inline: $mitmproxy, privileged: false

  # Build the docker image (executed once)
  config.vm.provision "shell", inline: $docker

end
