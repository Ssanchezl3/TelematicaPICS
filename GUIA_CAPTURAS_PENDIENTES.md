# Guía de capturas pendientes — Proyecto N2 (WordPress en AWS)

Esta guía te lleva paso a paso por lo que falta para cerrar el informe al 100 %.
Cada bloque dice: **dónde ir en AWS**, **qué comando ejecutar**, **qué foto tomar** y
**con qué nombre subirla al repo**. Sube cada PNG al repo `TelematicaPICS` con el
nombre indicado (columna "archivo") y avísame para adjuntarla al informe.

Todas las instancias se administran con **Session Manager**:
Consola AWS → **EC2** → *Instances* → seleccionar la instancia → botón **Connect**
→ pestaña **Session Manager** → **Connect**. Se abre una terminal en el navegador.

Referencia de IPs (de tu informe):

| Instancia        | IP privada | IP pública       |
|------------------|------------|------------------|
| nat-a            | 10.0.1.98  | autoasignada     |
| nat-b            | 10.0.3.122 | FALTA registrar  |
| balanceador-a    | 10.0.1.203 | 44.209.50.188    |
| balanceador-b    | 10.0.3.239 | 54.157.180.145   |
| wordpress-a      | 10.0.2.85  | —                |
| wordpress-b      | 10.0.4.25  | —                |
| db               | 10.0.2.52  | —                |
| bind9-primario   | FALTA      | 34.231.99.252    |
| bind9-secundario | FALTA      | 98.82.66.161     |

---

## PRIORIDAD 1 — Captura y análisis de tráfico (requisito formal del enunciado)

Es lo único exigido explícitamente que aún no tiene ninguna evidencia.
Se hace con `tcpdump` en terminal (no necesitas Wireshark). Vamos a capturar los
4 protocolos que pide el enunciado: DNS, TCP, HTTP y la traducción NAT.

### 1A. Traducción NAT vista en la propia NAT (lo más importante)

> La idea: capturar en nat-a el MISMO paquete en las dos interfaces/lados para
> ver cómo cambia la IP de origen (10.0.2.85 → 10.0.1.98).

**Dónde:** dos sesiones de Session Manager abiertas al mismo tiempo:
una en **wordpress-a** y otra en **nat-a**.

**En nat-a**, deja corriendo la captura (muestra el tráfico hacia Internet):
```bash
sudo tcpdump -n -i ens5 'host 10.0.2.85 and (tcp port 80 or tcp port 443 or icmp)' -c 20
```

**En wordpress-a**, genera tráfico:
```bash
curl -s https://checkip.amazonaws.com; echo
ping -c 3 8.8.8.8
```

**Foto:** la terminal de **nat-a** mostrando las líneas de tcpdump donde se ve
`10.0.2.85 > ...` saliendo. Si además corres el tcpdump sin filtro de puerto,
verás el retorno hacia 10.0.1.98.
→ **archivo: `trafico-nat-tcpdump.png`**

### 1B. Resolución DNS capturada

**Dónde:** una sesión en **bind9-primario** (o en wordpress-a, cualquiera con red).

Deja la captura corriendo en bind9-primario:
```bash
sudo tcpdump -n -i ens5 'port 53' -c 10
```
En otra sesión (wordpress-a o tu PC), lanza una consulta contra ese servidor:
```bash
dig @34.231.99.252 www.whatever.casa A
```
**Foto:** la terminal con tcpdump mostrando el paquete UDP/53 de consulta y respuesta.
→ **archivo: `trafico-dns-tcpdump.png`**

### 1C. Establecimiento TCP + intercambio HTTP

**Dónde:** sesión en **balanceador-a**.

Deja corriendo:
```bash
sudo tcpdump -n -i ens5 'tcp port 80' -c 25
```
Desde tu PC (o wordpress-a) genera una petición:
```bash
curl -s http://44.209.50.188 > /dev/null
```
**Foto:** la terminal con tcpdump mostrando el `[S]`, `[S.]`, `[.]` (el 3-way
handshake) y las líneas con `Flags [P.]` del HTTP.
→ **archivo: `trafico-tcp-http-tcpdump.png`**

> Opcional recomendado: si prefieres Wireshark, guarda un .pcap con
> `sudo tcpdump -n -i ens5 -w captura.pcap` y ábrelo luego; una captura de la
> ventana de Wireshark con la columna de protocolos también sirve.

---

## PRIORIDAD 2 — Source/Destination Check de ambas NAT (requisito del enunciado)

**Dónde:** Consola AWS → **EC2** → *Instances* → clic en **nat-a**.
→ menú **Actions** → **Networking** → **Change source/destination check**.
Se abre un cuadro donde debe verse **"Stop" marcado** (o el texto indicando que
el check está *disabled*).

**Foto 1:** ese cuadro de diálogo de **nat-a** mostrando el check desactivado.
→ **archivo: `srcdest-nat-a.png`**

Repite exactamente lo mismo con **nat-b**.
**Foto 2:** el cuadro de **nat-b** con el check desactivado.
→ **archivo: `srcdest-nat-b.png`**

> Alternativa: en el detalle de la instancia, pestaña **Networking**, hay un campo
> "Source/destination check: Disabled". Una captura de esa fila también vale.

---

## PRIORIDAD 3 — conntrack de nat-b (completa la prueba N2)

