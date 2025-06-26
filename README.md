# Cloud-Automation
Cloud Automation using Ansible
---
- name: Create Digital Ocean Droplets
  hosts: localhost
  connection: local
  gather_facts: false

- vars:
    droplet_name_prefix: "CoryS-2504-WP"
    region: "nyc3"
    size: "s-1vcpu-1gb"
    image: "ubuntu-22-04-x64"
    ssh_key_name: "Cory-Searcy-2504"
    number_of_droplets: 2
    droplets:
      - CoryS-2504-WP1
      - CoryS-2504-WP2

- tasks:
    - name: Create SSH Key in Digital Ocean
      digital_ocean_sshkey:
        name: "{{ ssh_key_name }}"
        ssh_pub_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
      register: ssh_key_result

    - name: Create Droplets
      digital_ocean_droplet:
        name: "{{ droplet_name_prefix }}{{ item }}"
        region: "{{ region }}"
        size: "{{ size }}"
        image: "{{ image }}"
        ssh_keys: "{{ ssh_key_result.data.ssh_key.id }}"
        unique_name: yes
        state: present
      loop: "{{ range(1, number_of_droplets + 1) | list }}"
      register: droplet_result

    - name: Wait for droplets to be created
      pause:
        seconds: 30

    - name: Add new droplets to host group
      add_host:
        name: "{{ item.networks.v4[0].ip_address }}"
        groups: droplets
      loop: "{{ droplets_info.data }}"
      when: item.networks is defined and item.networks.v4 | length > 0

lamp_wordpress.yml

---
- name: Install LAMP and deploy WordPress on Ubuntu
  hosts: webservers
  become: true
  vars_files:
    - group_vars/all.yml

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Apache, MySQL, PHP and dependencies
      apt:
        name:
          - apache2
          - mysql-server
          - php
          - php-mysql
          - php-curl
          - php-gd
          - php-mbstring
          - php-xml
          - php-xmlrpc
          - php-soap
          - php-intl
          - unzip
          - curl
          - libapache2-mod-php
        state: present

    - name: Start and enable Apache
      service:
        name: apache2
        state: started
        enabled: yes

    - name: Start and enable MySQL
      service:
        name: mysql
        state: started
        enabled: yes

    - name: Ensure Python MySQL library is installed
      apt:
        name: python3-pymysql
        state: present

    - name: Set MySQL root password
      mysql_user:
        name: root
        host: localhost
        password: "{{ mysql_root_password }}"
        login_unix_socket: /var/run/mysqld/mysqld.sock
      ignore_errors: true

    - name: Create WordPress database
      mysql_db:
        name: "{{ db_name }}"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"


    - name: Create WordPress user
      mysql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        priv: "{{ db_name }}.*:ALL"
        host: "%"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"

    - name: Download WordPress
      get_url:
        url: "{{ wordpress_url }}"
        dest: /tmp/latest.tar.gz

    - name: Extract WordPress
      unarchive:
        src: /tmp/latest.tar.gz
        dest: /var/www/html/
        remote_src: yes

    - name: Set permissions for WordPress directory
      file:
        path: /var/www/html/wordpress
        state: directory
        recurse: yes
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Copy wp-config.php
      template:
        src: wp-config.php.j2
        dest: /var/www/html/wordpress/wp-config.php
        owner: www-data
        group: www-data
        mode: '0644'

    - name: Enable Apache rewrite module
      apache2_module:
        name: rewrite
        state: present

    - name: Restart Apache
      service:
        name: apache2
        state: restarted

# System-Hardening
System Hardening with Ansible using DIA STIGs

---
- name: Apply DISA STIG Hardening to Ubuntu Systems hosts: all
become: yes
vars:
    min_password_length: 14
    max_password_age: 60
    password_warn_age: 7
tasks:
- name: Ensure auditd is installed apt:
        name: auditd
        state: present
        update_cache: yes
- name: Ensure auditd is enabled and started systemd:
        name: auditd
        enabled: yes
        state: started
- name: Ensure libpam-pwquality is installed
 8
  apt:
    name: libpam-pwquality
    state: present
- name: Set password maximum age lineinfile:
    path: /etc/login.defs
    regexp: '^PASS_MAX_DAYS'
    line: "PASS_MAX_DAYS {{ max_password_age }}"
- name: Set password minimum length lineinfile:
    path: /etc/security/pwquality.conf
    regexp: '^minlen'
    line: "minlen = {{ min_password_length }}"
