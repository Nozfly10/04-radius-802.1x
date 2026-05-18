# 04 — Authentification Réseau RADIUS & 802.1X

## 🎯 Objectif
Mettre en place une authentification réseau centralisée avec FreeRADIUS et le protocole 802.1X afin de contrôler l'accès au réseau et n'autoriser que les équipements et utilisateurs légitimes.

---

## 🏗️ Architecture

```
[Poste client]                [Switch Cisco]              [Serveur RADIUS]
    |                               |                            |
    |--- 802.1X EAPOL request ----> |                            |
    |                               |--- RADIUS Access-Request ->|
    |                               |<-- RADIUS Access-Accept ---|
    |<--- Accès autorisé ---------- |                            |
```

---

## 📋 Composants

| Composant | Rôle |
|---|---|
| FreeRADIUS | Serveur d'authentification |
| Switch Cisco | Authenticator 802.1X |
| Active Directory | Base d'utilisateurs |
| Poste client | Supplicant (Windows/Linux) |

---

## ⚙️ Installation FreeRADIUS

```bash
sudo apt update && sudo apt install freeradius freeradius-ldap -y
sudo systemctl enable freeradius
```

---

## ⚙️ Configuration FreeRADIUS

### /etc/freeradius/3.0/clients.conf
```
client switch-cisco {
    ipaddr  = 192.168.1.254
    secret  = cle_partagee_cisco
    nas_type = cisco
}
```

### /etc/freeradius/3.0/users
```
# Utilisateurs locaux
benjamin    Cleartext-Password := "MonMotDePasse"
            Service-Type = Framed-User,
            Tunnel-Type = VLAN,
            Tunnel-Medium-Type = IEEE-802,
            Tunnel-Private-Group-Id = "20"

admin       Cleartext-Password := "AdminPass"
            Service-Type = Framed-User,
            Tunnel-Private-Group-Id = "10"
```

### Intégration Active Directory (LDAP)
```
# /etc/freeradius/3.0/mods-enabled/ldap
ldap {
    server   = "192.168.1.10"
    port     = 389
    identity = "CN=radius,OU=Services,DC=benjamin,DC=local"
    password = "ldap_password"
    base_dn  = "DC=benjamin,DC=local"
    user {
        base_dn = "OU=Utilisateurs,${..base_dn}"
        filter  = "(sAMAccountName=%{%{Stripped-User-Name}:-%{User-Name}})"
    }
}
```

---

## ⚙️ Configuration Switch Cisco (802.1X)

```
! Activer AAA
Switch(config)# aaa new-model
Switch(config)# aaa authentication dot1x default group radius
Switch(config)# aaa authorization network default group radius

! Configurer le serveur RADIUS
Switch(config)# radius server FREERADIUS
Switch(config-radius-server)# address ipv4 192.168.1.20 auth-port 1812 acct-port 1813
Switch(config-radius-server)# key cle_partagee_cisco

! Activer 802.1X globalement
Switch(config)# dot1x system-auth-control

! Activer 802.1X sur les ports
Switch(config)# interface range fa0/1-20
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# dot1x port-control auto
Switch(config-if-range)# authentication host-mode single-host
```

---

## ✅ Tests et vérification

```bash
# Test d'authentification RADIUS
radtest benjamin MonMotDePasse 127.0.0.1 0 cle_partagee_cisco

# Logs en temps réel
sudo freeradius -X

# Vérification sur le switch
Switch# show dot1x all
Switch# show authentication sessions
```

---

## 🎓 Compétences acquises

- Déploiement et configuration de FreeRADIUS
- Protocole d'authentification 802.1X (EAP)
- Intégration RADIUS avec Active Directory (LDAP)
- Configuration AAA sur switch Cisco
- Attribution dynamique de VLAN par authentification
