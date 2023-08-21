- reference link: [Join Debian to AD domain](https://poweradm.com/join-ubuntu-debian-active-directory-domain/)

- apt -y install realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin oddjob oddjob-mkhomedir packagekit
- realm discover ad001.siemens.net --verbose
    - /usr/sbin/realm
- Set the attributes of Linux host to be stored in the computer account in Active Directory (operatingSystem and operatingSystemVersion attributes):
    - vim /etc/realmd.conf
    ```txt
    [active-directory]
    os-name = Debian GNU/Linux
    os-version = 12 (Bookwarm)
    ```
- Join to the AD domain
    - realm join -U AdministratorAccountName ad001.siemens.net
    - or define the OU:
        - realm join --verbose --user=AdministratorAccountName --computer-ou="OU=xx,OU=xx,DC=ad001,DC=siemens,DC=com" ad001.siemens.net
    - verify: realm list
- To automatically create a user home directory, run:
    ```txt
    bash -c "cat > /usr/share/pam-configs/mkhomedir" <<EOF
    Name: activate mkhomedir
    Default: yes
    Priority: 900
    Session-Type: Additional
    Session:
    required pam_mkhomedir.so umask=0022 skel=/etc/skel
    EOF
    ```
    - pam-auth-update
- allow domain user to login in via console and SSH
    - users: realm permit xxx@xxx xxx@xxx
    - group: realm permit -g xxx@xxx
    - allow or deny all domain users:
        - realm permit --all
        - realm deny --all
- allow users or group with sudo
    - vim /etc/sudoers.d/linux-admin
    ```txt
    %LinuxAdmins@xxx.com ALL=(ALL) ALL
    xxx@xxx.com ALL=(ALL) ALL
    ```
    - chmod 0440 /etc/sudoers.d/linux-admin