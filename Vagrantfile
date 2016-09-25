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
COREOS_CHANNEL = ENV["COREOS_CHANNEL"] || "alpha"
COREOS_VERSION = ENV['COREOS_VERSION'] || 'latest'
upstream = "http://#{COREOS_CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_VERSION}"
if COREOS_VERSION == "latest"
  upstream = "http://#{COREOS_CHANNEL}.release.core-os.net/amd64-usr/current"
  url = "#{upstream}/version.txt"
  Object.redefine_const(:COREOS_VERSION,
    open(url).read().scan(/COREOS_VERSION=.*/)[0].gsub('COREOS_VERSION=', ''))
end


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
end
