### 노드구성
mgr01
> 기능: ansible, mgr, mon, grafana 의 역할을 함
> public ip: 192.168.120.239, cluster ip: 10.0.0.1

com01
> 기능: osd, mon 의 역할을 함

### 설치과정
1. Ceph-ansible 다운로드(mgr01)
```
$ cd ~
$ git clone https://github.com/ceph/ceph-ansible
$ cd ceph-ansible
$ git checkout stable-4.0
$ pip install -r requirement.txt
pip이 없다고 뜰 시
$ yum install epel-release
$ yum install python-pip
한 후에 진행
```
2. Ansible download(이 부분은 건너뜀)(mgr01)
```
$ wget https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.9.6-1.el7.ans.noarch.rpm
```
3. 초기설정1(모든 노드)
! Ansible에서 배포를 하는 노드에 대해서는 root권한으로 ssh-copy-id가 설정되어 있어야 한다.
```
$ systemctl stop firewalld
$ systemctl disable friewalld
$ sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
# /etc/hosts 파일 수정
$ vi /etc/hosts
'''
10.0.0.1 mgr01
10.0.0.1 com01
'''
```
4. 초기설정2(mgr0)
```
$ root@mgr01 echo "admin ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/admin
$ root@mgr01 chmod 0440 /etc/sudoers.d/admin
$ root@mgr01 ssh-keygen -t rsa
$ root@mgr01 ssh-copy-id root@mgr01
$ root@mgr01 ssh-copy-id node@com01
$ root@mgr01 vi ~/.ssh/config
! 주의 ssh-copy-id와 동일하게 할 것!(이름이면 이름, IP면 IP)
'''
Host mgr01
        Hostname mgr01
        User root
Host com01
        Hostname com01
        User node
'''
$ root@mgr01 chmod 644 ~/.ssh/config
$ root@mgr01 parted -s /dev/sdb mklabel gpt mkpart primary xfs 0% 100%
$ root@mgr01 mkfs.xfs /dev/sdb -f
```
5. 초기설정3(com0)
```
$ root@com01 echo "node ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/node
$ root@com01 chmod 0440 /etc/sudoers.d/node
$ node@com01 ssh-keygen -t rsa
$ node@com01 ssh-copy-id root@mgr01
$ node@com01 ssh-copy-id node@com01
$ node@com01 vi ~/.ssh/config
'''
Host mgr01
        Hostname mgr01
        User root
Host com01
        Hostname com01
        User node
'''
$ node@com01 chmod 644 ~/.ssh/config
$ node@com01 sudo parted -s /dev/sdb mklabel gpt mkpart primary xfs 0% 100%
$ node@com01 sudo mkfs.xfs /dev/sdb -f
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
journal_size: 5120
public_network: 192.168.120.0/24
cluster_network: 10.0.0.0/24
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
 - /dev/sdc
```
7.3 mons.yml 파일
```
미정
```
7.4 inventory 파일
ansible이 어떤 노드에 배포할지를 지정하는 파일(site.yml파일에서 host에 해당)
```
[mons]
mgr01 monitor_interface=enp3s0f1
com01 monitor_interface=enp25s0f3

[osds]
com01

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