- name: Set password warning age lineinfile:
path: /etc/login.defs
regexp: '^PASS_WARN_AGE'
line: "PASS_WARN_AGE {{ password_warn_age }}"
- name: Disable core dumps sysctl:
    name: fs.suid_dumpable
    value: '0'
9

    state: present
    reload: yes
- name: Set sticky bit on /tmp command: chmod +t /tmp
- name: Ensure permissions on /etc/shadow are correct file:
    path: /etc/shadow
    owner: root
    group: shadow
    mode: '0640'
- name: Ensure permissions on /etc/passwd are correct file:
    path: /etc/passwd
    owner: root
    group: root
    mode: '0644'
- name: Ensure permissions on /etc/group are correct file:
    path: /etc/group
    owner: root
    group: root
    mode: '0644'
10

- name: Disable USB storage (if applicable) lineinfile:
    path: /etc/modprobe.d/disable-usb.conf
    create: yes
    line: "install usb-storage /bin/true"
- name: Configure sysctl for IP spoofing protection sysctl:
    name: net.ipv4.conf.all.rp_filter
    value: '1'
    state: present
    reload: yes
- name: Disable ICMP redirects sysctl:
name: net.ipv4.conf.all.accept_redirects value: '0'
state: present
reload: yes
- name: Enable ExecShield (if supported) sysctl:
    name: kernel.exec-shield
    value: '1'
    state: present
11

    reload: yes
  ignore_errors: yes
- name: Set permissions on /etc/cron.d file:
    path: /etc/cron.d
    state: directory
    owner: root
    group: root
    mode: '0700'
- name: Restrict cron to authorized users copy:
    dest: /etc/cron.allow
    content: "root\n"
    owner: root
    group: root
mode: '0600'
- name: Disable IPv6 if not needed sysctl:
    name: net.ipv6.conf.all.disable_ipv6
    value: '1'
    state: present
    reload: yes
12

- name: Ensure default umask is 027 lineinfile:
        path: /etc/profile
        regexp: '^umask'
        line: 'umask 027'
- name: Lock inactive user accounts after 35 days lineinfile:
        path: /etc/default/useradd
        regexp: '^INACTIVE='
        line: 'INACTIVE=35'
- name: Enforce PAM password complexity lineinfile:
        path: /etc/pam.d/common-password
regexp: '^password\s+requisite\s+pam_pwquality.so'
line: 'password requisite pam_pwquality.so retry=3 minlen={{ min_password_length }} ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1'
- name: Ensure rsyslog is installed apt:
        name: rsyslog
        state: present
- name: Ensure rsyslog is enabled and running
13

  systemd:
    name: rsyslog
    enabled: yes
    state: started
- name: Ensure permissions on rsyslog.conf are correct file:
    path: /etc/rsyslog.conf
    owner: root
    group: root
    mode: '0644'
- name: Ensure telnet is not installed apt:
    name: telnet
    state: absent
- name: Ensure rsh-server is not installed apt:
    name: rsh-server
    state: absent
- name: Ensure xinetd is not installed apt:
    name: xinetd
    state: absent
14

- name: Disable Ctrl+Alt+Del reboot copy:
dest: /etc/systemd/system/ctrl-alt-del.target content: ''
owner: root
group: root
mode: '0644'
- name: Remove .netrc files (insecure) find:
    paths: /home
    patterns: '.netrc'
    recurse: yes
  register: netrc_files
- name: Delete .netrc files file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ netrc_files.files }}"
  when: netrc_files.matched > 0
- name: Disable SSH root login lineinfile:
15

    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
    create: yes
    backup: yes
  notify: restart ssh
- name: Set SSH idle timeout to 10 minutes lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^ClientAliveInterval'
    line: 'ClientAliveInterval 600'
    create: yes
  notify: restart ssh
- name: Set SSH session disconnect after 3 failures lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^MaxAuthTries'
    line: 'MaxAuthTries 3'
    create: yes
  notify: restart ssh
- name: Set login banner (DoD warning) copy:
    dest: /etc/issue
16

•
content: |
You are accessing a U.S. Government (USG) Information System
        that is provided for USG-authorized use only.
      owner: root
      group: root
      mode: '0644'
handlers:
- name: restart auditd
    service:
      name: auditd
      state: restarted
- name: restart ssh service:
name: ssh
      state: restarted
  Now we are ready to run all of our playbooks.
(IS)
17
