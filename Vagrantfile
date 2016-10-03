# -*- mode: ruby -*-
# vi: set ft=ruby :

################
# FILE INCLUDE #
################
require 'fileutils'
require 'net/http'
require 'open-uri'
require 'json'
require 'date'

class Module
  def redefine_const(name, value)
    __send__(:remove_const, name) if const_defined?(name)
    const_set(name, value)
  end
end

#######################################
# CONSTANTS AND ENVIRONMENT VARIABLES #
#######################################
VAGRANT_API_VERSION = "2"
REMOVE_VAGRANTFILE_USER_DATA_BEFORE_HALT = (ENV['REMOVE_VAGRANTFILE_USER_DATA_BEFORE_HALT'].to_s.downcase == 'true')
# if this is set true, remember to use --provision when executing vagrant up / reload


COREOS_CHANNEL = ENV["COREOS_CHANNEL"] || "stable"
COREOS_VERSION = ENV['COREOS_VERSION'] || 'latest'
upstream = "http://#{COREOS_CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_VERSION}"
if COREOS_VERSION == "latest"
  upstream = "http://#{COREOS_CHANNEL}.release.core-os.net/amd64-usr/current"
  url = "#{upstream}/version.txt"
  Object.redefine_const(:COREOS_VERSION,
    open(url).read().scan(/COREOS_VERSION=.*/)[0].gsub('COREOS_VERSION=', ''))
end

KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || '1.3.7'
REQUIRED_BINARIES_FOR_MASTER = ['kube-apiserver', 'kube-controller-manager', 'kube-scheduler']
REQUIRED_BINARIES_FOR_NODES  = ['kube-proxy']
REQUIRED_BINARIES = REQUIRED_BINARIES_FOR_MASTER + REQUIRED_BINARIES_FOR_NODES


NODES = ENV["NODES"] || 3
MASTER_MEM = ENV["MASTER_MEM"] || 1024
MASTER_CPUS = ENV['MASTER_CPUS'] || 1

NODE_MEM= ENV['NODE_MEM'] || 2048
NODE_CPUS = ENV['NODE_CPUS'] || 1

BASE_IP_ADDR = ENV['BASE_IP_ADDR'] || "172.17.8"

DNS_DOMAIN = ENV['DNS_DOMAIN'] || "cluster.local"
DNS_UPSTREAM_SERVERS = ENV['DNS_UPSTREAM_SERVERS'] || "8.8.8.8:53,8.8.4.4:53"

SERIAL_LOGGING = (ENV['SERIAL_LOGGING'].to_s.downcase == 'true')

GUI = (ENV['GUI'].to_s.downcase == 'true')
USE_KUBE_UI = ENV['USE_KUBE_UI'] || true


BOX_TIMEOUT_COUNT = ENV['BOX_TIMEOUT_COUNT'] || 50

MASTER_YAML  = File.join(File.dirname(__FILE__), "master.yaml")
NODE_YAML    = File.join(File.dirname(__FILE__), "node.yaml")
CERTS_SCRIPT = File.join(File.dirname(__FILE__), "make-certs.sh")

USE_DOCKERCFG = ENV['USE_DOCKERCFG'] || false
DOCKERCFG = File.expand_path(ENV['DOCKERCFG'] || "~/.dockercfg")

DOCKER_OPTIONS = ENV['DOCKER_OPTIONS'] || ''

CLOUD_PROVIDER = ENV['CLOUD_PROVIDER'].to_s.downcase
validCloudProviders = [ 'gce', 'gke', 'aws', 'azure', 'vagrant', 'vsphere',
  'libvirt-coreos', 'juju' ]
Object.redefine_const(:CLOUD_PROVIDER,
  '') unless validCloudProviders.include?(CLOUD_PROVIDER)


###########
# PLUGINS #
###########
required_plugins = %w(vagrant-triggers)

required_plugins.push('vagrant-timezone')

required_plugins.each do |plugin|
  need_restart = false
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
    need_restart = true
  end
  exec "vagrant #{ARGV.join(' ')}" if need_restart
