# frozen_string_literal: true

require 'rake/clean'
require 'fileutils'
require 'base64'
require 'yaml'
require 'json'

task :default do
  sh 'rake -T'
end

desc 'Setup a complete system'
task setup: %w[kind:create registry:start flux:install kind:load gitserver:install flux:install-crs]

desc 'Reset'
task reset: %w[clean kind:delete setup]

# Generate ssh client key
file 'id' do
  ssh_key('id')
end
CLEAN << 'id'

# Generate ssh client public key
file 'id.pub' => 'id' do
  ssh_public_key('id', 'id.pub')
end
CLEAN << 'id.pub'

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
  # Generate ssh host key
  file 'gitserver/ssh_host_key' do
    ssh_key('gitserver/ssh_host_key')
  end
  CLEAN << 'gitserver/ssh_host_key'

  # Generate ssh host public key
  file 'gitserver/ssh_host_key.pub' => 'gitserver/ssh_host_key' do
    ssh_key('gitserver/ssh_host_key', 'gitserver/ssh_host_key.pub')
  end
  CLEAN << 'gitserver/ssh_host_key.pub'

  # Generate authorized_keys file
  file 'gitserver/authorized_keys' => 'id.pub' do
    FileUtils.cp('id.pub', 'gitserver/authorized_keys')
  end
  CLEAN << 'gitserver/authorized_keys'

  desc 'Build gitserver docker image'
  task build: %w[gitserver/ssh_host_key gitserver/authorized_keys] do
    sh 'docker build gitserver -t gitserver'
  end

  desc 'Install the gitserver manifests on the kind cluster'
  task install: %w[kind:running kind:load] do
    sh 'kubectl apply -f gitserver/kubernetes.yaml'
  end
end

namespace :registry do
  desc 'Start a local docker registry for use in a kind cluster'
  task :start do
    sh <<~ENDOFSCRIPT
      if [ "$(docker inspect -f '{{.State.Running}}' kind-registry 2>/dev/null || true)" != 'true' ]; then
        docker run \
          -d --restart=always -p "127.0.0.1:5001:5000" --name kind-registry \
          registry:2
      fi

      if [ "$(docker inspect -f='{{json .NetworkSettings.Networks.kind}}' kind-registry)" = 'null' ]; then
        docker network connect kind kind-registry
      fi

      cat <<EOF | kubectl apply -f -
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: local-registry-hosting
        namespace: kube-public
      data:
        localRegistryHosting.v1: |
          host: "localhost:5001"
          help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
      EOF
    ENDOFSCRIPT
  end
end

namespace :flux do
  # Generate flux credentials secret for git over ssh
  file 'flux/config-ssh-credentials.yaml': %w[id id.pub gitserver/ssh_host_key.pub] do
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

namespace :config do
  # Setup git repo in local config dir
  file 'config/.git' do
    Dir.chdir('config') do
      sh <<~ENDOFSCRIPT
        git init --initial-branch=main .
        git config user.name root
        git config user.email root@example.com
        git remote add origin ssh://git@#{node_ips.first}:#{port}/srv/git/config
      ENDOFSCRIPT
    end
  end
  CLEAN << 'config/.git'

  desc 'Push config to remote config git repository'
  task push: 'config/.git' do
    Dir.chdir('config') do
      sh <<~ENDOFSCRIPT
        git add .
        git commit -m '...'
        GIT_SSH_COMMAND="ssh -i ../id -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null" git push origin main
      ENDOFSCRIPT
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

def ssh_key(filename)
  sh "ssh-keygen -q -N \"\" -t ed25519 -f #{filename}"
end

def ssh_public_key(private_key_filename, filename)
  sh "ssh-keygen -y -f #{private_key_filename} > #{filename}"
end

def oneline(str)
  str.
    split("\n").
    join('')
end

def inline_base64(filename)
  oneline(Base64.encode64(File.read(filename)))
end
