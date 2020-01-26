---
layout: single
title: Pembuatan File Image CentOS 7 Untuk Instance OpenStack Liberty
modified:
categories: 
excerpt:
tags: []
image:
  feature:
date: 2016-03-30T18:20:17+07:00
comments: true
---

Salah satu fitur OpenStack adalah elastisitasnya yang memungkinkan kita untuk
membuat dan menjalankan _instance_ dengan cepat dan mudah menggunakan _image_
dari sistem operasi yang sudah kita buat sebelumnya. Tutorial kali ini akan
memberikan salah satu cara membuat sendiri _image_ sistem operasi CentOS 7
untuk digunakan di OpenStack Liberty.

## Persiapan Host Environment

Host yang kita gunakan untuk membuat _image_ adalah host dengan sistem operasi
CentOS 7. Pastikan sudah terinstall virtualisasi KVM/__libvirtd__ dan _tools_
yang diperlukan

    sudo yum install qemu-kvm qemu-img virt-manager libvirt libvirt-python libvirt-client virt-install virt-viewer
    sudo yum install libguestfs-tools-c

Konfigurasi _networking_ dari KVM menggunakan profil default dengan DHCP dan
koneksi NAT atau Bridge. Pada pembuatan _image_ CentOS 7 kali ini kita akan
menggunakan instalasi otomatis menggunakan file _kickstart_. Di bawah ini
adalah contoh alamat FTP berisi file _kickstart_ dan file-file instalasi CentOS
7 yang dapat dibuat dengan cara menyalin isi dari file ISO instalasi CentOS 7
ke dalam direktori FTP tersebut

    # direktori FTP berisi file instalasi CentOS 7
    ftp://<your FTP server>/repos/centos/7/
    # file kickstart di di dalam direktori FTP
    ftp://<your FTP server>/repos/kickstarts/centos-7.cfg

Berikut ini adalah contoh isi dari file _kickstart_ yang akan kita gunakan.

	#version=RHEL7
	# System authorization information
	auth --enableshadow --passalgo=sha512

	# Specify installation
	install
	# Run the Setup Agent on first boot
	firstboot --enable
	# Text mode (no graphical mode)
	text
	# Skip graphics
	skipx
	# Agree to EULA
	eula --agreed
	# Logging
	logging --level=info

	# Installation files FTP directory
	url --url="ftp://<your FTP server>/repos/centos/7/"

	# Language support
	lang en_US.UTF-8
	# Keyboard
	keyboard --vckeymap=us --xlayouts='us'

	# Network
	network --device eth0 --bootproto dhcp --noipv6 --hostname <hostname>

	# Timezone
	timezone  Asia/Jakarta --isUtc --ntpservers=<your NTP server>

	# Ignore other disk
	ignoredisk --only-use=vda
	# Zero MBR
	zerombr
	# Bootloader
	bootloader  --append=" crashkernel=auto" --location=mbr --boot-drive=vda
	# Remove all partitions
	clearpart --all --initlabel --drives=vda
	# Create partitions on the system
	part / --asprimary --fstype="ext4" --grow --size=1
	#part swap --recommended

	# No root password
	rootpw --lock
	# User
	user --groups=wheel --name=centos --password=<your password hash> --iscrypted --gecos="Centos Cloud User"

	# Enabled servies
	services --enabled=NetworkManager,sshd,chrony
	# SElinux permissive
	selinux --permissive
	# Firewall
	firewall --service=ssh

	# Reboot after install
	reboot

	# Packages installation
	%packages
	@core
	chrony
	bash-completion
	kexec-tools
	net-tools
	telnet
	wget
	git
	vim
	%end

	%addon com_redhat_kdump --enable --reserve-mb='auto'
	%end

	%post
	%end

## Instalasi Sistem Operasi CentOS 7 ke dalam File Image

Setelah file instalasi dan file _kickstart_ siap, maka kita bisa memulai
pembuatan _image_ sistem operasi CentOS 7. Langkah pertama, kita buat file
sebagai _virtual disk_ untuk _image_ sistem operasi CentOS 7.

    qemu-img create -f qcow2 centos-7.qcow2 2G

Setelah itu jalankan instalasi sistem operasi CentOS 7 ke dalam file _image_
tersebut. Pastikan sudah mengunduh file CentOS-7-x86_64-Minimal-1511.iso dari
situs resmi CentOS.

    sudo virt-install --virt-type kvm --name centos-7 --ram 1024 --disk ./centos-7.qcow2,format=qcow2 --network network=default --graphics none --os-type=linux --os-variant=rhel7 --console pty,target_type=serial --extra-args="console=ttyS0,115200n8 serial ksdevice=eth0 ip=dhcp ks=ftp://<your FTP server>/repos/kickstarts/centos-7.cfg" --location=./CentOS-7-x86_64-Minimal-1511.iso 

Perintah tersebut akan menjalankan sebuah terminal serial yang akan menunjukkan
tahapan instalasi otomatis sistem operasi CentOS 7 ke dalam file _image_ CentOS
7. Pada saat instalasi selesai akan muncul permintaan login dari _virtual
   machine_ yang telah kita instal sistem operasi CentOS 7 tersebut.

## Bug pada Package Cloud-Init dari CentOS 7

