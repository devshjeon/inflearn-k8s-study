# Summary

VM 관리 툴인 Vagrant를 이용해서 로컬환경에서 ***k8s 클러스터***에 대한 setup을 진행했다. 클러스터는 default로 1개의 master와 1개의 worker로 구성되고 worker 노드 수는 Vagrantfile에서 변경할 수 있다. 여기서는 Github에서 [k8s-vagrant-centos](https://github.com/s-u-b-h-a-k-a-r/k8s-vagrant-centos) repo를 clone해서 k8s 클러스터 환경을 구성할 것이다.
setup 구성은 다음과 같다.
### k8s 구성
* Virtualbox
* Vagrant
* Centos 7
### k8s addons
* [helm](https://helm.sh/docs/install/) 
* [kubernetes-dashboard](https://github.com/helm/charts/tree/master/stable/kubernetes-dashboard)
* [nfs-volume-provisioner](https://github.com/helm/charts/tree/master/stable/nfs-client-provisioner)
### vagrant plugins
* [vagrant-reload](https://github.com/aidanns/vagrant-reload) 
* [vagrant-vbguest](https://github.com/dotless-de/vagrant-vbguest)
* [vagrant-timezone](https://github.com/tmatilai/vagrant-timezone)
* [vagrant-disksize](https://github.com/sprotheroe/vagrant-disksize)

<br/>

# Table of Contents
* [Virtualbox Setup](#setup-virtualbox)
* [Vagrant Setup](#setup-vagrant)
* [Repository Clone](#clone-repo)
* [Vagrantfile](#vagrantfile)
* [Vagrant up](#vagrant-up)
* [Vagrant halt](#vagrant-halt)
* [Vagrant destroy](#vagrant-destroy)

<br/>

<a id="setup-virtualbox"></a>
# Virtualbox Setup
Homebrew를 이용해서 virtualbox를 설치한다.
```bash
$ brew install virtualbox
```
> macOS의 경우 시스템 환경설정 -> 보안 및 개인 정보 보호 -> 일반에 들어가서 Oracle 앱에 대해 허용을 해주어야 정상적으로 설치된다.

<br/>

<a id="setup-vagrant"></a>
# Vagrant Setup
Homebrew를 이용해서 vagrant를 설치한다.
```bash
$ brew install vagrant
```

<br/>

<a id="clone-repo"></a>
# Repository Clone
clone할 폴더로 이동해서 repo를 clone한다.
```bash
$ git clone https://github.com/s-u-b-h-a-k-a-r/k8s-vagrant-centos.git

$ cd k8s-vagrant-centos
```
<br/>

<a id="vagrantfile"></a>
# Vagrantfile
[Vagrantfile](https://www.vagrantup.com/docs/vagrantfile)은 VM 설정에 대한 정보를 갖고 있고 ```vagrant up``` 명령어를 사용해서 Vagrantfile의 정보를 읽어 VM을 가동한다.
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :
# vagrant 버전이 2.2.0 이상만 적용되도록 설정
Vagrant.require_version ">= 2.2.0"
# VM 구동을 병렬이 아닌 순차적으로 실행되도록 설정 master-worker 순으로 VM이 생성되야 하므로 해당 옵션을 사용
ENV['VAGRANT_NO_PARALLEL'] = 'yes'

# vagrant plugins 설치 로직을 로드한다.
require File.dirname(__FILE__)+"/dependency_manager"
# 해당 plugin을 설치한다.
check_plugins ["vagrant-reload", "vagrant-vbguest", "vagrant-timezone", "vagrant-disksize"]

# Vagrant.configure(2): configure 2 버전 
Vagrant.configure(2) do |config|
  # plugin에서 설치한 vagrant-timezone 설정 
  if Vagrant.has_plugin?("vagrant-timezone")
    config.timezone.value = "Asia/Seoul"
  end
  # VM 가동 후 shell 파일 실행
  config.vm.provision "shell", path: "vm-prerequisites/install.sh",:args => ["kubeadmin"]
  # Kubernetes Master Node
  config.vm.define "kmaster" do |kmaster|
    kmaster.vm.box = "centos/7"
    kmaster.disksize.size = '20GB'
    kmaster.vm.hostname = "kmaster.example.com"
    kmaster.vm.network "private_network", ip: "100.10.10.100"
    kmaster.vm.provider "virtualbox" do |v|
      v.name = "kmaster"
      v.memory = 2048
      v.cpus = 2
    end
    kmaster.vm.provision "shell", path: "vm-master/install.sh",:args => ["100.10.10.100"]
    kmaster.vm.provision "shell", path: "helm/install.sh"
    kmaster.vm.provision "shell", path: "helm/install_tiller.sh"
  end

  NodeCount = 1
  # Kubernetes Worker Nodes
  (1..NodeCount).each do |i|
    config.vm.define "kworker#{i}" do |workernode|
      workernode.vm.box = "centos/7"
      workernode.vm.hostname = "kworker#{i}.example.com"
      workernode.vm.network "private_network", ip: "100.10.10.10#{i}"
      workernode.vm.provider "virtualbox" do |v|
        v.name = "kworker#{i}"
        v.memory = 4096
        v.cpus = 3
      end
      workernode.vm.provision "shell", path: "vm-worker/install.sh",:args => ["kubeadmin","kmaster.example.com"]
      workernode.vm.provision "shell", path: "helm/install.sh"
      workernode.vm.provision "shell", path: "addons/install.sh",:args => ["100.10.10.100"]
    end
  end
end
```

<br/>

<a id="vagrant-up"></a>
# Vagrant up

<a id="vagrant-halt"></a>
# Vagrant halt

<a id="vagrant-destroy"></a>
# Vagrant destroy