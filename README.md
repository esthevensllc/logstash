# Diagnostico Filebeat + Logstash + ClickHouse para NetFlow

Este repositorio documenta los pasos para inspeccionar un flujo NetFlow que dejo de insertar datos en ClickHouse.

Flujo esperado:

```text
Router / Exportador NetFlow -> Filebeat UDP 9995 -> Logstash TCP 5044 -> ClickHouse HTTP 8123
```

## Configuracion observada

### Filebeat

Archivo principal:

```bash
/etc/filebeat/filebeat.yml
```

Configuracion base observada:

```yaml
filebeat.inputs:
- type: netflow
  host: "0.0.0.0:9995"
  protocols: [v9, ipfix]
  expiration_timeout: 30m
  queue_size: 8192
  max_message_size: 10KiB

output.logstash:
  hosts: ["172.19.216.170:5044"]

logging.level: debug
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
```

### Logstash

Pipeline observado:

```bash
/usr/share/logstash/netflow_clickhouse_9995.conf
```

Entrada:

```ruby
input {
  beats {
    port => 5044
  }
}
```

Salida hacia ClickHouse:

```ruby
output {
  clickhouse {
    http_hosts      => ["http://172.19.242.107:8123"]
    user            => "<usuario>"
    password        => "<password>"
    table           => "ookla.netflow_logs_9995_%{[@metadata][table_suffix]}"
    flush_size      => 5000
    idle_flush_time => 5
  }
}
```

Nota importante: la tabla usa `[@metadata][table_suffix]`. Si no existe ningun `filter` que cree ese campo, Logstash puede intentar insertar en una tabla incorrecta o literal.

## 1. Verificar si Filebeat esta vivo

```bash
sudo systemctl status filebeat --no-pager
sudo journalctl -u filebeat -n 100 --no-pager
sudo tail -f /var/log/filebeat/filebeat
```

Buscar errores relacionados con:

```text
netflow
template
decode
output
logstash
connection refused
i/o timeout
```

Validar configuracion:

```bash
sudo filebeat test config -c /etc/filebeat/filebeat.yml
sudo filebeat test output -c /etc/filebeat/filebeat.yml
```

## 2. Confirmar que Filebeat recibe NetFlow

Verificar que el puerto UDP 9995 este escuchando:

```bash
sudo ss -lunp | grep 9995
```

Verificar que lleguen paquetes desde el router o exportador:

```bash
sudo tcpdump -ni any udp port 9995 -c 20
```

Interpretacion:

```text
No hay paquetes en tcpdump
```

El problema esta antes de Filebeat: router, exportador NetFlow, firewall, ruta, IP destino o puerto.

```text
Hay paquetes en tcpdump, pero Filebeat no genera salida
```

Revisar logs de Filebeat, templates NetFlow, errores de decode o saturacion.

## 3. Confirmar conectividad Filebeat -> Logstash

Desde el host donde corre Filebeat:

```bash
nc -vz 172.19.216.170 5044
tcpdump -ni any host 172.19.216.170 and tcp port 5044
```

Desde el host donde corre Logstash:

```bash
sudo ss -ltnp | grep 5044
sudo tcpdump -ni any tcp port 5044 -c 20
```

Interpretacion:

```text
Filebeat recibe UDP, pero no hay trafico TCP hacia 5044
```

El problema esta en Filebeat, en el output hacia Logstash o en firewall/ruteo.

```text
Logstash no escucha en 5044
```

El pipeline no cargo, Logstash esta detenido o la configuracion cargada no es la esperada.

### Caso confirmado: `connection refused` en 5044

Si Filebeat muestra:

```text
dial tcp 172.19.216.170:5044: connect: connection refused
```

Y `nc` confirma:

```bash
nc -vz 172.19.216.170 5044
```

Con salida similar a:

```text
Ncat: Connection refused.
```

El host `172.19.216.170` es alcanzable, pero no hay ningun proceso aceptando conexiones en TCP 5044 o el puerto esta siendo rechazado activamente.

Confirmar si existe un listener:

