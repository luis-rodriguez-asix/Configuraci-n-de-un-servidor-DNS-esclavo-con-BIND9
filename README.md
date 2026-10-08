# Configuración de un servidor DNS esclavo con BIND9

Práctica de configuración de un servidor DNS secundario con BIND9 en Debian. La estructura se adapta a la guía de referencia de Francisco Javier Cruces Doval y documenta el laboratorio con las capturas propias almacenadas en [`Evidencia/`](Evidencia/), colocadas en orden cronológico dentro de cada fase.

**Guía de referencia:** [Configuración de un servidor DNS esclavo con BIND9](https://www.javiercd.es/posts/servicios/dns/bind9/dns_esclavo/dns_esclavo/)

> Los dominios, nombres e IP que aparecen en los bloques de configuración son ejemplos. Deben sustituirse por los valores reales del laboratorio.

## Objetivo

Configurar un servidor DNS esclavo para que descargue y mantenga automáticamente una réplica de las zonas directa e inversa publicadas por un servidor DNS maestro.

## Escenario de laboratorio

| Equipo | Nombre de ejemplo | IP de ejemplo | Función |
|---|---|---:|---|
| DNS maestro | `dns1.empresa.test` | `192.168.10.1` | Mantiene las zonas originales y permite las transferencias. |
| DNS esclavo | `dns2.empresa.test` | `192.168.10.200` | Recibe una copia de las zonas y responde como secundario. |
| Cliente | `cliente1` | `192.168.10.20` | Comprueba la resolución DNS. |

Zonas de ejemplo:

- Directa: `empresa.test`
- Inversa: `10.168.192.in-addr.arpa`

## 1. Preparar el servidor esclavo

En el servidor que actuará como esclavo, configura su nombre de host y su nombre completo. En `/etc/hostname` usa, por ejemplo:

```text
dns2
```

Añade la asociación local correspondiente en `/etc/hosts`:

```text
127.0.1.1 dns2.empresa.test dns2
```

Comprueba el FQDN:

```bash
hostname -f
```

La salida esperada es `dns2.empresa.test`.

**Evidencia 1 — Preparación e identificación del servidor DNS esclavo**

![Preparación del entorno](Evidencia/Captura%20de%20pantalla%202026-10-08%20194535.png)

**Evidencia 2 — Configuración del hostname y comprobación del FQDN**

![Comprobación del FQDN](Evidencia/Captura%20de%20pantalla%202026-10-08%20194552.png)

## 2. Instalar BIND9

Actualiza el sistema e instala BIND9, las utilidades de diagnóstico, la documentación y `rsync`:

```bash
sudo apt update && sudo apt install -y bind9 bind9utils bind9-doc dnsutils rsync
```

Habilita el servicio y comprueba que está activo:

```bash
sudo systemctl enable --now bind9
sudo systemctl status bind9 --no-pager
```

**Evidencia 3 — Instalación de BIND9 y herramientas de administración DNS**

![Instalación de BIND9](Evidencia/Captura%20de%20pantalla%202026-10-08%20194608.png)

## 3. Configurar opciones de BIND

Edita `/etc/bind/named.conf.options` en el servidor esclavo. El siguiente ejemplo permite consultas desde la red del laboratorio, habilita la recursión y configura reenviadores:

```conf
options {
    directory "/var/cache/bind";

    allow-query { 127.0.0.1; 192.168.10.0/24; };
    recursion yes;

    dnssec-validation no;

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };
};
```

Limita `allow-query` a las redes que deban utilizar el resolver. En un entorno real, aplica la política DNSSEC de tu organización.

**Evidencia 4 — Configuración de `named.conf.options`**

![Opciones de BIND](Evidencia/Captura%20de%20pantalla%202026-10-08%20194622.png)

## 4. Declarar las zonas esclavas

Edita `/etc/bind/named.conf.local` en `dns2` y define las zonas que serán transferidas desde el maestro:

```conf
zone "empresa.test" {
    type slave;
    masters { 192.168.10.1; };
    file "/var/cache/bind/slaves/db.empresa.test";
};

zone "10.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.10.1; };
    file "/var/cache/bind/slaves/db.192.168.10";
};
```

Crea el directorio donde BIND guardará las copias transferidas y asigna los permisos al usuario del servicio:

```bash
sudo mkdir -p /var/cache/bind/slaves
sudo chown bind:bind /var/cache/bind/slaves
```

**Evidencia 5 — Declaración de la zona directa como esclava**

![Zona directa esclava](Evidencia/Captura%20de%20pantalla%202026-10-08%20194640.png)

**Evidencia 6 — Declaración de la zona inversa y directorio de réplicas**

![Zona inversa y directorio de réplicas](Evidencia/Captura%20de%20pantalla%202026-10-08%20194657.png)

Valida la sintaxis y reinicia BIND:

```bash
sudo named-checkconf
sudo systemctl restart bind9
sudo journalctl -xeu bind9
```

**Evidencia 7 — Validación de la configuración y reinicio de BIND9**

![Validación y reinicio](Evidencia/Captura%20de%20pantalla%202026-10-08%20194714.png)

Comprueba que el secundario recibe o puede recibir las copias de zona:

```bash
sudo ls -lh /var/cache/bind/slaves
```

**Evidencia 8 — Comprobación de zonas recibidas y registros de BIND**

![Comprobación de transferencia](Evidencia/Captura%20de%20pantalla%202026-10-08%20194734.png)

## 5. Autorizar transferencias en el maestro

En el servidor `dns1`, edita `/etc/bind/named.conf.local`. Para evitar que la zona sea expuesta a hosts no autorizados, permite transferencias solamente a la IP del esclavo. `also-notify` avisa al secundario cuando se produce un cambio:

```conf
zone "empresa.test" {
    type master;
    file "/var/cache/bind/db.empresa.test";
    allow-transfer { 192.168.10.200; };
    also-notify { 192.168.10.200; };
};

zone "10.168.192.in-addr.arpa" {
    type master;
    file "/var/cache/bind/db.192.168.10";
    allow-transfer { 192.168.10.200; };
    also-notify { 192.168.10.200; };
};
```

**Evidencia 9 — Autorización de transferencias desde el maestro al esclavo**

![Allow transfer en el maestro](Evidencia/Captura%20de%20pantalla%202026-10-08%20194754.png)

## 6. Revisar las zonas del maestro

La zona directa del maestro debe contener los dos servidores autoritativos y el registro A del secundario:

```dns
$TTL 86400
@ IN SOA dns1.empresa.test. root.empresa.test. (
    2026100801 ; Incrementar en cada cambio
    604800     ; Refresh
    86400      ; Retry
    2419200    ; Expire
    86400      ; Negative Cache TTL
)
;
@       IN NS dns1.empresa.test.
@       IN NS dns2.empresa.test.

$ORIGIN empresa.test.
dns1    IN A 192.168.10.1
dns2    IN A 192.168.10.200
www     IN A 192.168.10.3
```

**Evidencia 10 — Zona directa del maestro con registros NS y A del esclavo**

![Zona directa maestro](Evidencia/Captura%20de%20pantalla%202026-10-08%20194941.png)

La zona inversa debe resolver la dirección IP del secundario hacia su nombre completo:

```dns
$TTL 86400
@ IN SOA dns1.empresa.test. root.empresa.test. (
    2026100801 ; Incrementar en cada cambio
    604800
    86400
    2419200
    86400
)
;
@ IN NS dns1.empresa.test.
@ IN NS dns2.empresa.test.

$ORIGIN 10.168.192.in-addr.arpa.
1   IN PTR dns1.empresa.test.
200 IN PTR dns2.empresa.test.
```

**Evidencia 11 — Zona inversa del maestro con el registro PTR del esclavo**

![Zona inversa maestro](Evidencia/Captura%20de%20pantalla%202026-10-08%20195004.png)

Antes de aplicar cambios, valida ambos archivos de zona y recarga el servicio:

```bash
sudo named-checkconf
sudo named-checkzone empresa.test /var/cache/bind/db.empresa.test
sudo named-checkzone 10.168.192.in-addr.arpa /var/cache/bind/db.192.168.10
sudo rndc reload
```

**Evidencia 12 — Validación, recarga y sincronización de las zonas DNS**

![Recarga y sincronización](Evidencia/Captura%20de%20pantalla%202026-10-08%20195039.png)

## 7. Pruebas de resolución

Para verificar que maestro y esclavo responden con los mismos datos, realiza consultas explícitas contra ambos servidores:

```bash
dig @192.168.10.1 empresa.test SOA +noall +answer
dig @192.168.10.200 empresa.test SOA +noall +answer

dig @192.168.10.1 www.empresa.test A +noall +answer
dig @192.168.10.200 www.empresa.test A +noall +answer

dig @192.168.10.200 -x 192.168.10.200 +noall +answer
```

Los seriales SOA de ambos servidores deben coincidir. Cuando modifiques un registro en el maestro, incrementa obligatoriamente el serial SOA, valida el fichero y ejecuta `sudo rndc reload empresa.test`; después comprueba que el cambio también se responde desde el esclavo.

**Evidencia 13 — Prueba final de resolución DNS desde el maestro y el esclavo**

![Prueba final con dig](Evidencia/Captura%20de%20pantalla%202026-10-08%20195057.png)

## Resolución de problemas

| Incidencia | Comprobación |
|---|---|
| BIND no se inicia | Ejecuta `sudo named-checkconf` y consulta `sudo journalctl -xeu bind9`. |
| El esclavo no descarga la zona | Comprueba `masters`, `allow-transfer`, conectividad TCP/53 y permisos de `/var/cache/bind/slaves`. |
| El esclavo muestra datos antiguos | Aumenta el serial SOA en el maestro, valida la zona y recárgala con `rndc reload`. |
| Una consulta devuelve `NXDOMAIN` | Comprueba que el registro exista en el maestro, que el serial se haya incrementado y que la transferencia haya finalizado. |
| Los clientes no consultan el servidor | Revisa `allow-query`, la configuración del cliente y las reglas de firewall UDP/TCP 53. |

## Buenas prácticas

- Restringe siempre `allow-transfer` a las IP de los secundarios autorizados.
- Usa TSIG para autenticar transferencias en entornos de producción.
- Incrementa el serial SOA en toda modificación.
- Mantén más de un servidor autoritativo para mejorar la disponibilidad.
- Conserva las capturas y los logs como evidencia de la práctica.

## Créditos

La estructura técnica de esta práctica se basa en la guía de Francisco Javier Cruces Doval, [Configuración de un servidor DNS esclavo con BIND9](https://www.javiercd.es/posts/servicios/dns/bind9/dns_esclavo/dns_esclavo/), publicada el 10 de mayo de 2025. El texto ha sido adaptado y las evidencias son capturas propias almacenadas en este repositorio.
