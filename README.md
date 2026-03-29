# HAProxy Ansible

Déploiement et configuration automatisés de HAProxy via Ansible, pour deux modes de fonctionnement : **TCP (passthrough SNI)** et **HTTP (terminaison SSL)**.

## Architecture

Deux instances HAProxy distinctes sont gérées :

| Instance | IP | Mode | Rôle |
|---|---|---|---|
| `haproxy_tcp` | 192.168.20.5 | TCP | Routage SNI, SSL passthrough |
| `haproxy_http` | 192.168.20.6 | HTTP | Reverse proxy, terminaison SSL |

### Flux TCP

```
Client → HAProxy (port 443) → inspection SNI → backend correspondant (SSL non terminé)
```

Domaines routés :
- `devver.app` → 192.168.45.99:443
- `nextcloud.vrhometech.fr` → 192.168.20.6:443
- `*.media.vrhometech.fr` → 192.168.10.2:443

### Flux HTTP

```
Client → HAProxy (port 443, SSL terminé) → détection WebSocket → backend
```

Vhosts configurés :
- `nextcloud.vrhometech.fr` → 192.168.20.3:11000 (avec support WebSocket)

## Structure

```
haproxy/
└── ansible/
    ├── ansible.cfg
    ├── inventory/
    │   ├── hosts
    │   └── group_vars/
    │       ├── all.yml             # Variables globales
    │       ├── haproxy_http.yml    # Vhosts HTTP
    │       └── haproxy_tcp.yml     # Vhosts TCP
    ├── playbooks/
    │   ├── install-haproxy.yml
    │   ├── configure-haproxy-http.yml
    │   ├── configure-haproxy-tcp.yml
    │   └── acme-cloudflare.yml
    └── roles/
        ├── haproxy_install/        # Installation du paquet
        ├── haproxy_config_http/    # Config mode HTTP + SSL
        ├── haproxy_config_tcp/     # Config mode TCP + SNI
        └── haproxy_acme/          # Certificats Let's Encrypt (DNS Cloudflare)
```

## Utilisation

```bash
cd ansible/

# Installation de HAProxy
ansible-playbook playbooks/install-haproxy.yml

# Configuration TCP (SNI passthrough)
ansible-playbook playbooks/configure-haproxy-tcp.yml

# Configuration HTTP (reverse proxy + SSL)
ansible-playbook playbooks/configure-haproxy-http.yml

# Provisionnement des certificats Let's Encrypt
ansible-playbook playbooks/acme-cloudflare.yml
```

## Certificats SSL

Les certificats sont générés via **Let's Encrypt** avec validation DNS Cloudflare (challenge DNS-01). Cela permet la validation sans exposer les ports 80/443.

- Plugin : `certbot-dns-cloudflare`
- Credentials : `/etc/letsencrypt/cloudflare.ini` (déployé par Ansible)
- Format HAProxy : certificat + clé privée concaténés en `.pem`
- Renouvellement automatique : cron journalier à 3h du matin avec rechargement HAProxy

## Prérequis

- Ansible installé sur la machine de contrôle
- Accès SSH aux serveurs (`~/.ssh/yes`)
- Token API Cloudflare (dans `group_vars/all.yml`)
- Les serveurs cibles sous Debian/Ubuntu
