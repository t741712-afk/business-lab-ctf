# CTF Laboratories Corporation TAI: notas personales

Recordatorio de lo que monta este proyecto, cómo desplegarlo y qué queda pendiente.
Es un proyecto independiente de `business-lab`.

---

## Qué es

Una sola plantilla de CloudFormation (`entorno-ctf.yaml`) que levanta una VPC `10.0.0.0/16` con dos redes:

- **DMZ `10.0.0.0/24`** (pública): frontales y monitor.
- **MZ `10.0.1.0/24`** (privada, salida a Internet por NAT): controlador de dominio, conector ZTSA y los workers de k3s.

Las direcciones están fijas en la plantilla y no aparecen como parámetros.

---

## Máquinas

| Red | Máquina | IP | Tipo | Acceso |
|---|---|---|---|---|
| DMZ | `servicegateway` | 10.0.0.75 | c5.2xlarge | IP pública dinámica, SSH como `admin` |
| DMZ | `monitor` | 10.0.0.40 | t3.medium | Grafana HTTPS 443, Prometheus 9090 (admin) |
| DMZ | `web-apache-php` | 10.0.0.33 | t3.small | Pública, puerto 80 |
| DMZ | `web-iis` | 10.0.0.35 | t3.medium | Pública, puerto 80 |
| DMZ | `k3s-master` | 10.0.0.50 | t3.medium | IP pública dinámica: web `:30080`, pgAdmin `:30081`, SSH y API `6443` (admin) |
| MZ | `dc1` | 10.0.1.10 | t3.medium | Sin IP pública: RDP desde una máquina de la DMZ |
| MZ | `ztsa-private-connector` | 10.0.1.16 | c5.2xlarge | Sin IP pública: SSH desde una máquina de la DMZ |
| MZ | `k3s-worker-1` | 10.0.1.51 | t3.medium | Sin IP pública: SSH desde una máquina de la DMZ |
| MZ | `k3s-worker-2` | 10.0.1.52 | t3.medium | Sin IP pública: SSH desde una máquina de la DMZ |

Las máquinas de la MZ no tienen IP pública: para entrar en ellas hay que saltar desde una máquina de la DMZ con IP pública (por ejemplo `k3s-master` o `servicegateway`) y conectar a su IP privada. Son 9 instancias en total. Contando sin los workers, son 7 máquinas, que es el número que querías.

---

## Desplegar

Parámetros que **hay que pasar** (no tienen valor por defecto):

- `K3sToken`: token compartido de k3s, lo inventas.
- `GroqApiKey`, `TrendAiGuardApiKey`, `FileSecurityApiKey`: claves de IA.
- `VisionOneBootstrapToken`: bootstrap token de Container Security.

Parámetros recomendados:

- `AllowedAdminCidr`: tu IP en `/32`. Por defecto es `0.0.0.0/0` y deja SSH, RDP y la API abiertos a Internet.
- `TrendTenantId` y `TrendToken`: si quieres el agente de Trend en el DC, IIS y Apache.

```bash
MIIP="$(curl -s https://checkip.amazonaws.com)/32"

aws cloudformation create-stack \
  --stack-name business-lab-ctf \
  --template-body file://entorno-ctf.yaml \
  --parameters \
    ParameterKey=AllowedAdminCidr,ParameterValue="$MIIP" \
    ParameterKey=K3sToken,ParameterValue='<token>' \
    ParameterKey=GroqApiKey,ParameterValue='<groq>' \
    ParameterKey=TrendAiGuardApiKey,ParameterValue='<ai-guard>' \
    ParameterKey=FileSecurityApiKey,ParameterValue='<file-security>' \
    ParameterKey=VisionOneBootstrapToken,ParameterValue='<bootstrap>' \
    ParameterKey=TrendTenantId,ParameterValue='<tenant>' \
    ParameterKey=TrendToken,ParameterValue='<token-trend>'

aws cloudformation wait stack-create-complete --stack-name business-lab-ctf
```

Notas:
- Se usa el key pair `corp-lab` (el valor por defecto). Debe existir en us-east-1.
- El stack tarda bastante: el DC y el clúster k3s tardan decenas de minutos en quedar listos.
- Las salidas del stack dan las URLs y las IPs públicas actuales.

---

## Dónde está cada cosa

- `entorno-ctf.yaml`: la plantilla.
- `scripts/`: los scripts que descargan las máquinas al arrancar (`dc1.ps1`, `monitor.sh`, `web-apache-php.sh`, `web-iis.ps1`, `trend-agent.sh`, `trend-agent.ps1`).
- `apache/`: las páginas PHP del frontal Apache (`index.php`, `info.php`, `diagnostic.php`).
- El clúster k3s clona el repositorio de la web, `k8-web-politica`, que está fuera de este proyecto.
- Las contraseñas de los fallos están en los propios scripts. No las pongo aquí.

Repositorio: `github.com/t741712-afk/business-lab-ctf` (público).

---

## Cosas a tener en cuenta

- **IPs públicas dinámicas.** Solo cambian si paras una instancia y la vuelves a arrancar, o al redesplegar. Si paras el master, su certificado de la API tiene la IP antigua, y habrá que reinstalar k3s o usar la IP privada desde dentro de la VPC.
- **Nat Gateway.** La única IP elástica es la del NAT, que es necesaria para que la MZ salga a Internet.
- **Monitor.** Sus objetivos son IPs del proyecto anterior, así que verás alertas de servicios caídos. Es lo esperado.
- **Repositorio público.** Los scripts, las contraseñas y las soluciones son visibles para cualquiera.
- **Dependencias externas**: el repositorio `k8-web-politica`, el script `vulnad.ps1` de safebuffer y los paquetes de Trend (`files.trendmicro.com`). Si cambian, hay que revisar las URLs.

---

## Pendiente

- [ ] Verificar el AMI del conector ZTSA (`ami-0a520ca8b8a290243`). Puede ser de Marketplace, lo que incumpliría PX.
- [ ] Decidir si `AllowedAdminCidr` pasa a ser un parámetro obligatorio sin valor por defecto.
- [ ] Decidir si el CTF se queda público o se pasa a token de solo lectura / bucket antes del juego.
- [ ] Revisar las alertas del monitor y si conviene cambiar sus objetivos.
- [ ] Cambiar el token de k3s, las claves de IA y el bootstrap de Vision One después del juego.
- [ ] Verificar el Service Gateway tras el último cambio (cifrado e IMDSv2 añadidos).

---

## Cambios recientes

- Eliminadas las IPs fijas de los parámetros y las salidas que mostraban IPs internas.
- Eliminada la IP elástica del master; ahora su IP pública es dinámica.
- Master k3s: construye sus imágenes, crea el secreto de IA e instala Vision One Container Security.
- `web-apache-php.sh` descarga también `diagnostic.php`.
- Descargas apuntan a `business-lab-ctf`, no a `business-lab`.