Package __cloud-init__ 0.7.5 dari CentOS tidak dapat mengambil metadata dari
OpenStack sehingga pada saat pembuatan instance informasi seperti SSH key tidak
terinstal ke dalam instance. Hal ini menyebabkan kita tidak bisa melakukan
login SSH ke dalam instance CentOS OpenStack. Permasalahan ini disebabkan oleh
[bug](https://bugs.launchpad.net/mos/+bug/1406286) yang ada di dalam package
__cloud-init__ 0.7.5 dari CentOS. Pada tutorial ini terdapat salah satu cara 
untuk menghindari bug tersebut dengan menggunakan package __cloud-init__ 0.7.6 
dari Red-Hat.

    wget http://ftp.redhat.com/pub/redhat/linux/enterprise/7Server/en/RH-COMMON/SRPMS/cloud-init-0.7.6-2.el7.src.rpm

Package yang baru saja kita unduh harus dibangun ulang sehingga dapat diinstal
di sistem operasi CentOS 7. Kita instal _tools_ yang diperlukan untuk melakukan
proses _build_.

    sudo yum install rpm-build redhat-rpm-config gcc make
    mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
    rpmbuild --rebuild cloud-init-0.7.6-2.el7.src.rpm

Setelah proses _build_ selesai maka kita unggah package tersebut ke _virtual
machine_ yang akan kita jadikan file _image_ dan instal package tersebut.

    cd rpmbuild/RPMS/x86_64
    scp cloud-init-0.7.6-2.el7.centos.x86_64.rpm user@centos-image:~/
    ssh user@centos-image
    sudo yum -y install epel-release
    sudo yum -y install cloud-utils PyYAML policycoreutils-python python-jsonpatch python-prettytable
    sudo rpm -ivh cloud-init-0.7.6-2.el7.centos.x86_64.rpm

## Konfigurasi Image CentOS 7

Setelah itu lakukan konfigurasi pada _networking_ dari _image_ CentOS tersebut
sehingga dapat digunakan dengan baik untuk membuat instance.

    sudo su -c 'cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth0
    DEVICE="eth0"
    BOOTPROTO="dhcp"
    ONBOOT="yes"
    TYPE="Ethernet"
    PERSISTENT_DHCLIENT=1
    NM_CONTROLLED="no"
    EOF'

    sudo su -c 'cat << EOF > /etc/sysconfig/network
    NETWORKING=yes
    NOZEROCONF=yes
    EOF'

Non-aktifkan repositori EPEL sehingga kita dapat menggunakannya hanya ketika
diperlukan.

    sudo sed -i 's/enabled=1/enabled=0/' /etc/yum.repos.d/epel.repo

Ubah file konfigurasi __cloud-init__ untuk melakukan penyesuaian agar bisa
digunakan pada OpenStack Liberty.

    sudo vi /etc/cloud/cloud.cfg 
    ...
    # gunakan opsi ini jika ingin mengaktifkan ssh menggunakan user root
    disable_root: 0
    ...
    cloud_init_modules:
    ...
     - resolv-conf
    ...
    system_info:
      default_user:
        name: centos
    ...

Setelah itu matikan _virtual machine_ yang menjalankan _image_ CentOS. Kita
akan kembali ke terminal host.

    sudo poweroff

Reset dan bersihkan _image_ sehingga dapat digunakan kembali tanpa masalah.

    sudo virt-sysprep -d centos-7

Kurangi ukuran file _image_ dengan cara mengubah nilai dari blok yang tidak
terpakai menjadi nol di dalam _virtual disk_.

    sudo virt-sparsify --compress centos-7.qcow2 centos-7-new.qcow2

Unggah file _image_ tersebut ke server OpenStack kita.

    scp centos-7-new.qcow2 user@openstack:~/

Buat _image_ OpenStack baru dengan file _image_ CentOS 7 baru.

    glance image-create --name centos-7-new --disk-format qcow2 --container-format bare --min-ram 512 --min-disk 10 --visibility public --file ./centos-7-new.qcow2 --progress

Buat dan jalankan sebuah instance menggunakan _image_ OpenStack baru.

    nova boot --flavor m1.medium --image centos-7-new --nic net-id=<your openstack network id> --security-group default --key-name <your ssh key> <your instance name>
    nova console-log <your instance name>

Jika menggunakan _network_ privat, konfigurasi _floating ip_ untuk instance
CentOS tersebut.

    nova floating-ip-associate <your instance name> <your floating ip address>

Akses instance CentOS baru kita.

    ssh -o "StrictHostKeyChecking no" <your floating ip address> -lcentos

## Referensi

* [Creating CentOS and Fedora images ready for Openstack](https://www.rdoproject.org/resources/creating-centos-and-fedora-images-ready-for-openstack/)
* [Creating Centos-7 Image for Openstack](http://www.adminz.in/2014/10/creating-centos-7-image-for-openstack.html)
* [CentOS 6 cloud image with cloud-init-0.7.5 can not retrieve OpenStack metadata](https://bugs.launchpad.net/mos/+bug/1406286)
* [Linux : How to install source rpm on RHEL/CentOS](http://www.itechlounge.net/2012/12/linux-how-to-install-source-rpm-on-rhelcentos/)
