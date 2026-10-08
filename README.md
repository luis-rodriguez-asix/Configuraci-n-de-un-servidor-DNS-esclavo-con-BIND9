# Configuración de un servidor DNS esclavo con BIND9

Guía práctica para instalar, configurar y verificar una arquitectura DNS maestro-esclavo (primario-secundario) con BIND9 en sistemas Debian/Ubuntu. Incluye evidencias disponibles en el directorio [`Evidencia/`](Evidencia/).

> **Nota de seguridad:** las direcciones IP, el dominio y las claves de ejemplo deben sustituirse por los datos reales del laboratorio. No publiques claves TSIG privadas reales.

## Objetivos

- Instalar BIND9 en los servidores maestro y esclavo.
- Definir una zona directa autoritativa en el servidor maestro.
- Permitir la transferencia de zona exclusivamente al DNS esclavo.
- Configurar el servidor esclavo para obtener una copia automática de la zona.
- Verificar la resolución DNS y la transferencia AXFR.
- Comprobar la actualización de la zona después de incrementar el número de serie SOA.

## Topología de ejemplo

| Equipo | Función | IP de ejemplo |
|---|---|---|
| `dns-maestro` | DNS primario / maestro | `192.168.1.10` |
| `dns-esclavo` | DNS secundario / esclavo | `192.168.1.11` |
| `cliente` | Equipo de pruebas | `192.168.1.20` |

Dominio de ejemplo: `empresa.test`

## Requisitos previos

- Dos máquinas Debian o Ubuntu con conectividad IP entre ellas.
- Privilegios mediante `sudo`.
- IPs estáticas o reservas DHCP para ambos servidores.
- Firewall que permita DNS TCP y UDP por el puerto 53 entre maestro, esclavo y clientes autorizados. El protocolo TCP es imprescindible para las transferencias de zona.

En ambos servidores, actualiza el índice de paquetes e instala BIND9 junto con las utilidades de prueba:

```bash
sudo apt update
sudo apt install -y bind9 bind9-utils dnsutils
```

Activa y comprueba el servicio:

```bash
sudo systemctl enable --now bind9
sudo systemctl status bind9 --no-pager
```

## Configuración del DNS maestro

### Declarar la zona

Edita `/etc/bind/named.conf.local` en `dns-maestro`:

```bash
sudo nano /etc/bind/named.conf.local
```

Añade la definición de la zona. La directiva `allow-transfer` restringe la transferencia al servidor esclavo:

```conf
zone "empresa.test" {
    type master;
    file "/etc/bind/db.empresa.test";
    allow-transfer { 192.168.1.11; };
    also-notify { 192.168.1.11; };
};
```

### Crear el fichero de zona directa

Crea `/etc/bind/db.empresa.test`:

```bash
sudo nano /etc/bind/db.empresa.test
```

Contenido de ejemplo:

```dns
$TTL 86400
@   IN  SOA dns-maestro.empresa.test. admin.empresa.test. (
        2026100801 ; Serial: YYYYMMDDNN
        3600       ; Refresh
        900        ; Retry
        604800     ; Expire
        86400      ; Negative Cache TTL
)

; Servidores autoritativos
@               IN  NS      dns-maestro.empresa.test.
@               IN  NS      dns-esclavo.empresa.test.

; Registros A
dns-maestro     IN  A       192.168.1.10
dns-esclavo     IN  A       192.168.1.11
www             IN  A       192.168.1.30
app             IN  A       192.168.1.31
```

Valida la sintaxis antes de reiniciar:

```bash
sudo named-checkconf
sudo named-checkzone empresa.test /etc/bind/db.empresa.test
sudo systemctl restart bind9
```

La salida esperada de `named-checkzone` termina con `OK`.

## Configuración del DNS esclavo

En `dns-esclavo`, declara la misma zona en `/etc/bind/named.conf.local`:

```bash
sudo nano /etc/bind/named.conf.local
```

```conf
zone "empresa.test" {
    type slave;
    file "/var/cache/bind/db.empresa.test";
    masters { 192.168.1.10; };
};
```

La ruta `/var/cache/bind/` se utiliza porque el proceso de BIND puede escribir ahí la copia descargada desde el maestro. Comprueba la configuración y reinicia el servicio:

```bash
sudo named-checkconf
sudo systemctl restart bind9
sudo systemctl status bind9 --no-pager
```

Comprueba que el fichero de zona se haya transferido al esclavo:

```bash
sudo ls -l /var/cache/bind/
sudo cat /var/cache/bind/db.empresa.test
```

Si la transferencia no aparece, revisa los mensajes de BIND:

```bash
sudo journalctl -u bind9 -n 100 --no-pager
```

## Pruebas de funcionamiento

### 1. Consultar el maestro

Desde un cliente o desde el propio maestro:

```bash
dig @192.168.1.10 empresa.test SOA +noall +answer
dig @192.168.1.10 www.empresa.test A +noall +answer
```

