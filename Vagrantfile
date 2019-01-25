# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'
require 'erb'
require 'tempfile'

BOX           = "opensuse/openSUSE-15.0-x86_64"

GOPATH_FIRST  = ENV["GOPATH"].split(":")[0]
KUBEROOT      = ENV["KUBEROOT"]      || "#{GOPATH_FIRST}/src/github.com/kubernetes/kubernetes"
NUM_NODES     = ENV["NUM_NODES"]     || 2
PACKAGES_REPO = ENV["PACKAGES_REPO"] || "https://download.opensuse.org/tumbleweed/repo/oss/"
PACKAGES      = ENV["PACKAGES"]      || "kubernetes-client kubernetes-kubelet docker"
GUESTS_PREFIX = ENV["GUESTS_PREFIX"] || "kubeadm"

# the local templates for the config file
KUBEADM_INIT_TEMPLATE = ENV["KUBEADM_INIT_TEMPLATE"] || "#{__dir__}/configs/kubeadm.config.erb"
KUBEADM_JOIN_TEMPLATE = ENV["KUBEADM_JOIN_TEMPLATE"] || "#{__dir__}/configs/kubeadm-join.config.erb"

# the mount point for the KUBEROOT in the guests
REMOTE_KUBEROOT    = "/kubernetes"

# default `kubeadm` binary
REMOTE_KUBEADM_EXE = ENV["REMOTE_KUBEADM_EXE"] || "#{REMOTE_KUBEROOT}/_output/local/go/bin/kubeadm"

# path for the remote `kubeadm.config`
REMOTE_KUBEADM_CFG = ENV["REMOTE_KUBEADM_CFG"] || "/etc/kubernetes/kubeadm.config"

# some runtime configuration
CFG_PODS_CIDR      = ENV["CFG_PODS_CIDR"]     || "10.244.0.0/16"
CFG_SERVICES_CIDR  = ENV["CFG_SERVICES_CIDR"] || "10.96.0.0/12"
TOKEN              = ENV["TOKEN"]             || "5ynki1.3erp9i3yo7gqg1nv"

# some kubelet hacks
KUBELET_DROPIN        = "#{__dir__}/configs/kubelet.drop-in.conf"
REMOTE_KUBELET_DROPIN = "/etc/systemd/system/kubelet.service.d/kubelet.drop-in.conf"
KUBELET_CFG           = "#{__dir__}/configs/kubelet-config.yaml"
REMOTE_KUBELET_CFG    = "/var/lib/kubelet/config.yaml"

# we assume we are using a OpenSUSE distro: 
# we use some commands for installing the Guest Additions for VirtualBox
GUEST_ADD_RPM      = 'virtualbox-guest-kmp-default'
GUEST_ADD_CMD      = "zypper removelock #{GUEST_ADD_RPM} ; zypper in -y --force --force-resolution #{GUEST_ADD_RPM}"
GUEST_LOCKS        = "#{GUEST_ADD_RPM} bash-completion"  # some extra locks to remove

KUBEADM_FLAGS = "--ignore-preflight-errors=Swap"

