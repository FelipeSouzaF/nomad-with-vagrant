# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
    # The most common configuration options are documented and commented below.
    # For a complete reference, please see the online documentation at
    # https://docs.vagrantup.com.

    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://vagrantcloud.com/search.
    config.vm.box = "bento/ubuntu-22.04"

    config.vm.provider :virtualbox do |v|
      v.memory = 4096
      v.cpus = 2
    end

    # Define two VMs with static private IP addresses.
    boxes = [
        { :name => "nomadn1", :ip => "192.168.56.120", :hostname => "nomadn1" },
    ]

    # Provision each of the VMs.
    boxes.each do |opts|
        config.vm.define opts[:name] do |config|
            config.vm.hostname = opts[:hostname]
            config.vm.network :private_network, ip: opts[:ip]

            # config.vm.synced_folder "files/", "/opt/vagrant-files", create: true

            config.vm.provision "shell" do |s|
                # Read host ssh pub key
                ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_ed25519.pub").first.strip

                # Create alias for the ip addresses of the other boxes.
                hosts_file = ""
                boxes.each do |address|
                    # Skip the current box.
                    if address[:name] == opts[:name]
                        next
                    end
                    hosts_file += "#{address[:ip]} #{address[:hostname]}\n"
                end

                # Add the host ssh pub key into authorized_keys of the boxes.
                # Apply the other boxes alias to hosts file.
                s.inline = <<-SHELL
                echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys &&
                echo -e "#{hosts_file}" >> /etc/hosts

                apt update -y

                apt install zip net-tools wget gpg coreutils ca-certificates curl gnupg -y

                # install docker
                install -m 0755 -d /etc/apt/keyrings

                curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

                chmod a+r /etc/apt/keyrings/docker.gpg

                echo \
                    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
                    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
                    tee /etc/apt/sources.list.d/docker.list > /dev/null

                apt update -y

                apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

                # install nomad
                wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
                echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/hashicorp.list

                apt update -y

                apt install nomad -y

                # nomad systemd
                touch /etc/systemd/system/nomad.service

                echo '
[Unit]
Description=Nomad
Documentation=https://www.nomadproject.io/docs/
Wants=network-online.target
After=network-online.target

# When using Nomad with Consul it is not necessary to start Consul first. These
# lines start Consul before Nomad as an optimization to avoid Nomad logging
# that Consul is unavailable at startup.
#Wants=consul.service
#After=consul.service

[Service]

# Nomad server should be run as the nomad user. Nomad clients
# should be run as root
# Tive um problema com o usuário nomad (Constraint "missing drivers" filtered 1 nodes)
User=root
Group=root

ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/bin/nomad agent -config /etc/nomad.d
KillMode=process
KillSignal=SIGINT
LimitNOFILE=65536
LimitNPROC=infinity
Restart=on-failure
RestartSec=2

## Configure unit start rate limiting. Units which are started more than
## *burst* times within an *interval* time span are not permitted to start any
## more. Use `StartLimitIntervalSec` or `StartLimitInterval` (depending on
## systemd version) to configure the checking interval and `StartLimitBurst`
## to configure how many starts per interval are allowed. The values in the
## commented lines are defaults.

# StartLimitBurst = 5

## StartLimitIntervalSec is used for systemd versions >= 230
# StartLimitIntervalSec = 10s

## StartLimitInterval is used for systemd versions < 230
# StartLimitInterval = 10s

TasksMax=infinity
OOMScoreAdjust=-1000

[Install]
WantedBy=multi-user.target
' > /etc/systemd/system/nomad.service

                systemctl enable nomad

                systemctl start nomad
                SHELL
            end
        end
    end
end