Debe aparecer un resultado autoritativo y el registro `www.empresa.test` debe resolver a `192.168.1.30`.

### 2. Consultar el esclavo

```bash
dig @192.168.1.11 empresa.test SOA +noall +answer
dig @192.168.1.11 www.empresa.test A +noall +answer
```

El número de serie SOA debe ser igual al mostrado por el maestro. Esto confirma que el esclavo tiene una copia vigente de la zona.

### 3. Probar una transferencia AXFR autorizada

Ejecuta esta prueba desde la IP del servidor esclavo o desde un host incluido explícitamente en `allow-transfer`:

```bash
dig @192.168.1.10 empresa.test AXFR
```

La respuesta debe incluir el contenido completo de la zona. Desde un host no autorizado debe fallar, lo cual es el comportamiento deseado.

### 4. Verificar una actualización

1. En `dns-maestro`, añade un nuevo registro, por ejemplo:

```dns
ftp             IN  A       192.168.1.32
```

2. Incrementa el serial SOA, por ejemplo de `2026100801` a `2026100802`. Sin cambiar el serial, el esclavo no sabrá que existe una versión nueva de la zona.

3. Valida y recarga la zona:

```bash
sudo named-checkzone empresa.test /etc/bind/db.empresa.test
sudo rndc reload empresa.test
```

4. Comprueba desde el esclavo que la réplica se ha actualizado:

```bash
dig @192.168.1.11 empresa.test SOA +noall +answer
dig @192.168.1.11 ftp.empresa.test A +noall +answer
```

## Evidencias del laboratorio

Las siguientes capturas ya estaban subidas al repositorio y se muestran a continuación como evidencia del proceso y de las pruebas realizadas.

| Evidencia | Captura |
|---|---|
| Captura 1 | ![Captura 1](Evidencia/Captura%20de%20pantalla%202026-10-08%20194535.png) |
| Captura 2 | ![Captura 2](Evidencia/Captura%20de%20pantalla%202026-10-08%20194552.png) |
| Captura 3 | ![Captura 3](Evidencia/Captura%20de%20pantalla%202026-10-08%20194608.png) |
| Captura 4 | ![Captura 4](Evidencia/Captura%20de%20pantalla%202026-10-08%20194622.png) |
| Captura 5 | ![Captura 5](Evidencia/Captura%20de%20pantalla%202026-10-08%20194640.png) |
| Captura 6 | ![Captura 6](Evidencia/Captura%20de%20pantalla%202026-10-08%20194657.png) |
| Captura 7 | ![Captura 7](Evidencia/Captura%20de%20pantalla%202026-10-08%20194714.png) |
| Captura 8 | ![Captura 8](Evidencia/Captura%20de%20pantalla%202026-10-08%20194734.png) |
| Captura 9 | ![Captura 9](Evidencia/Captura%20de%20pantalla%202026-10-08%20194754.png) |
| Captura 10 | ![Captura 10](Evidencia/Captura%20de%20pantalla%202026-10-08%20194941.png) |
| Captura 11 | ![Captura 11](Evidencia/Captura%20de%20pantalla%202026-10-08%20195004.png) |
| Captura 12 | ![Captura 12](Evidencia/Captura%20de%20pantalla%202026-10-08%20195039.png) |
| Captura 13 | ![Captura 13](Evidencia/Captura%20de%20pantalla%202026-10-08%20195057.png) |

## Resolución de incidencias

| Síntoma | Comprobación y solución |
|---|---|
| El servicio no arranca | Ejecuta `sudo named-checkconf` y consulta `sudo journalctl -u bind9 -n 100 --no-pager`. |
| La zona no carga en el maestro | Ejecuta `sudo named-checkzone empresa.test /etc/bind/db.empresa.test`; revisa puntos finales en los FQDN, sintaxis y serial SOA. |
| El esclavo no recibe la zona | Confirma la IP del maestro en `masters`, la IP del esclavo en `allow-transfer`, la conectividad TCP/53 y los logs de BIND. |
| Maestro y esclavo muestran seriales distintos | Incrementa el serial del SOA en el maestro, valida el archivo y ejecuta `sudo rndc reload empresa.test`. |
| AXFR se rechaza | Es correcto si el origen no está incluido en `allow-transfer`; limita siempre las transferencias a los secundarios autorizados. |

## Buenas prácticas

- Restringe siempre `allow-transfer`; una transferencia AXFR abierta expone todos los registros de la zona.
- Emplea claves TSIG para autenticar transferencias de zona en entornos reales.
- Mantén una convención de serial, como `YYYYMMDDNN`, y súbelo en cada modificación.
- Añade más de un servidor secundario para mejorar la disponibilidad.
- Supervisa los registros de BIND y realiza copias de seguridad de los ficheros de configuración y de zona.

## Licencia

Material educativo elaborado para prácticas de administración de sistemas y DNS.
