# Ceph deply bootstrap

1. **설치환경**
2. **노드정보**

| Node | Hostname | Public-nework | Cluster-network | Descriptions |
| ---- | -------- | ------------- | --------------- | ------------ |
|  | ADMIN | mon100 | 192.168.122.100/24 | 10.10.10.100/24 |
|  | OSD1 | osd101 | 192.168.122.101/24 | 10.10.10.101/24 |
|  | OSD2 | osd102 | 192.168.122.102/24 | 10.10.10.102/24 |
|  | OSD3 | osd103 | 192.168.122.103/24 | 10.10.10.103/24 |
|  | Client | client104 | 192.168.122.104/24 | 10.10.10.104/24 |

3. firewalld & selinux & Create a Ceph User (모든 노드)

```
$ useradd -d /home/cephuser -m cephuser
$ passwd 1234
$ echo "cephuser ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephuser
$ chmod 0440 /etc/sudoers.d/cephuser
$ systemctl stop firewalld
$ systemctl disable firewalld
$ sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

4. Install and Configure NTP (모든 노드)

```
$ yum install -y ntp ntpdate ntp-doc
$ ntpdate 0.kr.pool.ntp.org
$ hwclock --systohc
$ systemctl enable ntpd.service
$ systemctl start ntpd.service
```

5. Configure Hosts File(admin노드만 - mon100)

```
10.10.0.100        mon100
10.10.0.101        osd101
10.10.0.102        osd102
10.10.0.103        osd103
10.10.0.104        client104
```

6. Configure the SSH Server(cephuser에서 진행, admin노드만 - mon100)

```
$ su cephuser
$ ssh-keygen
$ ssh-copy-id cephuser@[mon101, osd101~osd103, client104]
$ vim ~/.ssh/config
$ chmod 644 ~/.ssh/config
```

```
vim ~/.ssh/config

ADMIN노드에서 접속할 모든 노드들을 기록
Host mon1
        Hostname mon1
        User cephuser
 
Host osd1
        Hostname osd1
        User cephuser
 
Host client
        Hostname client
        User cephuser

chmod 644 ~/.ssh/config

마지막으로 모든 노드들에 대해서 '$ssh osd102'를 하면서 확인
```

7. Configure the Ceph OSD Nodes(모든 OSD노드에서 실행, 디스크 설정 필요)

```
$ sudo parted -s [OSD에 연결할 디스크 ex) /dev/sdb] mklabel gpt mkpart primary xfs 0% 100%
$ sudo mkfs.xfs [OSD에 연결할 디스크 ex) /dev/sdb] -f
```

8. Add yum repository
{ceph-release}: 인스톨 하고 싶은 버전, {distro}: centos버전
&#91;https://download\.ceph\.com/&#93;\(세프버전 주소\) 여기에서 ceph버전 확인

```
$sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
$sudo vim /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-{ceph-release}/{distro}/$basearch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-{ceph-release}/{distro}/noarch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=https://download.ceph.com/rpm-{ceph-release}/{distro}/SRPMS
enabled=0
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

# You may need to install python setuptools required by ceph-deploy:
$ sudo yum install python-setuptools
$ sudo yum update && yum install ceph-deploy
```

9. Create New Cluster Config and Install Ceph packages to all node(mgmt노드에서 할 것)

```
$cd ~ && mkdir [cluster-foler-name] && cd [cluster-foler-name]
$cehp-deploy new {initial-monitor-node(s)} ex) cehp-deploy new mon100 --public-network x.x.x.x/24
$ceph-deploy install {ceph-node} [...] ex) ceph-deploy mon100 osd101 osd102 osd103 client104
$ceph-deploy mon create-initial #Deploy the initial monitor(s) and gather the keys:
$ceph-deploy admin {ceph-node(s)} ex) ceph-deploy admin mon100 osd101 osd102 osd103 client104
$sudo chmod +r /etc/ceph/ceph.client.admin.keyring
$ceph-deploy mgr create node1
$ceph-deploy disk list {ceph-nodes}
$ceph-deploy osd create --data {device} {ceph-node}
```

10. Check Ceph health(on monitor node)

```
$ceph -s
```

![Screenshot from 2020-03-09 14-20-37.png](/wikis/2479632164736111388/files/2696327748175243938)

11. Block Device 할당
  OSD POOL 생성
   ```
    $ceph osd pool create testrbdpool 8 8 replicated
    $ceph osd lspools # 생성된 풀 확인
   ```
   RBD(On Client Node)
  ```
  $rbd pool init <pool-name>
  $rbd create foo --size 4096 --image-feature layering [-m {mon-IP}] [-k   /path/to/ceph.client.admin.keyring]
   ex) rbd create foo --size 원하는용량(Mega단위)  --image-feature layering -m 10.10.10.101 -k /etc/ceph/ceph.client.admin.keyring
   $sudo rbd map foo --name client.admin [-m {mon-IP}] [-k /path/to/ceph.client.admin.keyring]
 ```
  Disk format & attach to client
  ```
   $sudo fdisk -l #생성된 디스크 확인
   $sudo mkfs.ext4 -m0 /dev/rbd[number]
   $sudo mkdir /mnt/ceph-block-device
   $sudo mount /dev/rbd[number] /mnt/ceph-block-device 
   $cd /mnt/ceph-block-device
  ```
   /mnt/ceph-block-device 에서 클라이언트는 할당된 용량 만큼 클러스트링 된 디스크를 사용 할 수 있다.

12. Ceph 대쉬보드 설정
   Intall package(on mgr node)
   ```
     $sudo yum install ceph-mgr-dashboard
   ```
   Enable the ceph mgr dashboard
   ```
     $ceph mgr module enable dashboard 
     $ceph mgr module ls 
   ```
   Create self-signed certificate and account
   ```
    $sudo ceph dashboard create-self-signed-cert
    $ceph dashboard ac-user-create <username> <password> administrator
   ```
   Open the dashboard url in any browser
   > https://10.10.10.100(ceph-mgr-ip):8443
   
   대쉬보드 완료 시 아래와 같은 화면을 얻을 수 있음
    ![Screenshot from 2020-03-09 14-25-47.png](/wikis/2479632164736111388/files/2696330378720931660)
    ![Screenshot from 2020-03-09 14-25-58.png](/wikis/2479632164736111388/files/2696330479876142004)


# Reference link

[https://bobocomi.tistory.com/2](https://bobocomi.tistory.com/2)
[https://docs.ceph.com/docs/master/start/quick-ceph-deploy/](https://docs.ceph.com/docs/master/start/quick-ceph-deploy/)
[https://www.howtoforge.com/tutorial/how-to-build-a-ceph-cluster-on-centos-7/](https://www.howtoforge.com/tutorial/how-to-build-a-ceph-cluster-on-centos-7/)
[https://www.kangtaeho.com/39](https://www.kangtaeho.com/39)
[https://docs.ceph.com/docs/mimic/start/quick-rbd/](https://docs.ceph.com/docs/mimic/start/quick-rbd/)
&#91;https://www\.youtube\.com/watch?v=dDA1sBg4H98&#93;\(세프 강의\)