# Recu DMZ

Herramientas de utilidad:

```bash
sudo apt-get update
sudo apt-get dist-upgrade
sudo apt-get install nano
sudo apt-get install vim
# Tiene ifconfig
sudo apt-get install net-tools
# Tiene ping
sudo apt-get install iputils-ping
# Tiene nslookup
sudo apt-get install dnsutils
sudo apt-get install openssh-server
```

## Conexiones de red e la máquina virtual

Conector 1 (enp0s3): Conexión NAT
Conector 2 (enp0s8): Conexión Red interna: JGS_DMZ

## Obtención de IPs

`ip a`: para obtener la IP que se asigna a 'addresses'. Seguramente `10.0.2.15/24`.
`resolvectl`: para obtener la IP que se asigna a 'nameservers'. Seguramente `10.2.1.254`.
`ip route`: para obtener la IP que se asigna a 'routes'. Seguramente `10.0.2.2`.

## Modificar Netplan

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      # dhcp4: true
      addresses:
        - 10.0.2.15/24
      nameservers:
        addresses:
          - 10.2.1.254
      routes:
        - to: default
          via: 10.0.2.2
    enp0s8:
      addresses:
        - 192.168.207.50/24
```

Se aplican los cambios:

```bash
sudo netplan generate
sudo netplan apply
```

Comprobar conexión:

```bash
sudo apt-get update
```

## Instalar y configurar Bind9

### Instalación

```bash
sudo apt-get install ufw -y
sudo apt-get install bind9 bind9-utils -y
# Permitir el tráfico de Bind9 y SSH
sudo ufw allow bind9
sudo ufw allow ssh
# Reiniciar firewall
sudo systemctl restart ufw
# Comprobar estado firewall
sudo systemctl status ufw
# Comprobar el estado de Bind9
sudo systemctl status bind9
```

### Configuración

#### Configurar forwarders

```bash
sudo nano /etc/bind/named.conf.options
```

```bash
...
listen-on {
  any;
};

allow-query {
  localhost;
  192.168.207.0/24;
};

forwarders {
  10.0.2.2;
  // 10.2.1.254; Si no funciona la de arriba
};
...
dnssec-validation no;
```

Reiniciar el servicio de Bind9:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

#### Editar default named

```bash
sudo nano /etc/default/named
```

Modifcar la última línea:

```bash
OPTIONS="-u bind -4"
```

Reiniciar el servicio de Bind9:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

#### Enrutamiento

Instalar IP Tables y configurar enrutamiento:

```bash
sudo apt-get install iptables -y
# Habilitar reenvío de paquetes
sudo nano /etc/sysctl.conf
# Descomentar la siguiente línea
net.ipv4.ip_forward = 1
# Aplicar los cambios
sudo sysctl -p
# Configurar NAT
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
# Permitir tráfico entre interfaces
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
# Guardar configuración
sudo apt install iptables-persistent
sudo netfilter-persistent save
# Comprobar los cambios
sudo nano /etc/iptables/rules.v4
# Si se hacen cambios
sudo netfilter-persistent reload
```

## GIT

Configurar el archivo Netplan:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      # dhcp4: true
      addresses:
        - 192.168.207.100/24
      nameservers:
        addresses:
          - 192.168.207.50
      routes:
        - to: default
          via: 192.168.207.50
```

Aplicar cambios:

```bash
sudo netplan generate
sudo netplan apply
```

Actualizar máquina:

```bash
sudo apt-get update
```

Instalar Ping:

```bash
sudo apt-get install iputils-ping -y
```

Establecer conexión con el servidor DNS:

```bash
ping 192.168.207.50
```

## FTP

Configurar el archivo Netplan:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      # dhcp4: true
      addresses:
        - 192.168.207.200/24
      nameservers:
        addresses:
          - 192.168.207.50
      routes:
        - to: default
          via: 192.168.207.50
```

```bash
sudo apt-get install dnsutils iputils-ping -y

sudo apt-get install proftpd -y
```

Conexión con el servidor FTP mediante Filezilla:

Usuario: `uftp`
Contraseña: `uftp`

Conexión mediante comandos:

```bash
ftp 192.168.207.200
```

Usuario: `uftp`
Contraseña: `uftp`

## Servidor Web

Configurar el archivo Netplan:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      # dhcp4: true
      addresses:
        - 192.168.207.150/24
      nameservers:
        addresses:
          - 192.168.207.50
      routes:
        - to: default
          via: 192.168.207.50
```

Instalar Apache:

```bash
sudo apt install apache2
```

Habilitar Apache:

```bash
sudo systemctl enable apache2
```

Comprobar que Apache está funcionando:

```bash
sudo systemctl status apache2
```

Comprobar funcionamiento desde el cliente:

Apache: `192.168.207.150`