Tu informe solo tiene la asociación en nat-a. Hay que mostrar la de nat-b con
tráfico originado en WordPress B.

**Dónde:** dos sesiones: **wordpress-b** y **nat-b**.

**En wordpress-b**, genera conexiones:
```bash
curl -s https://checkip.amazonaws.com; echo
curl -s https://aws.amazon.com > /dev/null
```
**En nat-b**, consulta conntrack:
```bash
sudo conntrack -L | grep "10.0.4"
```
**Foto:** la terminal de **nat-b** mostrando la tupla con origen 10.0.4.25 y la
dirección 10.0.3.122 (la de nat-b) en la respuesta.
→ **archivo: `nat-n2-conntrack-b.png`**

---

## PRIORIDAD 4 — B1 y B2 por balanceador-b (el enunciado exige ambos)

Ya tienes B1/B2 por balanceador-a. Repite por balanceador-b (IP 54.157.180.145).

**Dónde:** tu PC, o cualquier instancia con salida (wordpress-a sirve).

**B1 (reparto) — con los dos WordPress encendidos:**
```bash
for i in $(seq 1 10); do
  curl -s http://54.157.180.145 | grep -o "Servidor WordPress:[^<]*"
done
```
**Foto:** la salida con las 10 líneas alternando 10.0.2.85 y 10.0.4.25.
→ **archivo: `balanceo-b1-balanceador-b.png`**

**B2 (caída) — detén wordpress-b y repite:**
1. Consola EC2 → selecciona **wordpress-b** → **Instance state** → **Stop instance**
   (o dentro de wordpress-b: `sudo systemctl stop apache2`).
2. Vuelve a correr el mismo `for` de arriba contra 54.157.180.145.

**Foto:** las 10 respuestas, todas de 10.0.2.85 (WordPress A).
→ **archivo: `balanceo-b2-balanceador-b.png`**
3. **Reinicia wordpress-b** al terminar (Start instance / `systemctl start apache2`).

---

## PRIORIDAD 5 — Resolución DNS desde red externa + traza (completa D1)

Las consultas del informe se hicieron dentro de la VPC. El enunciado pide
resolución desde fuera y verificar la delegación pública.

**Dónde:** **tu propio computador** (NO una instancia AWS), fuera de la VPC.
En Windows usa `nslookup`; en Mac/Linux usa `dig`.

```bash
# Linux / Mac
dig +trace www.whatever.casa
dig @8.8.8.8 www.whatever.casa A
dig NS whatever.casa
```
```text
:: Windows (PowerShell/CMD)
nslookup -type=NS whatever.casa 8.8.8.8
nslookup www.whatever.casa 8.8.8.8
```
**Foto:** la salida mostrando que un resolvedor público (8.8.8.8) devuelve tus NS
y las dos IP de www. El `+trace` muestra la cadena desde la raíz hasta tu zona.
→ **archivo: `dns-externo-trace.png`**

---

## PRIORIDAD 6 — Contenido corporativo de XYZ S.A.

El enunciado pide publicar info de la empresa y sus productos. Hoy se ve
"Hello world". Entra al WordPress (`http://www.whatever.casa/wp-admin`), crea
una página de inicio con texto de XYZ S.A. y una página/entradas de productos.

**Foto:** la home del sitio ya con contenido de la empresa (y el pie que muestra
la IP del servidor que respondió).
→ **archivo: `sitio-xyz-contenido.png`**

> Debes hacerlo en los DOS WordPress o sincronizar wp-content/BD, porque la BD es
> compartida pero los archivos (imágenes/temas) no se replican solos.

---

## PRIORIDAD 7 — Datos faltantes del inventario (sin fotos, solo dato)

En la consola EC2, mira el detalle de estas instancias y pásame los valores:
- **nat-b** → IP pública (campo "Public IPv4 address").
- **bind9-primario** → IP privada ("Private IPv4 address").
- **bind9-secundario** → IP privada.

No hace falta foto; basta con que me escribas los 3 valores y yo completo la Tabla 2.

---

## Resumen de archivos a subir al repo

| Prioridad | Archivo a subir                      | Qué muestra                          |
|-----------|--------------------------------------|--------------------------------------|
| 1A        | `trafico-nat-tcpdump.png`            | Traducción NAT en tcpdump            |
| 1B        | `trafico-dns-tcpdump.png`            | Consulta/respuesta DNS en tcpdump    |
| 1C        | `trafico-tcp-http-tcpdump.png`       | Handshake TCP + HTTP en tcpdump      |
| 2         | `srcdest-nat-a.png`                  | Source/Dest check desactivado nat-a  |
| 2         | `srcdest-nat-b.png`                  | Source/Dest check desactivado nat-b  |
| 3         | `nat-n2-conntrack-b.png`             | conntrack en nat-b                   |
| 4         | `balanceo-b1-balanceador-b.png`      | Reparto por balanceador-b            |
| 4         | `balanceo-b2-balanceador-b.png`      | Falla de WP-B por balanceador-b      |
| 5         | `dns-externo-trace.png`              | Resolución externa + delegación      |
| 6         | `sitio-xyz-contenido.png`            | Sitio con contenido corporativo      |
| 7         | (sin foto: 3 IPs por texto)          | Completar inventario Tabla 2         |

Súbelas al repo con esos nombres exactos. Puedes subir de a una o en lote; cada vez
que subas, dime cuáles y las voy adjuntando al informe con su figura y pie de foto.
