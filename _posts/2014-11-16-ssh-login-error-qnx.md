---
layout: article_page
title: "Solving sshd \"login: No such file\" error on QNX"
date: 2014-11-16 19:33:46
description: "Fix OpenSSH client login error when connecting to server on QNX"
tags: qnx ssh
---

Recently at work, someone said "Ed, can you add a password for ssh access to this embedded board which is running QNX?". Shouldn't take long I thought, being basically just an admin task. It then did take quite a chunk of the day, with my ssh client in Ubuntu stubbornly refusing to allow me to log in to the board. Here is one reason why it doesn't work.

<!--more-->

Having added the password to /etc/passwd on the QNX target, and then altering /etc/ssh/sshd_config to allow only passworded logins, the client on my Ubuntu desktop would give me:

<div class="preformatted_console"><pre>
$ ssh -o PubkeyAuthentication=no root@192.168.1.2
root@192.168.1.2's password: 
Environment:
  SSH_CLIENT=192.168.1.71 45739 22
  SSH_CONNECTION=192.168.1.71 45739 192.168.1.2 22
  SSH_TTY=/dev/ttyp0
  TERM=xterm
debug3: channel 0: close_fds r -1 w -1 e -1 c -1
login: No such file or directory
Connection to 192.168.1.2 closed.
</pre></div>

Hmm, that complaint about "login: No such file or directory" is a bit suspicious.

My main bug bear with OpenSSH is that the server never really gives any meaningful clues about issues encountered, even with its most verbose debug output enabled. Case in point:

