---
layout: single
title: Instalasi Virtualenv, Nodeenv Dan RVM Sebagai Development Environment
modified:
categories: sysadmin
excerpt:
tags: [sysadmin, virtualenv, nodeenv, rvm]
image:
  feature:
date: 2016-03-27T13:02:45+07:00
comments: true
---

## Virtualenv

[Virtualenv](https://virtualenv.readthedocs.org/en/latest/) adalah aplikasi untuk membuat membuat environment virtual Python terisolasi. Aplikasi ini membuat sebuah environment virtual yang mempunyai direktori instalasi sendiri, yang tidak berbagi library dengan environment Virtualenv lainnya dan dapat dikonfigurasi untuk tidak mengakses library yang diinstal secara global.

Instal pip di sistem operasi masing-masing, contohnya CentOS

    sudo yum update -y
    sudo yum install -y python-pip

Instal **virtualenv** menggunakan **pip**

    sudo pip install virtualenv

Buat environment virtual Python menggunakan **virtualenv**

    virtualenv -p PYTHON_VER ENV
    # Python 2.x
    virtualenv -p python2 venv
    # Python 3.x
    virtualenv -p python3 venv

Jalankan environment virtual Python

    source ./ENV/bin/activate
    # contoh
    source ./venv/bin/activate

Cek versi Python yang terinstal

    python -V

Keluar dari environment virtual Python

    deactivate

## Nodeenv

[Nodeenv](http://ekalinin.github.io/nodeenv/) (environment virtual Node.js) adalah sebuat aplikasi untuk membuat environment virtual Node.js terisolasi. Aplikasi ini membuat sebuah environment virtual yang mempunyai direktori instalasi sendiri, yang tidak berbagi library dengan environment Nodeenv lain. Environment virtual baru juga dapat diintegrasikan dengan environment virtual yang dibuat menggunakan Virtualenv (Python).

Instal **nodeenv** menggunakan **pip**

    sudo pip install nodeenv

Tampilkan semua versi Node.js yang tersedia

    nodeenv -l

Instal environment virtual Node.js

    nodeenv -n NODE_VER -v NENV

Instal environment virtual Node.js dengan paket yang sudah terkompilasi

    nodeenv -n NODE_VER --prebuilt -v NENV

Jalankan environment virtual Node.js

    source ./NENV/bin/activate

Integrasi environment node.js dengan environment Virtualenv yang sudah ada, ketika menjalankan environment Virtualenv maka akan secara otomatis menjalankan environment Nodeenv

    nodeenv -n NODE_VER --prebuilt -v -p

Keluar dari environment virtual Node.js

    deactivate_node

## RVM (Ruby Version Manager)

[RVM](https://rvm.io/) adalah aplikasi command-line yang membuat kita dapat dengan mudah melakukan instalasi, manajemen, dan bekerja dengan banyak environment Ruby mulai dari interpreter sampai set dari gem. Selain mempermudah instalasi banyak Ruby interpreter/runtime secara konsisten, aplikasi ini juga memberikan fitur seperti gemset yang tidak biasanya langsung didukung oleh banyak instalasi Ruby. RVM juga mengizinkan kita menggunakan berbagai macam Ruby tanpa merusak instalasi Ruby yang sudah ada.

Impor GPG key

    curl -sSL https://rvm.io/mpapis.asc | gpg2 --import -

Instlasi RVM di environment kita, mode instalasi yang dianjurkan adalah Single-User

    \curl -sSL https://get.rvm.io | bash -s stable
    # instalasi tanpa integrasi dengan file environment (.bash_profile)
    \curl -sSL https://get.rvm.io | bash -s stable --ignore-dotfiles
    echo "source $HOME/.rvm/scripts/rvm" >> ~/.bashrc

Jalankan ulang shell atau ekspor profile RVM

    source ~/.rvm/scripts/rvm

Tes instalasi RVM

    type rvm | head -n 1
    # output
    rvm is a function

Tampilkan versi Ruby yang tersedia

    rvm list known

Instal sebuah versi Ruby

    rvm install 2.3

Jalankan versi Ruby yang telah diinstal

    rvm use 2.3

Cek versi Ruby

    ruby -v
