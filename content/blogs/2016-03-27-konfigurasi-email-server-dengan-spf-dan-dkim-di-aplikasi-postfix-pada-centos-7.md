---
layout: single
title: Konfigurasi Email Server Dengan SPF Dan DKIM Di Aplikasi Postfix Pada CentOS 7
date: 2016-03-27T12:59:09+07:00
excerpt:
modified:
categories: sysadmin
tags: [sysadmin, email server, postfix, spf, dkim]
image:
  feature:
comments: true
---

Setelah kita melakukan instalasi email server, cek skor email server di website [Mail Tester](https://www.mail-tester.com/). Salah satu contoh hasil pengecekan website tersebut sebagai berikut.

![Mail-Tester](/images/mail-tester.png)

    Content:
    [Text]
    [HTML]
    [Source]

    SpamAssassin:
    [Score]
     The famous spam filter SpamAssassin. A score below -5 is considered spam.

    Authentication:
     We check if the server you are sending from is authenticated
    [SPF]
     Sender Policy Framework (SPF) is an email validation system designed to prevent email spam by detecting email spoofing, a common vulnerability, by verifying sender IP addresses.
    [Sender ID]
     Sender ID is like SPF, but it checks the FROM address, not the bounce address.
    [DKIM]
     DomainKeys Identified Mail (DKIM) is a method for associating a domain name to an email message, thereby allowing a person, role, or organization to claim some responsibility for the message.
    [DMARC record]
     A DMARC policy allows a sender to indicate that their emails are protected by SPF and/or DKIM, and give instruction if neither of those authentication methods passes. Please be sure you have a DKIM and SPF set before using DMARC.
    [PTR record]
     Reverse DNS lookup or reverse DNS resolution (rDNS) is the determination of a domain name that is associated with a given IP address. Some companies such as AOL will reject any message sent from a server without rDNS, so you must ensure that you have one. You cannot associate more than one domain name with a single IP address.
    [A record]
     We check if there is a mail server (A Record) behind your hostname.

    Errors:
     Checks whether your message is well formatted or not.
    [Image alternate tag]
     alt attributes provide a textual alternative to your images. It is a useful fallback for people suffering from sight problems and for cases where your images cannot be displayed.
    [HTML elements]
     Checks whether your message contains dangerous html elements such as javascript, iframes, embed content or applet.
    [URL shortener]
     Checks whether your message uses URL shortener systems.
    [List-Unsubscribe header]
     The List-Unsubscribe header is required if you send mass emails, it enables the user to easily unsubscribe from your mailing list.
    [Blacklist]
     Matches your server IP address against 22 of the most common ipv4 blacklists.

Untuk meningkatkan skor, kita bisa memperbaiki konfigurasi email server dengan SPF dan DKIM

## SPF

Sender Policy Framework (SPF) adalah sebuah sistem validasi email yang didesain untuk mencegah email spam dengan cara mendeteksi email spoofing, sebuah celah umum, dengan melakukan verifikasi alamat IP pengirim. SPF berupa TXT record yang ditambahkan ke domain email server kita.

    # contoh
    example.com IN  TXT "v=spf1 mx a ip4:192.168.1.0/24 include:myexample.com ~all"

Contoh di atas memberikan informasi bahwa email dengan domain example.com valid jika dikirim dari alamat-alamat IP di jaringan 192.168.1.0/24 dan server yang memiliki hostname myexample.com. Email dengan domain example.com yang dikirim dari selain alamat-alamat tersebut akan ditandai tetapi tetap diterima.

## DKIM

DomainKeys Identified Mail (DKIM) adalah metode untuk mengasosiasikan sebuah nama domain ke sebuah pesan email, sehingga mengizinkan seseorang atau sebuah organisasi untuk mengklaim tanggung jawab untuk pesan tersebut. Berikut ini adalah langkah instalasi OpenDKIM.

    sudo yum -y install epel-release
    sudo yum -y install opendkim
    sudo opendkim-default-keygen
    sudo su
    cd /etc/opendkim/keys
    ll

File _default.private_ adalah private key untuk domain kita dan _default.txt_ adalah public key yang akan kita masukkan ke dalam DNS record (TXT).

Ubah file konfigurasi untuk OpenDKIM

    vi /etc/opendkim.conf
    # contoh konfigurasi
    Mode    sv
    Socket  inet:8891@localhost
    Canonicalization    relaxed/simple
    Domain  example.com
    #KeyFile    /etc/opendkim/keys/default.private
    KeyTable    refile:/etc/opendkim/KeyTable
    SigningTable    refile:/etc/opendkim/SigningTable
    ExternalIgnoreList  refile:/etc/opendkim/TrustedHosts
    InternalHosts   refile:/etc/opendkim/TrustedHosts

Ubah file _KeyTable_ yang mendefinisikan alamat private key untuk domain kita

    vi /etc/opendkim/KeyTable
    # contoh konfigurasi
    default._domainkey.example.com example.com:default:/etc/opendkim/keys/default.private

Ubah file _SigningTable_ yang memberikan informasi bagaimana OpenDKIM menggunakan key files.

    vi /etc/opendkim/SigningTable
    # contoh konfigurasi
    *@example.com default._domainkey.example.com

Ubah file _TrustedHosts_ yang mendefinisikan host mana yang berhak menggunakan key files tersebut.

    vi /etc/opendkim/TrustedHosts
    # contoh konfigurasi
    127.0.0.1
    mail.example.com

Ubah file konfigurasi untuk Postfix

    vi /etc/postfix/main.cf
    # contoh konfigurasi
    smtpd_milters = inet:127.0.0.1:8891
    non_smtpd_milters = $smtpd_milters
    milter_default_action = accept

Jalankan ulang OpenDKIM dan Postfix

    sudo hash -r
    sudo systemctl start opendkim ; systemctl enable opendkim ; systemctl restart postfix

Tambahkan key ke DNS record

    default._domainkey IN TXT "v=DKIM1; g=*; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDNHHZ5Wq8bmpNTwzg+5wwwgKiYphgdRnngxS6Wd1lq9kQRh2OxzOH4kW1gsPe8UA396e9zaXjGlSzbIkeHEt86JzuS+fg+utLrVtIH6gLXJgxppBjCMhLy95oBLrG9M3rqGtrzHgVclANnYdfGs3Tg6r+RnS7GHW3YqW+7tr45YQIDAQAB" ; ----- DKIM default for example.com

Berikut ini adalah daftar layanan yang menyediakan informasi alamat-alamat IP yang termasuk dalam daftar hitam karena sering mengirimkan spam.

    ASPEWS
    BACKSCATTERER
    BARRACUDA
    BBFHL1
    BBFHL2
    BLOCKLIST.DE
    BSB
    CASA CBL
    CASA CBLESS
    CASA CBLPLUS
    CASA CDL
    CBL
    CYMRU BOGONS
    DAN TOR
    DAN TOREXIT
    DNS REALTIME BLACKHOLE LIST
    DNS SERVICIOS
    DRONE BL
    DULRU
    EFNET RBL
    EMAILBASURA
    FABELSOURCES
    HIL
    HIL2
    IBM DNS BLACKLIST
    ICMFORBIDDEN
    IMP SPAM
    IMP WORM
    INPS_DE
    INTERSERVER
    IPRANGE RBL PROJECT
    IVMSIP
    IVMSIP24
    JIPPG
    KEMPTBL
    KONSTANT
    LASHBACK
    LNSGBLOCK
    LNSGBULK
    LNSGMULTI
    LNSGOR
    LNSGSRC
    MADAVI
    MAILBLACKLIST
    MAILSPIKE BL
    MAILSPIKE Z
    MEGARBL
    MSRBL PHISHING
    MSRBL SPAM
    NETHERRELAYS
    NETHERUNSURE
    NIXSPAM
    NOSOLICITADO
    ORVEDB
    OSPAM
    PROTECTED SKY
    PSBL
    RATS ALL
    RATS DYNA
    RATS NOPTR
    RATS SPAM
    RBL JP
    REDHAWK
    RSBL
    SCHULTE
    SECTOOR EXITNODES
    SEM BACKSCATTER
    SEM BLACK
    SENDER SCORE REPUTATION NETWORK
    SERVICESNET
    SORBS BLOCK
    SORBS DUHL
    SORBS HTTP
    SORBS MISC
    SORBS NEW
    SORBS SMTP
    SORBS SOCKS
    SORBS SPAM
    SORBS WEB
    SORBS ZOMBIE
    SPAMCANNIBAL
    SPAMCOP
    SPAMHAUS ZEN
    SPEWS1
    SPEWS2
    SWINOG
    TRIUMF
    TRUNCATE
    UCEPROTECTL1
    UCEPROTECTL2
    UCEPROTECTL3
    URIBL MULTI IP
    VIRBL
    WOODYS SMTP BLACKLIST
    WPBL
    ZAPBL
