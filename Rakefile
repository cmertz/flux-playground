# frozen_string_literal: true

require 'rake/clean'
require 'fileutils'
require 'base64'
require 'yaml'
require 'json'

def ssh_key(filename)
  sh "ssh-keygen -q -N \"\" -t ed25519 -f #{filename}"
end

def ssh_public_key(private_key_filename, filename)
  sh "ssh-keygen -y -f #{private_key_filename} > #{filename}"
end

task :default do
  sh 'rake -T'
end

desc 'Setup a complete system'
task setup: %w[kind:create flux:install kind:load gitserver:install flux:install-crs]

desc 'Reset'
task reset: %w[clean kind:delete setup]

desc 'Generate ssh client key'
file 'id' do
  ssh_key('id')
end
CLEAN << 'id'

desc 'Generate ssh client public key'
file 'id.pub' => 'id' do
  ssh_public_key('id', 'id.pub')
end
CLEAN << 'id.pub'

desc 'Generate ssh host key'
file 'gitserver/ssh_host_key' do
  ssh_key('gitserver/ssh_host_key')
end
CLEAN << 'gitserver/ssh_host_key'

desc 'Generate ssh host public key'
file 'gitserver/ssh_host_key.pub' => 'gitserver/ssh_host_key' do
  ssh_key('gitserver/ssh_host_key', 'gitserver/ssh_host_key.pub')
end
CLEAN << 'gitserver/ssh_host_key.pub'

desc 'Generate authorized_keys file'
file 'gitserver/authorized_keys' => 'id.pub' do
  FileUtils.cp('id.pub', 'gitserver/authorized_keys')
end
CLEAN << 'gitserver/authorized_keys'

desc 'Generate flux credentials secret for git over ssh'
file 'flux/config-ssh-credentials.yaml': %w[id id.pub gitserver/ssh_host_key.pub] do
  def oneline(str)
    str.
      split("\n").
      join('')
  end

  def inline_base64(filename)
    oneline(Base64.encode64(File.read(filename)))
  end

  File.write('flux/config-ssh-credentials.yaml', <<~ENDOFTEMPLATE
    apiVersion: v1
    kind: Secret
    type: Opaque
    metadata:
        name: config-ssh-credentials
        namespace: flux-system
    data:
        identity: #{inline_base64('id')}
        identity.pub: #{inline_base64('id.pub')}
        known_hosts: #{oneline(Base64.encode64("gitserver #{File.read('gitserver/ssh_host_key.pub')}"))}
    ENDOFTEMPLATE
  )
end
CLEAN << 'flux/config-ssh-credentials.yaml'

namespace :kind do
  desc 'Load local docker images to kind nodes'
  task load: %w[gitserver:build] do
    sh 'kind load docker-image gitserver'
  end

  desc 'Create kind cluster'
  task :create do
    sh 'kind create cluster --config=kind-cluster.yaml'
  end

  desc 'Delete kind cluster'
  task :delete do
    sh 'kind delete cluster'
  end

  desc 'Ensure we have a connection via the `kind-kind` context and a usable cluster'
  task :running do
    unless YAML.load(File.read(File.expand_path('~/.kube/config')))['current-context'] == 'kind-kind'
      raise "need `kind-kind` kubectl context"
    end
    sh 'kubectl cluster-info'
  end
end

namespace :gitserver do
  desc 'Build gitserver docker image'
  task build: %w[gitserver/ssh_host_key gitserver/authorized_keys] do
    sh 'docker build gitserver -t gitserver'
  end

  desc 'Install the gitserver manifests on the kind cluster'
  task install: %w[kind:running kind:load] do
    sh 'kubectl apply -f gitserver/kubernetes.yaml'
  end
end

namespace :flux do
  desc 'Install flux via its cli on the kind cluster'
  task install: %w[kind:running flux/config-ssh-credentials.yaml] do
    sh 'flux install'
  end

  desc 'Install flux manifests for the config repo'
  task 'install-crs': %w[kind:running flux:install] do
    Dir.glob('flux/*.yaml').each do |file|
      sh "kubectl apply -f #{file}"
    end
  end
end

def port
  YAML.load(`kubectl get svc gitserver -n flux-system -oyaml`)['spec']['ports'].first['nodePort']
end

def node_ips
  `kubectl get nodes -ojsonpath='{ .items[*].status.addresses }'`.
    split(" ").
    map{|e| JSON.parse(e)}.
    map{|e| e.first}.
    reject{|e| e['type'] == 'Hostname'}.
    map{|e| e['address']}
end

desc 'Setup local config dir'
file 'config' do
  sh <<~ENDOFSCRIPT
    git init --initial-branch=main config
    cd config
    git config user.name root
    git config user.email root@example.com
    git commit -a -m 'initial'
    git remote add origin ssh://git@#{node_ips.first}:#{port}/srv/git/config
    git push origin main
  ENDOFSCRIPT
end
CLEAN << 'config'
