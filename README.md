# Cloud-Automation
Cloud Automation using Ansible

---
- name: Create Digital Ocean Droplets
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    droplet_name_prefix: "CoryS-2504-WP"
    region: "nyc3"
    size: "s-1vcpu-1gb"
    image: "ubuntu-22-04-x64"
    ssh_key_name: "Cory-Searcy-2504"
    number_of_droplets: 2
    droplets:
      - CoryS-2504-WP1
      - CoryS-2504-WP2

  tasks:
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

