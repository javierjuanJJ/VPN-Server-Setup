# VPN Server Setup

Guía completa, paso a paso, para montar tu **propio servidor VPN** sobre un servidor Linux con IP pública. El objetivo es cifrar tu tráfico cuando uses redes no confiables (WiFi público, hoteles, aeropuertos) y poder acceder a tu red privada de forma remota.

Elige **una** de las dos soluciones:

- **Opción 1 — WireGuard** (recomendada): moderno, rápido, sencillo y soportado por el kernel Linux.
- **Opción 2 — OpenVPN**: maduro, muy compatible y con amplio soporte en clientes antiguos.

> Documentación técnica obtenida y contrastada con Context7 a partir de las fuentes oficiales de [WireGuard](https://www.wireguard.com/) y [OpenVPN](https://openvpn.net/community-resources/).

---

## Índice

1. [Objetivo del proyecto](#objetivo-del-proyecto)
2. [Requisitos previos](#requisitos-previos)
3. [Conceptos clave](#conceptos-clave)
4. [Opción 1 — WireGuard (recomendada)](#opción-1--wireguard-recomendada)
5. [Opción 2 — OpenVPN](#opción-2--openvpn)
6. [Después de configurar el servidor (ambas opciones)](#después-de-configurar-el-servidor-ambas-opciones)
7. [Objetivos avanzados (stretch goals)](#objetivos-avanzados-stretch-goals)
8. [Solución de problemas](#solución-de-problemas)
9. [Resultados de aprendizaje](#resultados-de-aprendizaje)
10. [Referencias](#referencias)

---

## Objetivo del proyecto

Configurar tu propio servidor VPN para:

- Proteger tu tráfico en redes no confiables.
- Acceder a tu red privada (casa/oficina) desde cualquier lugar.
- Entender técnicamente cómo funcionan los túneles, el cifrado y el enrutado de tráfico.

---

## Requisitos previos

Antes de empezar necesitas:

- **Un servidor Linux con IP pública** (VPS de cualquier proveedor: DigitalOcean, Hetzner, AWS, etc.). Se recomienda Debian/Ubuntu.
- **Conocimientos básicos de línea de comandos** Linux.
- **Firewall configurado** (UFW o iptables).
- **Acceso root o sudo** en el servidor.
- Nociones básicas de redes: direcciones IP, puertos y enrutado.

Convenciones usadas en esta guía:

| Concepto | Valor de ejemplo | Descripción |
|---|---|---|
| IP pública del servidor | `203.0.113.10` | Sustitúyela por la tuya |
| Interfaz de red externa | `eth0` | Detéctala con `ip -4 route ls \| grep default` |
| Subred de la VPN | `10.8.0.0/24` | Red interna del túnel |
| IP del servidor en la VPN | `10.8.0.1` | Gateway del túnel |
| Puerto WireGuard | `51820/udp` | Por defecto |
| Puerto OpenVPN | `1194/udp` | Por defecto |

Detecta primero tu interfaz externa y tu IP pública:

```bash
# Interfaz que sale a Internet (ajusta 1.1.1.1 si quieres)
ip -4 route ls | grep default

# IP pública del servidor
curl -4 ifconfig.me
```

---

## Conceptos clave

- **Túnel (tunneling):** los paquetes de tu dispositivo se encapsulan dentro de otro paquete cifrado que viaja hasta el servidor VPN, donde se desencapsulan y salen a Internet.
- **Cifrado:** WireGuard usa criptografía moderna (Curve25519, ChaCha20-Poly1305); OpenVPN usa OpenSSL/TLS.
- **IP forwarding:** el servidor debe reenviar paquetes entre la interfaz del túnel y la interfaz de Internet.
- **NAT (masquerade):** traduce la IP privada del túnel a la IP pública del servidor para que el tráfico salga a Internet.
- **AllowedIPs / rutas:** definen qué tráfico entra al túnel (todo `0.0.0.0/0`, o solo subredes concretas para *split tunneling*).

---

## Opción 1 — WireGuard (recomendada)

WireGuard es un protocolo VPN moderno, extremadamente rápido y mucho más simple de configurar que OpenVPN.

### 1.1 Instalar WireGuard en el servidor

```bash
sudo apt update
sudo apt install -y wireguard wireguard-tools qrencode
```

En distribuciones basadas en RHEL/Fedora:

```bash
sudo dnf install -y wireguard-tools qrencode
```

Verifica que el módulo del kernel esté disponible:

```bash
sudo modprobe wireguard && lsmod | grep wireguard
```

> En VPS con kernel antiguo puede hacer falta instalar el módulo DKMS (`wireguard-dkms`), aunque hoy casi todos los kernels ya lo incluyen.

### 1.2 Habilitar IP forwarding

Crea un archivo de configuración persistente:

```bash
sudo tee /etc/sysctl.d/99-wireguard-forward.conf > /dev/null <<'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

sudo sysctl --system
```

Comprueba que se aplicó:

```bash
sysctl net.ipv4.ip_forward        # debe devolver 1
```

### 1.3 Generar las claves del servidor

```bash
cd /etc/wireguard
umask 077   # evita que las claves queden legibles por otros usuarios

wg genkey | tee server_private.key | wg pubkey > server_public.key

cat server_private.key
cat server_public.key
```

> `umask 077` es importante: WireGuard avisa (y es un riesgo de seguridad) si las claves tienen permisos de lectura para otros usuarios.

### 1.4 Crear el archivo `wg0.conf` del servidor

Localiza tu interfaz externa (`eth0` a continuación) y edita `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <CLAVE_PRIVADA_DEL_SERVIDOR>

# Reglas NAT y reenvío. Sustituye eth0 por tu interfaz externa real.
PostUp   = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

Pega tu clave privada real `cat /etc/wireguard/server_private.key`. Los clientes (bloques `[Peer]`) se añaden más adelante, en el paso **1.7**.

> `%i` es sustituido automáticamente por WireGuard con el nombre de la interfaz (`wg0`).

### 1.5 Abrir el puerto en el firewall

Con **UFW**:

```bash
sudo ufw allow 51820/udp
sudo ufw allow OpenSSH      # ¡no te bloquees el SSH!

# Para que el tráfico reenviado funcione con UFW, permite el reenvío:
sudo sed -i 's/^DEFAULT_FORWARD_POLICY=.*/DEFAULT_FORWARD_POLICY="ACCEPT"/' /etc/default/ufw
sudo ufw reload
```

Con **iptables**:

```bash
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -A FORWARD -i wg0 -j ACCEPT
sudo iptables -A FORWARD -o wg0 -j ACCEPT
```

### 1.6 Iniciar y habilitar el servicio

El servicio `wg-quick@wg0` levanta la interfaz a partir del archivo `/etc/wireguard/wg0.conf`:

```bash
sudo systemctl enable --now wg-quick@wg0
sudo systemctl status wg-quick@wg0
```

Comprueba el estado de la interfaz:

```bash
sudo wg show
ip a show wg0          # debe mostrar 10.8.0.1/24
```

### 1.7 Crear la configuración de un cliente

Genera un par de claves por cada cliente:

```bash
cd /etc/wireguard
umask 077
wg genkey | tee cliente1_private.key | wg pubkey > cliente1_public.key
```

Añade el cliente como `[Peer]` al final de `/etc/wireguard/wg0.conf`:

```ini
[Peer]
# cliente1
PublicKey = <CLAVE_PUBLICA_DE_CLIENTE1>
AllowedIPs = 10.8.0.2/32
```

Aplica los cambios **sin cortar las conexiones activas**:

```bash
sudo wg syncconf wg0 <(wg-quick strip wg0)
```

O reinicia el servicio (esto sí corta las sesiones):

```bash
sudo systemctl restart wg-quick@wg0
```

Crea el archivo del cliente `cliente1.conf`:

```ini
[Interface]
Address = 10.8.0.2/32
PrivateKey = <CLAVE_PRIVADA_DE_CLIENTE1>
DNS = 1.1.1.1

[Peer]
PublicKey = <CLAVE_PUBLICA_DEL_SERVIDOR>
Endpoint = 203.0.113.10:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

- `AllowedIPs = 0.0.0.0/0, ::/0` enruta **todo** el tráfico por la VPN (*full tunnel*).
- `PersistentKeepalive = 25` mantiene el túnel vivo detrás de NAT.

### 1.8 Generar un código QR para móviles

```bash
qrencode -t ansiutf8 < /etc/wireguard/cliente1.conf
```

Escanea el QR con la app oficial de WireGuard (Android/iOS) o importa el archivo `.conf` en el escritorio.

---

## Opción 2 — OpenVPN

OpenVPN es maduro y muy compatible, pero requiere gestionar una PKI (infraestructura de clave pública) con certificados.

### 2.1 Instalar OpenVPN y Easy-RSA

```bash
sudo apt update
sudo apt install -y openvpn easy-rsa
```

### 2.2 Inicializar la PKI y crear la CA

Easy-RSA construye la Autoridad Certificadora (CA). Copia la plantilla a `/etc/openvpn/easy-rsa`:

```bash
make-cadir /etc/openvpn/easy-rsa 2>/dev/null || sudo cp -r /usr/share/easy-rsa /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa

./easyrsa init-pki
./easyrsa build-ca nopass
```

`build-ca` crea la CA en `pki/ca.crt` con su clave `pki/private/ca.key`.

### 2.3 Generar el parámetro Diffie-Hellman

```bash
./easyrsa gen-dh
```

Esto genera `pki/dh.pem` (puede tardar varios minutos). Es recomendable generarlo en el propio servidor.

### 2.4 Crear el certificado del servidor

```bash
./easyrsa gen-req server nopass
./easyrsa sign-req server server
```

Genera también la clave `tls-auth` (o `tls-crypt` en versiones nuevas), que añade una capa de autenticación a la negociación TLS:

```bash
openvpn --genkey secret /etc/openvpn/ta.key
```

### 2.5 Copiar los archivos necesarios

```bash
cd /etc/openvpn
sudo cp easy-rsa/pki/ca.crt .
sudo cp easy-rsa/pki/issued/server.crt .
sudo cp easy-rsa/pki/private/server.key .
sudo cp easy-rsa/pki/dh.pem .
sudo chmod 600 server.key ta.key
```

### 2.6 Crear `server.conf`

Crea `/etc/openvpn/server/server.conf` (Debian/Ubuntu moderno usa el directorio `server/`):

```conf
# Interfaz virtual y protocolo
dev tun
proto udp
port 1194

# Certificados y claves
ca   ca.crt
cert server.crt
key  server.key
dh   dh.pem
tls-auth ta.key 0

# Reutilizar la clave para mejorar el rendimiento
topology subnet
server 10.8.0.0 255.255.255.0

# Mantener la clave y el túnel entre reconexiones del cliente
persist-key
persist-tun

# Rutas: enviar TODO el tráfico por la VPN
push "redirect-gateway def1 bypass-dhcp"

# Servidores DNS que se envían a los clientes
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 1.0.0.1"

keepalive 10 120
cipher AES-256-GCM
auth SHA256

user nobody
group nogroup

status /var/log/openvpn/openvpn-status.log
verb 3
```

> `redirect-gateway def1 bypass-dhcp` hace que el cliente envíe todo su tráfico por la VPN y es la pieza clave del *full tunnel*.

### 2.7 Habilitar IP forwarding y NAT

```bash
sudo tee /etc/sysctl.d/99-openvpn-forward.conf > /dev/null <<'EOF'
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

Reglas NAT y de reenvío (sustituye `eth0` por tu interfaz externa):

```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i tun0 -j ACCEPT
sudo iptables -A FORWARD -o tun0 -j ACCEPT
```

Para hacerlas persistentes:

```bash
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

### 2.8 Abrir el puerto en el firewall

```bash
sudo ufw allow 1194/udp
sudo ufw allow OpenSSH
```

### 2.9 Iniciar y habilitar el servicio

En Debian/Ubuntu modernos:

```bash
sudo systemctl enable --now openvpn-server@server
sudo systemctl status openvpn-server@server
```

En instalaciones que usan `/etc/openvpn/server.conf`:

```bash
sudo systemctl enable --now openvpn@server
```

Comprueba que la interfaz `tun0` está arriba:

```bash
ip a show tun0
```

### 2.10 Crear el certificado y el archivo `.ovpn` de un cliente

```bash
cd /etc/openvpn/easy-rsa
./easyrsa gen-req cliente1 nopass
./easyrsa sign-req client cliente1
```

Ahora construye un `.ovpn` con los certificados **en línea** (auto-contenido):

```bash
cd /etc/openvpn
cat > /tmp/cliente1.ovpn <<EOF
client
dev tun
proto udp
remote 203.0.113.10 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA256
key-direction 1
verb 3
EOF

echo '<ca>'         >> /tmp/cliente1.ovpn
cat ca.crt          >> /tmp/cliente1.ovpn
echo '</ca>'        >> /tmp/cliente1.ovpn
echo '<cert>'       >> /tmp/cliente1.ovpn
cat easy-rsa/pki/issued/cliente1.crt >> /tmp/cliente1.ovpn
echo '</cert>'      >> /tmp/cliente1.ovpn
echo '<key>'        >> /tmp/cliente1.ovpn
cat easy-rsa/pki/private/cliente1.key >> /tmp/cliente1.ovpn
echo '</key>'       >> /tmp/cliente1.ovpn
echo '<tls-auth>'   >> /tmp/cliente1.ovpn
cat ta.key          >> /tmp/cliente1.ovpn
echo '</tls-auth>'  >> /tmp/cliente1.ovpn
```

El archivo `/tmp/cliente1.ovpn` es todo lo que necesita el cliente. Si usas un cliente antiguo que no soporte `_auth`, puedes añadir la CA en un bloque `<ca>` al final (ya se hace arriba).

---

## Después de configurar el servidor (ambas opciones)

### Instalar el cliente

- **Android / iOS:** app oficial *WireGuard* u *OpenVPN Connect*.
- **Windows / macOS / Linux:** cliente oficial de WireGuard, o `openvpn` + NetworkManager para OpenVPN.

Importa el archivo `cliente1.conf` (WireGuard) o `cliente1.ovpn` (OpenVPN).

### Conectar y verificar que el tráfico pasa por la VPN

1. Conéctate desde el cliente.
2. Comprueba que tu **IP pública cambió** a la del servidor:

```bash
curl -4 ifconfig.me
```

Debe devolver la IP pública del VPS (`203.0.113.10`), no la de tu operador.

3. Verifica el servidor WireGuard con las estadísticas de peers:

```bash
sudo wg show
```

Verás el `latest handshake` y los bytes transferidos (`transfer`).

### Probar DNS (evitar fugas)

Una fuga de DNS ocurre cuando el cliente sigue usando el DNS local y no el del túnel.

- Visita [https://dnsleaktest.com](https://dnsleaktest.com) o [https://ipleak.net](https://ipleak.net).
- En OpenVPN, asegúrate de que las directivas `push "dhcp-option DNS ..."` están activas.
- En WireGuard, la línea `DNS = 1.1.1.1` del `[Interface]` es la que configura el DNS del cliente.
- Comprueba resolución desde el cliente:

```bash
nslookup example.com
dig +short example.com
```

Si el test muestra el DNS de tu operador, revisa la configuración y reconecta.

### Añadir múltiples clientes (móvil, portátil, tablet)

Repite el paso de creación de cliente asignando **una IP distinta** dentro de la subred del túnel a cada dispositivo:

| Dispositivo | IP en la VPN |
|---|---|
| Móvil | `10.8.0.2/32` |
| Portátil | `10.8.0.3/32` |
| Tablet | `10.8.0.4/32` |

- **WireGuard:** añade un bloque `[Peer]` por cliente en `wg0.conf` y aplica con `wg syncconf`.
- **OpenVPN:** genera un par `gen-req` / `sign-req` por cliente y un `.ovpn` por dispositivo.

---

## Objetivos avanzados (stretch goals)

### Split tunneling (solo parte del tráfico por la VPN)

**WireGuard** — cambia `AllowedIPs` en el cliente para incluir solo lo que quieres enrutar:

```ini
[Peer]
PublicKey = <CLAVE_PUBLICA_DEL_SERVIDOR>
Endpoint = 203.0.113.10:51820
# Solo la subred del túnel y tu LAN privada, el resto sale directo
AllowedIPs = 10.8.0.0/24, 192.168.1.0/24
```

**OpenVPN** — elimina `push "redirect-gateway def1"` del servidor y añade rutas concretas:

```conf
# En server.conf
push "route 192.168.1.0 255.255.255.0"
```

O usa `route-nopull` en el cliente y define solo las rutas que quieras.

### Pi-hole o AdGuard Home con la VPN (bloqueo de anuncios)

Instala Pi-hole y úsalo como DNS del túnel:

```bash
curl -sSL https://install.pi-hole.net | bash
```

1. Permite que Pi-hole escuche en todas las interfaces (incluida `wg0`):

```bash
pihole-FTL --config dns.listeningMode "ALL"
```

2. Retrasa el arranque de Pi-hole para dar tiempo a que la interfaz `wg0` exista:

```bash
pihole-FTL --config mic.delay_startup 5
```

3. Apunta el DNS de los clientes a la IP de la VPN del servidor (Pi-hole):

```ini
[Interface]
Address = 10.8.0.2/32
DNS = 10.8.0.1
```

> **Seguridad:** no expongas Pi-hole como resolutor abierto a Internet. `listeningMode = ALL` debe combinarse con el firewall para limitar el acceso a la subred del túnel. AdGuard Home se configura de forma equivalente y también sirve como DNS con filtrado.

### Conexión automática en redes no confiables

- **WireGuard:** `PersistentKeepalive = 25` mantiene el túnel; usa tareas/programas como `wg-quick` o el cliente de escritorio con reconexión automática.
- **Móvil:** activa la opción "Conectar siempre" / "on-demand" en la app para que se levante al salir de casa.
- **Escritorio Linux:** puedes automatizarlo con NetworkManager o un `systemd` unit que arranque `wg-quick@wg0`.

### Monitorización de clientes y ancho de banda

```bash
# Peers conectados, handshake y tráfico (WireGuard)
sudo wg show all dump

# Estado de conexiones OpenVPN
cat /var/log/openvpn/openvpn-status.log

# Ancho de banda por interfaz
sudo apt install -y vnstat
vnstat -i wg0
```

---

## Solución de problemas

| Síntoma | Causa probable / solución |
|---|---|
| El cliente conecta pero no navega | Falta IP forwarding o reglas NAT `MASQUERADE`. Verifica `sysctl net.ipv4.ip_forward`. |
| WireGuard no hace handshake | Puerto `51820/udp` cerrado, o firewall bloquea. Revisa `ufw status` y el security group del VPS. |
| Sin acceso a Internet en el VPS tras configurar | Asegúrate de mantener abierto el puerto SSH (22) en UFW. |
| Fuga de DNS | Revisa `DNS =` (WireGuard) o `push "dhcp-option DNS ..."` (OpenVPN). |
| OpenVPN: `Cannot open TUN/TAP` | Falta el módulo `tun` o permisos; reinstala `openvpn` y verifica `/dev/net/tun`. |
| `wg0.conf` con permisos inseguros | `chmod 600 /etc/wireguard/wg0.conf` y claves con `umask 077`. |
| No veo el peer en `wg show` | El cliente no ha hecho handshake aún, o la clave pública está mal copiada. |

---

## Resultados de aprendizaje

Al completar este proyecto serás capaz de:

- Entender cómo funcionan las VPN a nivel técnico: **tunneling**, **cifrado** y **enrutado de tráfico**.
- Levantar y administrar un servidor **WireGuard** u **OpenVPN** en producción.
- Gestionar **claves y certificados** (Curve25519/OpenSSL + Easy-RSA).
- Configurar **IP forwarding, NAT y firewall** correctamente.
- Proteger tu tráfico en redes no confiables y **acceder a recursos privados** de forma remota.
- Aplicar estos conocimientos a seguridad personal, trabajo remoto y conexiones seguras a infraestructura en la nube.

---

## Referencias

- WireGuard — [Sitio oficial](https://www.wireguard.com/) · [Quickstart](https://www.wireguard.com/quickstart/) · [wireguard-tools](https://github.com/wireguard/wireguard-tools)
- OpenVPN — [Documentación comunitaria](https://openvpn.net/community-resources/) · [Manual 2.x](https://openvpn.net/community-resources/reference-manual-for-openvpn-2-6/)
- Easy-RSA — [Repositorio oficial](https://github.com/OpenVPN/easy-rsa)
- Pi-hole — [Documentación oficial](https://docs.pi-hole.net/) · [Guía WireGuard + Pi-hole](https://docs.pi-hole.net/guides/vpn/wireguard/server/)
