# Ansible Role: HAProxy

:exclamation: This is a detached fork of the [original role from geerlingguy/ansible-role-haproxy](https://github.com/geerlingguy/ansible-role-haproxy) including new features but still retaining backward compatibility. New features will not be issued as PRs to the original (see https://github.com/Rosa-Luxemburgstiftung-Berlin/ansible-role-haproxy/pull/7). :exclamation:

---

[![CI](https://github.com/Rosa-Luxemburgstiftung-Berlin/ansible-role-haproxy/actions/workflows/ci.yml/badge.svg)](https://github.com/Rosa-Luxemburgstiftung-Berlin/ansible-role-haproxy/actions/workflows/ci.yml)

Installs HAProxy on RedHat/CentOS and Debian/Ubuntu Linux servers.

**Note**: This role _officially_ supports HAProxy versions 1.4 - 2.6. Future versions may require some rework.

## Requirements

None.

## Role Variables

Available variables are listed below, along with default values (see [defaults/main.yml](defaults/main.yml)):

```yaml
haproxy_socket: /var/lib/haproxy/stats
haproxy_socket_level: admin
haproxy_socket_params: ''
```
The socket through which HAProxy can communicate (for admin purposes or statistics). To disable/remove this directive, set `haproxy_socket: ''` (an empty string).


```yaml
haproxy_chroot: /var/lib/haproxy
```
The jail directory where chroot() will be performed before dropping privileges. To disable/remove this directive, set `haproxy_chroot: ''` (an empty string). Only change this if you know what you're doing!


```yaml
haproxy_user: haproxy
haproxy_group: haproxy
```
The user and group under which HAProxy should run. Only change this if you know what you're doing!


```yaml
haproxy_frontend_name: 'hafrontend'
haproxy_frontend_bind_address: '*'
haproxy_frontend_port: 80
haproxy_frontend_mode: 'http'
haproxy_frontend_options: []
```
HAProxy frontend configuration directives. (Deprecated!)


```yaml
haproxy_frontends:
   - name: httplb
     port: 80
     mode: http
     bind_address: '*'
   - name: httpslb
     port: 443
     # bind_args: additional args to be appended to the bind line after {{ bind_address }}:{{ port }}
     # see: https://www.haproxy.com/blog/haproxy-ssl-termination/
     bind_args: "ssl crt {{ ansible_fqdn }}.pem"
     default_backend: habackend
     description: front of front
     option: # list of options
       - forwardfor
       - tcplog
       - ...
```
List of HAProxy frontend configuration directives.


```yaml
haproxy_backend_name: 'habackend'
haproxy_backend_mode: 'http'
haproxy_backend_balance_method: 'roundrobin'
haproxy_backend_httpchk: 'HEAD / HTTP/1.1\r\nHost:localhost'
haproxy_backend_options:
  - cookie SERVERID insert indirect
  - option forwardfor
```
HAProxy backend configuration directives. (Deprecated!)


```yaml
haproxy_backend_servers:
  - name: app1
    address: 192.168.0.1:80
    # params: defaults to 'cookie {{name}} check' if haproxy_backend_mode == 'http' for backward compatibility
  - name: app2
    address: 192.168.0.2:80
    params: maxconn 32 check
```
A list of backend servers (name and address) to which HAProxy will distribute requests.


```yaml
haproxy_defaults:
  - log global
  - mode http
  - option httplog
  - option dontlognull
```
HAProxy default settings.


```yaml
haproxy_connect_timeout: 5000
haproxy_client_timeout: 50000
haproxy_server_timeout: 50000
```
HAProxy default timeout configurations.


```yaml
haproxy_log:
  - "/dev/log  local0"
  - "/dev/log  local1 notice"
```
HAProxy global log directives.


```yaml
haproxy_global_vars:
  - 'ssl-default-bind-ciphers ABCD+KLMJ:...'
  - 'ssl-default-bind-options no-sslv3'
```
A list of extra global variables to add to the global configuration section inside `haproxy.cfg`.

## Dependencies

None.

## Example Playbook

```yaml
    - hosts: balancer
      sudo: yes
      roles:
        - { role: ansible-role-haproxy }
```

## Example Vars

```yaml
ignore_old_haproxy_vars: true
deprecate_old_haproxy_vars: true
haproxy_socket_level: admin
haproxy_socket_params: mode 660 expose-fd listeners
haproxy_global_vars:
  - stats timeout 30s
  - ca-base /etc/ssl/certs
  - crt-base /etc/ssl/private
  - ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
  - ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
  - ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
haproxy_defaults:
  - log global
  - mode http
  - option dontlognull
  - option  httplog
haproxy_connect_timeout: '5000'
haproxy_client_timeout: '50000'
haproxy_server_timeout: '50000'

haproxy_frontend_defaults:
  bind_address: "{{ ansible_default_ipv4.address }}"
  mode: http
  option:
    - httplog
    - forwardfor

haproxy_frontends:
  - name: front_http
    port: 80
    acl:
      - lets_encrypt path_beg /.well-known/acme-challenge/
    use_backend:
      - lets_encrypt if lets_encrypt
    http-request:
      # do NOT redirect letsencrypt requests
      - redirect scheme https unless lets_encrypt
  - name: front_https
    port: 443
    bind_args: ssl crt /etc/haproxy/haproxy.pem ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256 ssl-max-ver TLSv1.2
    acl:
      - host_1 hdr(host) -i host1.example.tld
      - host_1 hdr(host) -i www1.example.tld
      - host_2 hdr(host) -i host2.example.tld
      - host_2 hdr(host) -i www2.example.tld
    use_backend:
      - backend_1 if host_1
      - backend_2 if host_2
    http-request:
      - add-header X-Forwarded-Proto https
      - set-header Content-Security-Policy upgrade-insecure-requests

haproxy_backend_defaults:
  http-request:
    - add-header X-Forwarded-Proto https
    - set-header Content-Security-Policy upgrade-insecure-requests

haproxy_backends:
  - name: lets_encrypt
    servers:
      - name: local
        address: "127.0.0.1:8080"
  - name: backend_1
    mode: http
    option:
      - forwardfor
    servers:
      - name: host1_1
        address: "10.224.2.10:8443"
        params: ssl verify none
      - name: host1_2
        address: "10.224.2.11:8443"
        params: ssl verify none
  - name: backend_2
    mode: http
    option:
      - forwardfor
    servers:
      - name: host2
        address: "10.224.2.6:443"
        params: ssl verify none
```

## Caveats

`check_mode` will not work until haproxy is installed!

## License

MIT / BSD

## Author Information

This role was created in 2015 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).

Sept 2022 [![@zerwes](https://avatars.githubusercontent.com/u/9654986?s=32&v=4)](https://github.com/zerwes)
for [![RLS](https://avatars.githubusercontent.com/u/35101423?s=32&v=4)](https://github.com/Rosa-Luxemburgstiftung-Berlin) :
started the current default `main` branch in this repository intended as a detached fork of the original repository due to inactivity and stagnation on the original repo. (see https://github.com/Rosa-Luxemburgstiftung-Berlin/ansible-role-haproxy/pull/7)