end

Vagrant.configure(VAGRANT_API_VERSION) do |config|
  # Use host timezone
  config.timezone.value = :host

  # Use vagrant's insecure key for ssh
  config.ssh.insert_key = false
  config.ssh.forward_agent = true

  config.vm.box = "coreos-#{COREOS_CHANNEL}"
  config.vm.box_version = ">= #{COREOS_VERSION}"
  config.vm.box_url = "#{upstream}/coreos_production_vagrant.json"

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
 if Vagrant.has_plugin?("vagrant-vbguest") then
   config.vbguest.auto_update = false
 end

 (1..(NODES.to_i + 1)).each do |i|
   if i == 1
     hostname = "master"
     ETCD_SEED_CLUSTER = "#{hostname}=http://#{BASE_IP_ADDR}.#{i+100}:2380"
     cfg = MASTER_YAML
     memory = MASTER_MEM
     cpus = MASTER_CPUS
     MASTER_IP = "#{BASE_IP_ADDR}.#{i+100}"
  else
    hostname = "minion-%02d" % (i - 1)
    cfg = NODE_YAML
    memory = NODE_MEM
    cpus = NODE_CPUS
  end

  config.vm.define vmName = hostname do |kHost|
    kHost.vm.hostname = vmName
    # suspend / resume is hard to be properly supported because we have no way
    # to assure the fully deterministic behavior of whatever is inside the VMs
    # when faced with XXL clock gaps... so we just disable this functionality.
    kHost.trigger.reject [:suspend, :resume] do
      info "'vagrant suspend' and 'vagrant resume' are disabled."
      info "- please do use 'vagrant halt' and 'vagrant up' instead."
    end

    config.trigger.instead_of :reload do
      exec "vagrant halt && vagrant up"
      exit
    end

    binaries_host_dir="./binaries"
    version_file="#{binaries_host_dir}/version"

    # vagrant-triggers has no concept of global
    # triggers so to avoid them to run as many
    # times as the total number of VMs we only
    # call them in the master
    if vmName == "master"
      kHost.trigger.before [:up, :provision] do
        info "#{Time.now}: setting up Kubernetes master..."
        info "Setting Kubernetes version #{KUBERNETES_VERSION}"

        # create setup file
        setupFile = "#{__dir__}/temp/setup"
        # find and replace kubernetes version and master IP in setup file
        setupFileData = File.read("setup.tmpl")
        setupFileData = setupFileData.gsub("__KUBERNETES_VERSION__", KUBERNETES_VERSION);
        setupFileData = setupFileData.gsub("__MASTER_IP__", MASTER_IP);
        setupFileData = setupFileData.gsub(/^\s*__PROXY_LINE__.*$\n/, "");

        # write new setup data to setup file
        File.open(setupFile, "wb") do |f|
          f.write(setupFileData)
        end

        # give setup file executable permissions
        system "chmod +x temp/setup"

        info "Downloading Kubernetes binaries if version mismatch or binaries not present.."
        REQUIRED_BINARIES.each do |filename|
          toDownload = 0
          file = "#{binaries_host_dir}/#{filename}"

          info "Checking for #{file} with version v#{KUBERNETES_VERSION}.."
          if File.exist?("#{version_file}")
            lastVersion = File.read("#{version_file}").chomp
          end

          if lastVersion != KUBERNETES_VERSION
            info "Versions mismatch [current: #{lastVersion}, desired: #{KUBERNETES_VERSION}]"
            toDownload = 1
          else
            info "Versions match, checking if binary exists..."
            if !File.exist?(file)
              toDownload = 1
            end
          end
          if toDownload == 1
            urlDomain = "storage.googleapis.com"
              urlResource = "/kubernetes-release/release/v#{KUBERNETES_VERSION}/bin/linux/amd64/#{filename}"
              info "Trying to download #{urlDomain}#{urlResource}..."
              Net::HTTP.new(urlDomain).start do |http|
                resp = http.get(urlResource)
                open(file, "wb") do |f|
                  f.write(resp.body)
                end
              end
              info "Download complete."
          end
        end

        # only write current version after all files have
        # been downloaded
        open("#{version_file}", "wb") do |f|
          f.write("#{KUBERNETES_VERSION}")
        end

        # create dns-controller.yaml file
        dnsControllerFile = "#{__dir__}/temp/dns-controller.yaml"
        dnsControllerData = File.read("#{__dir__}/plugins/dns/dns-controller.yaml.tmpl")

        dnsControllerData = dnsControllerData.gsub("__MASTER_IP__", MASTER_IP);
        dnsControllerData = dnsControllerData.gsub("__DNS_DOMAIN__", DNS_DOMAIN);
        dnsControllerData = dnsControllerData.gsub("__DNS_UPSTREAM_SERVERS__", DNS_UPSTREAM_SERVERS);

        # write new setup data to setup file
        File.open(dnsControllerFile, "wb") do |f|
          f.write(dnsControllerData)
        end
      end

      kHost.trigger.after [:up, :resume] do
        info "Sanitizing stuff..."
        system "ssh-add ~/.vagrant.d/insecure_private_key"
        system "rm -rf ~/.fleetctl/known_hosts"
      end

      kHost.trigger.after [:up] do
        info "Waiting for kubernetes master to become ready..."
        j, uri, res = 0, URI("http://#{MASTER_IP}:8080"), nil
        loop do
          j += 1
          begin
            res = Net::HTTP.get_response(uri)
          rescue
            sleep 10
          end
          break if res.is_a? Net::HTTPSuccess or j >= BOX_TIMEOUT_COUNT
        end
        if res.is_a? Net::HTTPSuccess
          info "#{Time.now}: successfully deployed #{vmName}"
        else
          info "#{Time.now}: failed to deploy #{vmName} within timeout count of #{BOX_TIMEOUT_COUNT}"
        end

        info "Installing kubectl for the Kubernetes version we just bootstrapped..."
        system "./temp/setup install"
        system "kubectl config set-cluster local --server=http://#{MASTER_IP}:8080 --insecure-skip-tls-verify=true"
        system "kubectl config set-context local --cluster=local --namespace=default"
        system "kubectl config use-context local"

        info "Configuring Kubernetes DNS..."
        res, uri.path = nil, '/api/v1/namespace/kube-system/replicationcontrollers/kube-dns'
        begin
          res = Net::HTTP.get_response(uri)
        rescue
        end

        if not res.is_a? Net::HTTPSuccess
          system "kubectl create -f temp/dns-controller.yaml"
        end

        res, uri.path = nil, '/api/v1/namespaces/kube-system/services/kube-dns'
        begin
          res = Net::HTTP.get_response(uri)
        rescue
        end

        if not res.is_a? Net::HTTPSuccess
          system "kubectl create -f plugins/dns/dns-service.yaml"
        end

        if USE_KUBE_UI
          info "Configuring Kubernetes dashboard..."

          res, uri.path = nil, '/api/v1/namespaces/kube-system/replicationcontrollers/kubernetes-dashboard'
          begin
            res = Net::HTTP.get_response(uri)
          rescue
          end

          if not res.is_a? Net::HTTPSuccess
            system "kubectl create -f plugins/dashboard/dashboard-controller.yaml"
          end

          res, uri.path = nil, '/api/v1/namespaces/kube-system/services/kubernetes-dashboard'
          begin
            res = Net::HTTP.get_response(uri)
          rescue
          end
          if not res.is_a? Net::HTTPSuccess
            system "kubectl create -f plugins/dashboard/dashboard-service.yaml"
          end

          info "Kubernetes dashboard will be available at http://#{MASTER_IP}:8080/ui"
        end

        # clean temp directory after master is destroyed
        kHost.trigger.after [:destroy] do
          FileUtils.rm_rf(Dir.glob("#{__dir__}/temp/*"))
        end
      end
    end
  end
 end
end
