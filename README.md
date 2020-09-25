# Ansible Role: HAProxy

[![Build Status](https://travis-ci.org/geerlingguy/ansible-role-haproxy.svg?branch=master)](https://travis-ci.org/geerlingguy/ansible-role-haproxy)

Installs HAProxy on RedHat/CentOS and Debian/Ubuntu Linux servers.

**Note**: This role _officially_ supports HAProxy versions 1.4 or 1.5. Future versions may require some rework.


**see**: https://www.haproxy.com/de/blog/the-four-essential-sections-of-an-haproxy-configuration/


## Requirements

None.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

    haproxy_socket: /var/lib/haproxy/stats

The socket through which HAProxy can communicate (for admin purposes or statistics). To disable/remove this directive, set `haproxy_socket: ''` (an empty string).

    haproxy_chroot: /var/lib/haproxy

The jail directory where chroot() will be performed before dropping privileges. To disable/remove this directive, set `haproxy_chroot: ''` (an empty string). Only change this if you know what you're doing!

    haproxy_user: haproxy
    haproxy_group: haproxy

The user and group under which HAProxy should run. Only change this if you know what you're doing!

    haproxy_certificates

Optional: list of certificates to install

    haproxy_frontends:
      - name: 'hafrontend'
        bind: '127.0.0.1:666'
        mode: 'http'

HAProxy frontend configuration(s) directives as list.

    haproxy_backends:
      - name: 'habackend'
        mode: 'http'
        balance_method: 'roundrobin'
        httpchk: 'HEAD / HTTP/1.1\r\nHost:localhost'
        servers:
          # A list of backend servers (name and address) to which HAProxy will distribute requests.
          - server1 10.0.1.3:80 cookie server1
          - server2 10.0.1.4:80 cookie server2

HAProxy backend configuration(s) directives as list.


    haproxy_global_vars:
      - 'ssl-default-bind-ciphers ABCD+KLMJ:...'
      - 'ssl-default-bind-options no-sslv3'

A list of extra global variables to add to the global configuration section inside `haproxy.cfg`.

    haproxy_defaults_vars:

A list of extra defaults variables to add to the defaults section inside `haproxy.cfg`.

## Dependencies

None.

## Example Playbook

    - hosts: balancer
      sudo: yes
      roles:
        - { role: geerlingguy.haproxy }

## License

MIT / BSD

## Author Information

This role was created in 2015 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