```bash
sudo ss -ltnp | grep 5044
```

Si no devuelve nada, Logstash no levanto el input `beats` en ese puerto.

Un `tcpdump` con respuesta `Flags [R.]` tambien confirma rechazo TCP:

```bash
sudo tcpdump -ni any tcp port 5044 -c 20
```

Ejemplo:

```text
172.19.216.170.50158 > 172.19.216.170.5044: Flags [S]
172.19.216.170.5044 > 172.19.216.170.50158: Flags [R.]
```

El `S` es el intento de conexion y el `R` es el rechazo. En este caso el siguiente paso no es ClickHouse, sino levantar correctamente Logstash en 5044.

Revisar estado y logs:

```bash
sudo systemctl status logstash --no-pager
sudo journalctl -u logstash -n 200 --no-pager
sudo tail -n 200 /var/log/logstash/logstash-plain.log
```

Confirmar que el pipeline este referenciado:

```bash
cat /etc/logstash/pipelines.yml
sudo grep -R "netflow_clickhouse_9995.conf\|port => 5044" /etc/logstash /usr/share/logstash -n
```

Si `/etc/logstash/pipelines.yml` no referencia el archivo, agregar:

```yaml
- pipeline.id: netflow_clickhouse_9995
  path.config: "/usr/share/logstash/netflow_clickhouse_9995.conf"
```

Validar antes de reiniciar:

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash \
  --config.test_and_exit \
  -f /usr/share/logstash/netflow_clickhouse_9995.conf
```

Si la validacion termina correctamente, reiniciar y volver a verificar:

```bash
sudo systemctl restart logstash
sleep 10
sudo systemctl status logstash --no-pager
sudo ss -ltnp | grep 5044
```

Resultado esperado:

```text
LISTEN ... :5044 ... java
```

Finalmente, probar otra vez desde Filebeat:

```bash
sudo filebeat test output -c /etc/filebeat/filebeat.yml
```

Resultado esperado:

```text
dial up... OK
```

## 4. Confirmar que Logstash esta usando el pipeline correcto

Revisar como arranca Logstash:

```bash
sudo systemctl cat logstash
ps -ef | grep '[l]ogstash'
cat /etc/logstash/pipelines.yml
```

Validar el archivo de pipeline:

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash \
  --config.test_and_exit \
  -f /usr/share/logstash/netflow_clickhouse_9995.conf
```

Revisar logs:

```bash
sudo systemctl status logstash --no-pager
sudo journalctl -u logstash -n 200 --no-pager
sudo tail -f /var/log/logstash/logstash-plain.log
```

Filtrar errores relevantes:

```bash
sudo grep -Ei "clickhouse|error|exception|failed|retry|timeout|table|401|403|404" /var/log/logstash/logstash-plain.log
```

## 5. Revisar metricas internas de Logstash

Logstash expone metricas locales por defecto en el puerto 9600:

```bash
curl -s http://127.0.0.1:9600/_node/stats/pipelines?pretty
```

Revisar estos contadores dentro del pipeline:

```text
events.in
events.filtered
events.out
```

Interpretacion:

```text
events.in no sube
```

Logstash no recibe eventos desde Filebeat.

```text
events.in sube pero events.out no sube
```

Hay un problema en filtros o salida.

```text
events.in y events.out suben
```

Logstash procesa los eventos; revisar ClickHouse, tabla destino o errores del plugin.

## 6. Probar ClickHouse desde el host de Logstash

Probar ping HTTP:

```bash
curl -sS -u '<usuario>:<password>' http://172.19.242.107:8123/ping
```

La respuesta esperada es:

```text
Ok.
```

Probar una consulta simple:

```bash
curl -sS -u '<usuario>:<password>' \
  'http://172.19.242.107:8123/?query=SELECT%201'
```

Listar bases y tablas:

```bash
curl -sS -u '<usuario>:<password>' \
  'http://172.19.242.107:8123/?query=SHOW%20DATABASES'

curl -sS -u '<usuario>:<password>' \
  'http://172.19.242.107:8123/?query=SHOW%20TABLES%20FROM%20ookla'
```

