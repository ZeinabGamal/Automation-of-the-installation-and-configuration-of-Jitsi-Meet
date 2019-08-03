# Automation of the installation and configuration of Jitsi-Meet

Ansible is used ot automate the installation process.

If you don't have SSL keys for the domain yet, consider using the excellent thefinn93.letsencrypt Ansible role to obtain (free!) SSL certs from LetsEncrypt.

we will need to expose ports for the Jitsi Meet components to work. By default the role will use ufw to allow these ports.In case you use AWS,you'll need to expose those ports in the associated Security Group.

to avoid to re-invent the wheel, i use existing ansible roles made by others. For Jitsi, I found this project: https://github.com/freedomofpress/ansible-role-jitsi-meet

Even if the role is not up-to-date (last commit is from 2017). So, I did this little playbook (jitsi.yml)



```ymal
- name: Configure jitsi-meet server.
  hosts: [conference]
  user: amarok
  vars:
    jitsi_meet_server_name: conference.facil.services
    jitsi_meet_lang: fr
  roles:
    - role: thefinn93.letsencrypt
      become: yes
      letsencrypt_email: "webmaster@{{ jitsi_meet_server_name }}"
      letsencrypt_cert_domains:
        - "{{ jitsi_meet_server_name }}"
      tags: letsencrypt

    - role: freedomofpress.jitsi-meet
      jitsi_meet_ssl_cert_path: "/etc/letsencrypt/live/{{ jitsi_meet_server_name }}/fullchain.pem"
      jitsi_meet_ssl_key_path: "/etc/letsencrypt/live/{{ jitsi_meet_server_name }}/privkey.pem"
      jitsi_meet_configure_nginx: true
      become: yes
      tags: jitsi

  post_tasks:
    # https://github.com/jitsi/jicofo/blob/master/README.md#certificates
    - name: Copy root certificate from prosody
      become: true
      copy:
        remote_src: yes
        src: "/var/lib/prosody/{{ jitsi_meet_server_name }}.crt"
        dest: "/usr/local/share/ca-certificates/"
    - name: Copy auth certificate from prosody
      become: true
      copy:
        remote_src: yes
        src: "/var/lib/prosody/auth.{{ jitsi_meet_server_name }}.crt"
        dest: "/usr/local/share/ca-certificates/"
    - name: Run update-ca-certificates for Jicofo
      become: true
      command: update-ca-certificates
```

## Running the tests
This role uses Molecule and ServerSpec for testing. To use it:

```yaml
pip install molecule
gem install serverspec
molecule test
```

You can also run selective commands:

```yaml
molecule idempotence
molecule verify
```


