# Configuración de un servidor DNS esclavo con BIND9

Documentación de la práctica para desplegar un servidor DNS secundario con BIND9 en Debian. La estructura técnica se ha adaptado de la guía de Francisco Javier Cruces Doval, **Configuración de un servidor DNS esclavo con BIND9** (10 de mayo de 2025), ajustando los nombres, las direcciones IP y las evidencias a este laboratorio.

**Guía de referencia:** [javiercd.es — Configuración de un servidor DNS esclavo con BIND9](https://www.javiercd.es/posts/servicios/dns/bind9/dns_esclavo/dns_esclavo/)

> Sustituye los valores de ejemplo por las IP, dominio y nombres usados realmente en tu red. No publiques claves TSIG privadas ni datos sensibles.

## Objetivo

Configurar un servidor DNS esclavo que reciba automáticamente desde el servidor maestro las zonas directa e inversa. El secundario permite mantener la resolución de nombres disponible y repartir las consultas dentro de la red.

## Escenario de laboratorio

| Equipo | Nombre | Dirección IP | Función |
|---|---|---:|---|
| Servidor maestro | `dns1.empresa.test` | `192.168.10.1` | Aloja las zonas maestras y autoriza la transferencia. |
| Servidor esclavo | `dns2.empresa.test` | `192.168.10.200` | Descarga y sirve una copia de las zonas. |
| Cliente | `cliente1` | `192.168.10.20` | Realiza consultas de verificación. |

Zonas utilizadas en el ejemplo:

- Zona directa: `empresa.test`
- Zona inversa: `10.168.192.in-addr.arpa`

## 1. Preparar el DNS esclavo

### Asignar el nombre del equipo

En el servidor esclavo, establece el nombre del host en `/etc/hostname`:

```text
dns2
```

Configura también `/etc/hosts` para asociar el FQDN con el host local:

```text
127.0.1.1 dns2.empresa.test dns2
```

Comprueba que el nombre completo se resuelve correctamente:

```bash
hostname -f
```

La salida esperada es:

```text
dns2.empresa.test
```

### Instalar BIND9

Actualiza los repositorios e instala BIND9, las herramientas de consulta, la documentación y `rsync`:

```bash
sudo apt update && sudo apt install -y bind9 bind9utils bind9-doc dnsutils rsync
```

Activa el servicio y confirma que se inicia:

```bash
sudo systemctl enable --now bind9
sudo systemctl status bind9 --no-pager
```

## 2. Configuración básica del esclavo

Edita `/etc/bind/named.conf.options` en `dns2`. Este ejemplo permite consultas desde localhost y desde la red interna, habilita recursión y define reenviadores DNS públicos:

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

En producción, limita `allow-query` a tus redes necesarias y revisa la política DNSSEC de tu organización.

## 3. Declarar las zonas esclavas

Edita `/etc/bind/named.conf.local` en el servidor `dns2`:

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

Crea el directorio que contendrá las copias transferidas. El usuario `bind` debe disponer de permisos para escribir en él:

```bash
sudo mkdir -p /var/cache/bind/slaves
sudo chown bind:bind /var/cache/bind/slaves
```

Verifica la sintaxis y reinicia el servicio:

```bash
sudo named-checkconf
sudo systemctl restart bind9
sudo journalctl -xeu bind9
```

Comprueba que BIND ha descargado los ficheros de zona:

```bash
ls -l /var/cache/bind/slaves
```

## 4. Autorizar al esclavo en el maestro

En `dns1`, edita `/etc/bind/named.conf.local`. El maestro debe permitir transferencias exclusivamente a la IP del esclavo:

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

> El uso de `also-notify` hace que el maestro avise al esclavo al recargar una zona. Aun así, el esclavo también comprueba periódicamente el serial SOA según el intervalo `Refresh`.

### Zona directa en el maestro

El archivo `/var/cache/bind/db.empresa.test` puede contener una estructura como esta:

```dns
$TTL 86400
@ IN SOA dns1.empresa.test. root.empresa.test. (
    2026100801 ; Serial: incrementar en cada cambio
    604800     ; Refresh
    86400      ; Retry
    2419200    ; Expire
    86400      ; Negative Cache TTL
)
;
@       IN NS dns1.empresa.test.
@       IN NS dns2.empresa.test.
@       IN MX 10 correo.empresa.test.

$ORIGIN empresa.test.
dns1    IN A 192.168.10.1
dns2    IN A 192.168.10.200
correo  IN A 192.168.10.2
thor    IN A 192.168.10.3
hela    IN A 192.168.10.4
www     IN CNAME thor
informatica IN CNAME thor
ftp     IN CNAME hela
```

### Zona inversa en el maestro

El archivo `/var/cache/bind/db.192.168.10` puede definirse así:

```dns
$TTL 86400
@ IN SOA dns1.empresa.test. root.empresa.test. (
    2026100801 ; Serial: incrementar en cada cambio
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
2   IN PTR correo.empresa.test.
3   IN PTR thor.empresa.test.
4   IN PTR hela.empresa.test.
200 IN PTR dns2.empresa.test.
```

Valida y recarga BIND en el maestro:

```bash
sudo named-checkconf
sudo named-checkzone empresa.test /var/cache/bind/db.empresa.test
sudo named-checkzone 10.168.192.in-addr.arpa /var/cache/bind/db.192.168.10
sudo rndc reload
```

## 5. Pruebas de transferencia y resolución

### Confirmar la transferencia de zonas

En el esclavo, observa los eventos de BIND mientras se reinicia o recarga el servicio en el maestro:

```bash
sudo journalctl -u named -f
```

También puedes revisar los ficheros transferidos:

```bash
sudo ls -lh /var/cache/bind/slaves
```

### Probar una modificación en el maestro

1. Añade en la zona directa del maestro un nuevo registro:

```dns
sentinel IN A 192.168.10.25
```

2. Incrementa el serial SOA, por ejemplo de `2026100801` a `2026100802`. Este paso es obligatorio: si el serial no aumenta, el esclavo no descargará la nueva versión de la zona.

3. Valida el fichero y recarga la zona:

```bash
sudo named-checkzone empresa.test /var/cache/bind/db.empresa.test
sudo rndc reload empresa.test
```

4. Como comprobación local del secundario, puedes sincronizar los datos en disco:

```bash
sudo rndc sync
```

### Consultar ambos servidores

Desde un cliente, las dos consultas deben devolver el mismo registro A:

```bash
dig @192.168.10.1 sentinel.empresa.test A +noall +answer
dig @192.168.10.200 sentinel.empresa.test A +noall +answer
```

Resultado esperado en ambos casos:

```text
sentinel.empresa.test. 86400 IN A 192.168.10.25
```

Comprueba también la zona inversa:

```bash
dig @192.168.10.200 -x 192.168.10.200 +noall +answer
```

## Evidencias y descripción de capturas

Las capturas siguientes son las evidencias almacenadas en `Evidencia/`. Cada una está asociada a una etapa de la configuración o a una prueba que debe aparecer en el laboratorio.

| Nº | Captura | Descripción de configuración o prueba |
|---:|---|---|
| 1 | ![Evidencia 1](Evidencia/Captura%20de%20pantalla%202026-10-08%20194535.png) | Preparación del entorno del DNS esclavo: identificación del servidor y comprobación inicial del sistema. |
| 2 | ![Evidencia 2](Evidencia/Captura%20de%20pantalla%202026-10-08%20194552.png) | Configuración del nombre de host y verificación del FQDN mediante `hostname -f`. |
| 3 | ![Evidencia 3](Evidencia/Captura%20de%20pantalla%202026-10-08%20194608.png) | Instalación de BIND9 y de las herramientas necesarias para administración y diagnóstico DNS. |
| 4 | ![Evidencia 4](Evidencia/Captura%20de%20pantalla%202026-10-08%20194622.png) | Revisión de la configuración base de BIND en `named.conf.options`, incluyendo la red autorizada y los reenviadores. |
| 5 | ![Evidencia 5](Evidencia/Captura%20de%20pantalla%202026-10-08%20194640.png) | Declaración de la zona directa como esclava en `named.conf.local`, indicando la IP del maestro y el archivo de réplica. |
| 6 | ![Evidencia 6](Evidencia/Captura%20de%20pantalla%202026-10-08%20194657.png) | Declaración de la zona inversa como esclava y preparación del directorio `/var/cache/bind/slaves` para las transferencias. |
| 7 | ![Evidencia 7](Evidencia/Captura%20de%20pantalla%202026-10-08%20194714.png) | Validación de la sintaxis de BIND y reinicio del servicio en el servidor secundario. |
| 8 | ![Evidencia 8](Evidencia/Captura%20de%20pantalla%202026-10-08%20194734.png) | Comprobación de los registros de BIND y de la recepción de las zonas transferidas en el esclavo. |
| 9 | ![Evidencia 9](Evidencia/Captura%20de%20pantalla%202026-10-08%20194754.png) | Configuración del maestro: autorización de transferencias de zona al servidor DNS esclavo mediante `allow-transfer`. |
| 10 | ![Evidencia 10](Evidencia/Captura%20de%20pantalla%202026-10-08%20194941.png) | Actualización de la zona directa del maestro con los registros NS y A correspondientes a `dns2`. |
| 11 | ![Evidencia 11](Evidencia/Captura%20de%20pantalla%202026-10-08%20195004.png) | Configuración o actualización de la zona inversa para que la IP del esclavo resuelva a su FQDN. |
| 12 | ![Evidencia 12](Evidencia/Captura%20de%20pantalla%202026-10-08%20195039.png) | Recarga de BIND y observación de los eventos de transferencia o sincronización de zonas. |
| 13 | ![Evidencia 13](Evidencia/Captura%20de%20pantalla%202026-10-08%20195057.png) | Prueba final con `dig`: verificación de que maestro y esclavo responden de forma coherente a las consultas DNS. |

> Las descripciones se han organizado según la secuencia técnica de la guía de referencia. Revisa que cada descripción corresponda visualmente con el contenido de su captura antes de entregar la práctica.

## Resolución de problemas

| Problema | Comprobación y solución |
|---|---|
| El servicio BIND no inicia | Ejecuta `sudo named-checkconf` y revisa `sudo journalctl -xeu bind9`. |
| El esclavo no descarga las zonas | Confirma que `masters` apunta al maestro correcto, que `allow-transfer` contiene la IP real del esclavo, que TCP/53 está permitido y que el directorio de destino pertenece a `bind:bind`. |
| El secundario conserva registros antiguos | Aumenta el serial SOA en el maestro, valida el fichero y ejecuta `sudo rndc reload`. |
| El nombre devuelve `NXDOMAIN` en el esclavo | Comprueba que el cambio está en la zona del maestro, que el serial se incrementó y que la transferencia se refleja en los logs. |
| Las consultas externas fallan | Revisa `allow-query`, las reglas de firewall UDP/TCP 53 y la configuración de red. |

## Buenas prácticas

- Restringe `allow-transfer` solo a los servidores secundarios autorizados.
- Protege transferencias de zona con TSIG en entornos de producción.
- Incrementa siempre el serial SOA tras cualquier modificación de registros.
- Mantén al menos dos servidores DNS autoritativos para mejorar la disponibilidad.
- Conserva las capturas de pruebas, los logs relevantes y los archivos de configuración como evidencias del laboratorio.

## Créditos

La secuencia de configuración se basa en la guía de Francisco Javier Cruces Doval: [Configuración de un servidor DNS esclavo con BIND9](https://www.javiercd.es/posts/servicios/dns/bind9/dns_esclavo/dns_esclavo/), publicada el 10 de mayo de 2025. El contenido se ha adaptado al presente repositorio y a sus evidencias.