Revisar tablas NetFlow:

```bash
curl -sS -u '<usuario>:<password>' \
  'http://172.19.242.107:8123/?query=SHOW%20TABLES%20FROM%20ookla%20LIKE%20%27netflow_logs_9995_%25%27'
```

## 7. Revisar el sufijo de tabla `table_suffix`

La configuracion usa:

```ruby
table => "ookla.netflow_logs_9995_%{[@metadata][table_suffix]}"
```

Buscar si el campo se esta creando:

```bash
sudo grep -R "table_suffix" /etc/logstash /usr/share/logstash -n
```

Si no existe ningun `filter` que defina `[@metadata][table_suffix]`, se debe agregar uno o cambiar el nombre de la tabla por una tabla fija existente.

Ejemplo de debug temporal:

```ruby
output {
  stdout {
    codec => rubydebug {
      metadata => true
    }
  }

  clickhouse {
    http_hosts      => ["http://172.19.242.107:8123"]
    user            => "<usuario>"
    password        => "<password>"
    table           => "ookla.netflow_logs_9995_%{[@metadata][table_suffix]}"
    flush_size      => 5000
    idle_flush_time => 5
  }
}
```

Luego reiniciar Logstash y revisar si el evento contiene:

```text
[@metadata][table_suffix]
```

## 8. Confirmar que la tabla destino existe

Si el sufijo es diario, mensual o depende de fecha, validar la tabla esperada:

```bash
date +%Y%m%d
date +%Y%m
```

Luego consultar ClickHouse:

```bash
curl -sS -u '<usuario>:<password>' \
  'http://172.19.242.107:8123/?query=SHOW%20TABLES%20FROM%20ookla%20LIKE%20%27netflow_logs_9995_%25%27'
```

Si cambio el dia o mes y la tabla nueva no existe, Logstash dejara de insertar.

## 9. Diagnostico rapido por resultado

```text
No hay paquetes en tcpdump UDP 9995
```

Problema antes de Filebeat: router, exportador, firewall, ruta o puerto.

```text
Hay UDP 9995 pero Filebeat no conecta a 5044
```

Problema en Filebeat, output.logstash o conectividad hacia Logstash.

```text
dial tcp 172.19.216.170:5044: connect: connection refused
```

Logstash no esta escuchando en 5044 o el puerto esta siendo rechazado. Revisar `systemctl status logstash`, `ss -ltnp | grep 5044`, logs y `pipelines.yml`.

```text
Logstash no escucha en 5044
```

Pipeline no cargado, servicio detenido o archivo no incluido en `pipelines.yml`.

```text
events.in no sube en Logstash
```

Logstash no recibe eventos desde Filebeat.

```text
events.in sube, pero events.out no sube
```

Problema en filtros, output o plugin de ClickHouse.

```text
events.out sube, pero ClickHouse no muestra data
```

Revisar tabla destino, sufijo, credenciales, schema, particiones o zona horaria.

```text
Errores 400 de ClickHouse
```

Formato incompatible, columna faltante, tipo incorrecto o tabla inexistente.

```text
Errores 401 o 403
```

Usuario/password o permisos incorrectos.

```text
Timeout o connection refused
```

ClickHouse caido, firewall, red o puerto 8123 inaccesible.

## 10. Comandos minimos para ubicar el tramo roto

Ejecutar estos comandos como primera revision:

```bash
sudo tcpdump -ni any udp port 9995 -c 20
sudo filebeat test output -c /etc/filebeat/filebeat.yml
curl -s http://127.0.0.1:9600/_node/stats/pipelines?pretty
sudo grep -Ei "clickhouse|error|exception|failed|table|timeout" /var/log/logstash/logstash-plain.log
```

Con esas salidas se puede saber si el corte esta en:

```text
NetFlow -> Filebeat
Filebeat -> Logstash
Logstash pipeline
Logstash -> ClickHouse
ClickHouse tabla/schema
```