<div class="preformatted_console"><pre>
# /usr/sbin/sshd -ddd
debug2: load_server_config: filename /etc/ssh/sshd_config
debug2: load_server_config: done config len = 317
debug2: parse_server_config: config /etc/ssh/sshd_config len 317
debug1: Config token is protocol
debug3: /etc/ssh/sshd_config:20 setting Protocol 2
debug1: Config token is hostkey
debug3: /etc/ssh/sshd_config:26 setting HostKey /var/chroot/sshd/ssh_host_dsa_key
debug1: Config token is logingracetime
debug3: /etc/ssh/sshd_config:40 setting LoginGraceTime 600
debug1: Config token is permitrootlogin
debug3: /etc/ssh/sshd_config:41 setting PermitRootLogin yes
debug1: Config token is passwordauthentication
debug3: /etc/ssh/sshd_config:60 setting PasswordAuthentication yes
debug1: Config token is permitemptypasswords
debug3: /etc/ssh/sshd_config:61 setting PermitEmptyPasswords yes
debug1: Config token is printmotd
debug3: /etc/ssh/sshd_config:83 setting PrintMotd no
debug1: Config token is uselogin
debug3: /etc/ssh/sshd_config:86 setting UseLogin yes
debug1: Config token is subsystem
debug3: /etc/ssh/sshd_config:103 setting Subsystem sftp         internal-sftp
debug1: HPN Buffer Size: 32768
debug1: sshd version OpenSSH_5.2 QNX_Secure_Shell-20090621
debug3: Not a RSA1 key file /var/chroot/sshd/ssh_host_dsa_key.
debug1: read PEM private key done: type DSA
debug1: private host key: #0 type 2 DSA
debug1: rexec_argv[0]='/usr/sbin/sshd'
debug1: rexec_argv[1]='-ddd'
debug2: fd 3 setting O_NONBLOCK
debug1: Bind to port 22 on 0.0.0.0.
debug1: Server TCP RWIN socket size: 32768
debug1: HPN Buffer Size: 32768
Server listening on 0.0.0.0 port 22.
debug1: fd 4 clearing O_NONBLOCK
debug1: Server will not fork when running in debugging mode.
debug3: send_rexec_state: entering fd = 7 config len 317
debug3: ssh_msg_send: type 0
debug3: send_rexec_state: done
debug1: rexec start in 4 out 4 newsock 4 pipe -1 sock 7
debug1: inetd sockets after dupping: 3, 3
Connection from 192.168.1.71 port 45739
debug1: HPN Disabled: 0, HPN Buffer Size: 32768
debug1: Client protocol version 2.0; client software version OpenSSH_6.6.1p1 Ubuntu-2ubuntu2
SSH: Server;Ltype: Version;Remote: 192.168.1.71-45739;Protocol: 2.0;Client: OpenSSH_6.6.1p1 Ubuntu-2ubuntu2
debug1: match: OpenSSH_6.6.1p1 Ubuntu-2ubuntu2 pat OpenSSH*
debug1: Remote is NON-HPN aware
debug1: Enabling compatibility mode for protocol 2.0
debug1: Local version string SSH-2.0-OpenSSH_5.2 QNX_Secure_Shell-20090621-hpn13v6
debug2: fd 3 setting O_NONBLOCK
debug2: Network child is on pid 1245215
debug3: preauth child monitor started
debug3: mm_request_receive entering
debug3: privsep user:group 15:6
debug1: permanently_set_uid: 15/6
debug1: MYFLAG IS 1
debug1: list_hostkey_types: ssh-dss
debug1: SSH2_MSG_KEXINIT sent
debug1: SSH2_MSG_KEXINIT received
debug1: AUTH STATE IS 0
debug2: kex_parse_kexinit: diffie-hellman-group-exchange-sha256,diffie-hellman-group-exchange-sha1,diffie-hellman-gr1
debug2: kex_parse_kexinit: ssh-dss
debug2: kex_parse_kexinit: aes128-ctr,aes192-ctr,aes256-ctr,arcfour256,arcfour128,aes128-cbc,3des-cbc,blowfish-cbc,ce
debug2: kex_parse_kexinit: aes128-ctr,aes192-ctr,aes256-ctr,arcfour256,arcfour128,aes128-cbc,3des-cbc,blowfish-cbc,ce
debug2: kex_parse_kexinit: hmac-md5,hmac-sha1,hmac-ripemd160,hmac-ripemd160@openssh.com,hmac-sha1-96,hmac-md5-96
debug2: kex_parse_kexinit: hmac-md5,hmac-sha1,hmac-ripemd160,hmac-ripemd160@openssh.com,hmac-sha1-96,hmac-md5-96
debug2: kex_parse_kexinit: none,zlib@openssh.com
debug2: kex_parse_kexinit: none,zlib@openssh.com
debug2: kex_parse_kexinit: 
debug2: kex_parse_kexinit: 
debug2: kex_parse_kexinit: first_kex_follows 0 
debug2: kex_parse_kexinit: reserved 0 
debug2: kex_parse_kexinit: curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,dif1
debug2: kex_parse_kexinit: ssh-dss-cert-v01@openssh.com,ssh-dss-cert-v00@openssh.com,ssh-dss,ecdsa-sha2-nistp256-cera
debug2: kex_parse_kexinit: aes128-ctr,aes192-ctr,aes256-ctr,arcfour256,arcfour128,aes128-gcm@openssh.com,aes256-gcm@e
debug2: kex_parse_kexinit: aes128-ctr,aes192-ctr,aes256-ctr,arcfour256,arcfour128,aes128-gcm@openssh.com,aes256-gcm@e
debug2: kex_parse_kexinit: hmac-md5-etm@openssh.com,hmac-sha1-etm@openssh.com,umac-64-etm@openssh.com,umac-128-etm@o6
debug2: kex_parse_kexinit: hmac-md5-etm@openssh.com,hmac-sha1-etm@openssh.com,umac-64-etm@openssh.com,umac-128-etm@o6
debug2: kex_parse_kexinit: none,zlib@openssh.com,zlib
debug2: kex_parse_kexinit: none,zlib@openssh.com,zlib
debug2: kex_parse_kexinit: 
debug2: kex_parse_kexinit: 
debug2: kex_parse_kexinit: first_kex_follows 0 
debug2: kex_parse_kexinit: reserved 0 
debug2: mac_setup: found hmac-md5
debug1: REQUESTED ENC.NAME is 'aes128-ctr'
debug1: kex: client->server aes128-ctr hmac-md5 none
SSH: Server;Ltype: Kex;Remote: 192.168.1.71-45739;Enc: aes128-ctr;MAC: hmac-md5;Comp: none
debug2: mac_setup: found hmac-md5
debug1: REQUESTED ENC.NAME is 'aes128-ctr'
debug1: kex: server->client aes128-ctr hmac-md5 none
debug1: SSH2_MSG_KEX_DH_GEX_REQUEST received
debug3: mm_request_send entering: type 0
debug3: mm_choose_dh: waiting for MONITOR_ANS_MODULI
debug3: monitor_read: checking request 0
debug3: mm_request_receive_expect entering: type 1
debug3: mm_answer_moduli: got parameters: 1024 3072 8192
debug3: mm_request_receive entering
debug3: mm_request_send entering: type 1
debug2: monitor_read: 0 used once, disabling now
debug3: mm_request_receive entering
debug3: mm_choose_dh: remaining 0
debug1: SSH2_MSG_KEX_DH_GEX_GROUP sent
debug2: dh_gen_key: priv key bits set: 126/256
debug2: bits set: 1544/3072
debug1: expecting SSH2_MSG_KEX_DH_GEX_INIT
debug2: bits set: 1549/3072
debug3: mm_key_sign entering
debug3: mm_request_send entering: type 4
debug3: mm_key_sign: waiting for MONITOR_ANS_SIGN
debug3: monitor_read: checking request 4
debug3: mm_request_receive_expect entering: type 5
debug3: mm_answer_sign
debug3: mm_request_receive entering
debug3: mm_answer_sign: signature 15c160(55)
debug3: mm_request_send entering: type 5
debug2: monitor_read: 4 used once, disabling now
debug1: SSH2_MSG_KEX_DH_GEX_REPLY sent
debug3: mm_request_receive entering
debug2: kex_derive_keys
debug2: set_newkeys: mode 1
debug1: SSH2_MSG_NEWKEYS sent
debug1: expecting SSH2_MSG_NEWKEYS
debug2: set_newkeys: mode 0
debug1: SSH2_MSG_NEWKEYS received
debug1: KEX done
debug1: userauth-request for user root service ssh-connection method password
debug1: attempt 2 failures 1
debug2: input_userauth_request: try method password
debug3: mm_auth_password entering
debug3: mm_request_send entering: type 10
debug3: mm_auth_password: waiting for MONITOR_ANS_AUTHPASSWORD
debug3: monitor_read: checking request 10
debug3: mm_request_receive_expect entering: type 11
debug3: mm_request_receive entering
debug3: mm_answer_authpassword: sending result 1
debug3: mm_request_send entering: type 11
Accepted password for root from 192.168.1.71 port 45739 ssh2
debug3: mm_auth_password: user authenticated
debug1: monitor_child_preauth: root has been authenticated by privileged process
debug3: mm_get_keystate: Waiting for new keys
debug3: mm_request_receive_expect entering: type 24
debug3: mm_send_keystate: Sending new keys: 175ee0 15e220
debug3: mm_request_receive entering
debug3: mm_newkeys_to_blob: converting 175ee0
debug3: mm_newkeys_to_blob: converting 15e220
debug3: mm_send_keystate: New keys have been sent
debug3: mm_send_keystate: Sending compression state
debug3: mm_request_send entering: type 24
debug3: mm_send_keystate: Finished sending state
debug3: mm_newkeys_from_blob: 1682e8(118)
debug2: mac_setup: found hmac-md5
debug3: mm_get_keystate: Waiting for second key
debug3: mm_newkeys_from_blob: 1682e8(118)
debug2: mac_setup: found hmac-md5
debug3: mm_get_keystate: Getting compression state
debug3: mm_get_keystate: Getting Network I/O buffers
debug3: mm_share_sync: Share sync
debug3: mm_share_sync: Share sync end
debug2: set_newkeys: mode 0
debug2: set_newkeys: mode 1
debug1: Entering interactive session for SSH2.
debug2: fd 4 setting O_NONBLOCK
debug2: fd 5 setting O_NONBLOCK
debug1: server_init_dispatch_20
debug1: server_input_channel_open: ctype session rchan 0 win 1048576 max 16384
debug1: input_session_request
debug1: channel 0: new [server-session]
debug2: session_new: allocate (allocated 0 max 10)
debug3: session_unused: session id 0 unused
debug1: session_new: session 0
debug1: session_open: channel 0
debug1: session_open: session 0: link with channel 0
debug1: server_input_channel_open: confirm session
debug1: server_input_global_request: rtype no-more-sessions@openssh.com want_reply 0
debug1: server_input_channel_req: channel 0 request pty-req reply 1
debug1: session_by_channel: session 0 channel 0
debug1: session_input_channel_req: session 0 req pty-req
debug1: Allocating pty.
debug1: session_pty_req: session 0 alloc /dev/ttyp0
debug1: Ignoring unsupported tty mode opcode 62 (0x3e)
debug1: server_input_channel_req: channel 0 request env reply 0
debug1: session_by_channel: session 0 channel 0
debug1: session_input_channel_req: session 0 req env
debug2: Ignoring env request LANG: disallowed name
debug1: server_input_channel_req: channel 0 request shell reply 1
debug1: session_by_channel: session 0 channel 0
debug1: session_input_channel_req: session 0 req shell
debug2: fd 3 setting TCP_NODELAY
debug2: channel 0: rfd 8 isatty
debug2: fd 8 setting O_NONBLOCK
debug3: fd 6 is O_NONBLOCK
debug1: Setting controlling tty using TIOCSCTTY.
debug2: tcpwinsz: 33580 for connection: 3
debug2: channel 0: read<=0 rfd 8 len 0
debug2: channel 0: read failed
debug2: channel 0: close_read
debug2: channel 0: input open -> drain
debug2: channel 0: ibuf empty
debug2: channel 0: send eof
debug2: channel 0: input drain -> closed
debug2: notify_done: reading
debug1: Received SIGCHLD.
debug1: session_by_pid: pid 1249311
debug1: session_exit_message: session 0 channel 0 pid 1249311
debug2: channel 0: request exit-status confirm 0
debug1: session_exit_message: release channel 0
debug2: channel 0: write failed
debug2: channel 0: close_write
debug2: channel 0: send eow
debug2: channel 0: output open -> closed
debug1: session_pty_cleanup: session 0 release /dev/ttyp0
debug2: channel 0: send close
debug3: channel 0: will not send data after close
debug2: channel 0: rcvd close
Received disconnect from 192.168.1.71: 11: disconnected by user
debug1: do_cleanup
</pre></div>

Nothing in any of that appears to tell me why I couldn't log in.

Going back to the error from the message about login, eventually I downloaded the source code for OpenSSH to take a look at what it is doing, on the basis that the QNX version is a straight build of OpenSSH. Sure enough, the path to the login binary gets hardcoded into sshd, in ssh.h:

{% highlight c %}
#  define LOGIN_PROGRAM		"/usr/bin/login"
{% endhighlight %}

Theres some #ifdef logic which allows overriding this definition, but only as a parameter to the configure script at build time. No run time configuration of where to look for login is possible. When I then checked, sure enough, for this project the QNX filesystem had been populated with login under /bin/ not /usr/bin/.

When I moved login to where sshd was expecting it, lo and behold I was able to enter my password and get to a shell prompt over ssh.

So, an annoying problem becomes easy to solve, when you know where to get down to that next layer of needed information.