# things that must be setup before trying to start kubeadm
KUBEADM_PRE_SETUP = %{
    rm -f /usr/lib/systemd/system/kubelet.service.d/*.conf
    systemctl daemon-reload

    modprobe br_netfilter

    sysctl -w net.bridge.bridge-nf-call-arptables=0
    sysctl -w net.bridge.bridge-nf-call-ip6tables=0
    sysctl -w net.bridge.bridge-nf-call-iptables=0

    systemctl enable --now docker

    echo 1 > /proc/sys/net/ipv4/ip_forward
    echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
}

# IPs for the VMs network. The seeder will take the .10, and
# then we will assign to nodes .11, .12 and so on
IPS_BASE   = "172.18.7"
IPS_SEEDER = "#{IPS_BASE}.10"

###################################################################################
# support
###################################################################################

# install the Guest Additions as an RPM
class RPMInstaller < VagrantVbguest::Installers::Linux
    def install(opts=nil, &block)
        communicate.sudo(GUEST_ADD_CMD, opts, &block)
    end
end        

# configurre the global plugins
def config_plugins(config)
    if Vagrant.has_plugin?("vagrant-vbguest")
        # install using just an RPM
        config.vbguest.installer = RPMInstaller
    end

    if Vagrant.has_plugin?("vagrant-cachier")
        # http://docs.vagrantup.com/v2/synced-folders/basic_usage.html
        config.cache.scope = :box
        #config.cache.synced_folder_opts = {
        #    type: :nfs,
        #    mount_options: ['rw', 'vers=3', 'tcp', 'nolock']
        #}
    end

    if Vagrant.has_plugin?("vagrant-hostmanager")
        # https://github.com/devopsgroup-io/vagrant-hostmanager
        # NOTE: you should enable passwordless configuration:
        #       https://github.com/devopsgroup-io/vagrant-hostmanager#passwordless-sudo
        config.hostmanager.enabled           = true
        config.hostmanager.manage_host       = true
        config.hostmanager.manage_guest      = true
        config.hostmanager.ignore_private_ip = false
        config.hostmanager.include_offline   = true
    end
end

# prrovision a file, creating the container dir if necessary
def provision_file(vm, src, dst)
    # the copier is not provileged, so we must copy files to a temp file
    # and then move them
    b = File.basename(dst)
    d = File.dirname(dst)
    temp_file = "/tmp/tmp-#{b}"
    vm.provision "file", source: src, destination: temp_file

    code = %{
        echo "Making sure destination dir #{d} exists..."
        mkdir -p #{d}
        echo "Moving #{temp_file} to #{dst}"
        cp -f #{temp_file} #{dst}
    }
    vm.provision "shell", inline: code
end

# configure a VM
def config_vm(guest, name, ip)
    guest.vm.box      = BOX
    guest.vm.hostname = name

    # TODO: enable libvirt boxes: https://github.com/vagrant-libvirt/vagrant-libvirt

    guest.vm.network "private_network", ip: ip
    #guest.vm.network "public_network",
    #    use_dhcp_assigned_default_route: true

    guest.vm.synced_folder KUBEROOT, REMOTE_KUBEROOT

    guest.vm.provider "virtualbox" do |vb|
        vb.memory = 2048
        vb.cpus = 2
        #vb.customize ["modifyvm", :id, "--audio", "none"]
        vb.linked_clone = true if Gem::Version.new(Vagrant::VERSION) >= Gem::Version.new('1.8.0')
    end

    #if Vagrant.has_plugin?("vagrant-hostmanager")
    #    guest.hostmanager.aliases = %w(example-box.localdomain example-box-alias)
    #end
end

# upload a specific kubeadm configuration file
def upload_kubeadm_cfg(vm, filename)
    # return a the result of prcoessing a template
    def template_file(f, b)
        template = Tempfile.new
        template.write ERB.new(File.read(f)).result(b)
        template.rewind
        template
    end

    provision_file(vm, template_file(filename, binding), REMOTE_KUBEADM_CFG)
end

# ensure that some packages are installed, using the given repository
def ensure_packages(vm, repo, packages)
    if ! repo.empty?
        vm.provision "shell", inline: "zypper lr extra || zypper ar --enable --refresh --no-gpgcheck #{repo} extra"
    end

    if ! packages.empty? 
        vm.provision "shell", inline: "for f in #{GUEST_LOCKS} ; do echo Removing lock for $f ; zypper removelock $f ; done"
        vm.provision "shell", inline: "zypper in -y --no-recommends #{packages}"
    end
end

# start the kubeadm process
def start_kubeadm(vm, what, extra)
    provision_file(vm, KUBELET_DROPIN, REMOTE_KUBELET_DROPIN)
    provision_file(vm, KUBELET_CFG,    REMOTE_KUBELET_CFG)
    
    vm.provision "shell", inline: KUBEADM_PRE_SETUP

    kubeadm_reset_cmd = "#{REMOTE_KUBEADM_EXE} reset --force"
    vm.provision "shell", inline: kubeadm_reset_cmd

    kubeadm_cmd = "#{REMOTE_KUBEADM_EXE} #{what} #{KUBEADM_FLAGS} --config #{REMOTE_KUBEADM_CFG} #{extra}"
    vm.provision "shell", inline: kubeadm_cmd
end

###################################################################################
# main
###################################################################################

Vagrant.configure("2") do |config|
    config_plugins(config)

    config.vm.define "seeder", primary: true do |seeder|
        config_vm(seeder, "#{GUESTS_PREFIX}-seeder-0", IPS_SEEDER)
        ensure_packages(seeder.vm, PACKAGES_REPO, PACKAGES)
        upload_kubeadm_cfg(seeder.vm, KUBEADM_INIT_TEMPLATE)
        start_kubeadm(seeder.vm, "init", "")
    end

    (1..NUM_NODES).each do |i|
        config.vm.define "node-#{i}" do |node|
            ip = "#{IPS_BASE}.#{10 + i}"
            config_vm(node, "#{GUESTS_PREFIX}-node-#{i}", ip)
            ensure_packages(node.vm, PACKAGES_REPO, PACKAGES)
            upload_kubeadm_cfg(node.vm, KUBEADM_JOIN_TEMPLATE)
            start_kubeadm(node.vm, "join", "#{IPS_SEEDER}:6443")
        end
    end
end
