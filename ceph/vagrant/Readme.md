### 개요
Vagrant, Ceph-Ansible를 이용하여 테스트 환경을 구축하고 각 ceph 노드에 일괄 설치한다.

### 노드구성
10.10.1.2 mgr01
> 기능: ansible, mgr, mon, grafana 의 역할을 함

10.10.1.101 com101
> 기능: osd, mon 의 역할을 함

10.10.1.102 com102
> 기능: osd, mon 의 역할을 함

10.10.1.103 com103
> 기능: osd 의 역할을 함, 나중에 OSD추가 테스트에 쓰임

### 설치과정
Virtualbox 다운로드
```
$ cd ~
$ wget https://download.virtualbox.org/virtualbox/5.2.38/VirtualBox-5.2-5.2.38_136252_el7-1.x86_64.rpm
```
1. Vagrant 다운로드(개인 PC, 2.2.7 버전 사용)
```
$ wget wget https://releases.hashicorp.com/vagrant/2.2.7/vagrant_2.2.7_x86_64.rpm
yum local install vagrant_2.2.7_x86_64.rpm

# vagrant 2.2.7에서는 virtualbox 5.2가 호환이 된다.(버전 준수!)
$ wget virtualbox 설치(5.2)

# SSH 통신을 위해서 게스트 에디션이 필요하므로 plugin 설치
$ vagrant plugin install vagrant-vbguest
$ vagrant plugin install vagrant-disksize

$ git clone 
$ cd ceph/vagrant
$ vagrant up mgr01 com101
$ vagrant ssh mgro01 # vagrant mgr01노드 접속
```
3. Ceph-ansible 다운로드(mgr01)
```
$ cd ~
$ git clone https://github.com/ceph/ceph-ansible
$ cd ceph-ansible
$ git checkout stable-4.0
$ pip install -r requirement.txt
```
2. Ansible download(이 부분은 건너뜀)(mgr01)
```
$ wget https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.9.6-1.el7.ans.noarch.rpm
```
3. 초기설정1(모든 노드)
Vagrant에는 Vagrantfile에 /etc/hosts, firewalld등 전반 적인 초기 설정이 자동으로 되어 있으므로 물리노드 처럼 구성 할 필요는 없고 다음 단계부터 진행하면 된다.

4. 초기설정2(mgr01)
```
$ ssh-keygen -t rsa
$ ssh-copy-id root@mgr01
$ ssh-copy-id node@com101
$ vi ~/.ssh/config
! 주의 ssh-copy-id와 동일하게 할 것!(이름이면 이름, IP면 IP)
'''
Host mgr01
        Hostname mgr01
        User vagrant
Host com101
        Hostname com101
        User vagrant
Host com101
        Hostname com102
        User vagrant
'''
$ root@mgr01 chmod 644 ~/.ssh/config
```
5. 초기설정3(모든 com노드)
```
$ sudo parted -s /dev/sdb mklabel gpt mkpart primary xfs 0% 100%
$ sudo mkfs.xfs /dev/sdb -f

$ ssh-keygen -t rsa
$ ssh-copy-id root@mgr01
$ ssh-copy-id node@com101
$ ssh-copy-id node@com102
$ vi ~/.ssh/config
'''
Host mgr01
        Hostname mgr01
        User root
Host com01
        Hostname com101
        User vagrant
Host com01
        Hostname com102
        User vagrant
'''
$ vagrant@com01 chmod 644 ~/.ssh/config
```

6. 필요한 파일 복사(mgr01)
```
$ root@mgr01 cd ~/ceph-ansible
$ root@mgr01 cp site.yml.sample site.yml
$ root@mgr01 cp group_vars/all.yml.sample group_vars/all.yml
$ root@mgr01 cp group_vars/mons.yml.sample group_vars/mons.yml
$ root@mgr01 cp group_vars/osds.yml.sample group_vars/osds.yml
```
yml 파일에 대한 간략적인 설명
site.yml : ansible이 배포시 실행 할 명령어의 모임
all.yml : 모든 노드에게 공통적으로 적용되는 설정
osds.yml : osds노드들에 대한 설정
mons.yml : mons노드들에 대한 설정

7. ceph-ansible 옵션 설정

7.1 all.yml 파일(아래의 해당하는 내용만 변경해준다)
monitor_interface: 에서는 mon노드로 사용할 노드의 인터페이스를 기준으로 설정한다.
```
centos_package_dependencies:
 - epel-release
 - libselinux-python

ceph_origin: repository
ceph_repository: community
ceph_stable_release: nautilus
monitor_interface: eth2
journal_size: 1024
public_network: 10.10.1.0/24
cluster_network: "{{ public_network }}"
dashboard_admin_user: admin
dashboard_admin_password: qwe1212!Q
grafana_admin_user: admin
grafana_admin_password: qwe1212!Q

```
7.2 osds.yml 파일
osds노드에서 추가해줄 디스크만 입력한다.
```
devices:
 - /dev/sdb
```
7.3 mons.yml 파일
```
미정
```
7.4 inventory 파일
ansible이 어떤 노드에 배포할지를 지정하는 파일(site.yml파일에서 host에 해당)
```
[mons]
mgr01
com101

[osds]
com101
com102

[mgrs]
mgr01
 
[grafana-server]
mgr01
```

8. ansible 배포
```
$ root@mgr01 cd ~/ceph-ansible
$ root@mgr01 ansible-playbook site.yml -i inventory -vvvv

```