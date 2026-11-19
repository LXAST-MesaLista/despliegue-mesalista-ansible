# Manual: desplegar MesaLista con Ansible desde dentro de la máquina de Azure

Este manual explica cómo instalar o actualizar el **Servidor**
([LXPT-UIB/Servidor](https://github.com/LXPT-UIB/Servidor)) y la **WEB**
([LXPT-UIB/WEB](https://github.com/LXPT-UIB/WEB)) ejecutando Ansible **en la
propia máquina Ubuntu de Azure**, una vez que ya has entrado en ella por SSH.

Ansible no necesita aquí ninguna conexión más: trabaja sobre la máquina en la
que se ejecuta (conexión `local`). No hace falta WSL ni nada en tu ordenador,
aparte de la terminal con la que entras por SSH.

```
Tu ordenador ──SSH──► Máquina de Azure (Ubuntu 24.04)
                        └── ansible-playbook -i inventario-local.yml desplegar.yml
                              ├── git clone Servidor → install.sh --no-interactivo
                              ├── PayPal, correo de Azure y Firebase
                              ├── git clone WEB      → instalar.sh --no-interactivo
                              └── mesalista estado
```

---

## 1. Antes de empezar

| Qué | Cómo comprobarlo |
|---|---|
| La máquina es **Ubuntu Server 24.04 LTS** | `lsb_release -d` → `Ubuntu 24.04…` |
| Estás dentro por SSH, con tu usuario (p. ej. `azureuser`) | El indicador dice `azureuser@…:~$` |
| Tu usuario puede usar `sudo` **sin contraseña** (así vienen las máquinas de Azure creadas con clave SSH) | `sudo -n true && echo sí` → `sí` |
| La máquina tiene salida a internet (GitHub, paquetes, Let's Encrypt) | `curl -sI https://github.com \| head -1` → `HTTP/2 200` |
| En Azure, la IP pública es **estática** y tiene **etiqueta DNS** | Portal de Azure > Dirección IP pública > Configuración |
| En Azure, el **grupo de seguridad de red** deja entrar **80 y 443** desde cualquier sitio | Portal > máquina > Redes > Reglas de puerto de entrada |

Si `sudo -n true` pide contraseña, no pasa nada: añade `-K` a las órdenes de
`ansible-playbook` y Ansible te la pedirá al empezar.

---

## 2. Instalar Ansible y git en la máquina

```bash
sudo apt update
sudo apt install -y ansible git
ansible --version          # ansible [core 2.16…] o posterior
```

`ansible` del repositorio de Ubuntu 24.04 trae `ansible-core` 2.16, que sirve.

---

## 3. Bajar el proyecto de despliegue

En tu carpeta personal (no en `/srv/mesalista`, que es donde Ansible deja la
copia de los repositorios de MesaLista):

```bash
cd ~
git clone https://github.com/LXPT-UIB/despliegue-mesalista-ansible.git
cd despliegue-mesalista-ansible
ls
```

Verás:

| Archivo | Para qué |
|---|---|
| `inventario-local.yml` | La máquina, con conexión `local`: **el que se usa en este manual** |
| `inventario.yml` | Para desplegar desde otro ordenador por SSH (no se usa aquí) |
| `group_vars/all/main.yml` | Parámetros: dominio, correos, repositorios, ramas… |
| `group_vars/all/vault.yml.example` | Modelo de los secretos |
| `desplegar.yml` / `comprobar.yml` | Desplegar todo / solo comprobar |
| `roles/` | Los pasos: `base`, `servidor`, `web` y `comprobacion` |
| `files/` | Aquí va, si la quieres, la cuenta de servicio de Firebase |

Todas las órdenes de este manual se escriben **dentro de esta carpeta**.

---

## 4. Configurar

### 4.1 El inventario: no hay que tocar nada

`inventario-local.yml` ya dice que la máquina es la propia:

```yaml
all:
  hosts:
    mesalista:
      ansible_connection: local
      ansible_python_interpreter: /usr/bin/python3
```

### 4.2 Los parámetros: `group_vars/all/main.yml`

```bash
nano group_vars/all/main.yml
```

Revisa sobre todo estos (guarda con **Ctrl+O**, Enter, y sal con **Ctrl+X**):

| Variable | Qué poner |
|---|---|
| `mesalista_dominio` | El dominio de la máquina (la etiqueta DNS de Azure), sin `https://` |
| `mesalista_correo_letsencrypt` | **Tu correo** para los avisos del certificado. Con `cambiar@ejemplo.com` Ansible no sigue |
| `mesalista_correo_admin` | El correo de la cuenta de administrador de MesaLista |
| `mesalista_rama_servidor`, `mesalista_rama_web` | `main`, salvo que quieras probar otra rama |
| `mesalista_correo_remitente` | Solo si configuras el correo de Azure: `DoNotReply@….azurecomm.net` |
| `mesalista_instalar_web` | `true` para instalar también la web |

Para saber el dominio desde la propia máquina (si ya estaba MesaLista instalado):

```bash
sudo grep SERVER_DOMAIN /etc/mesalista/install.env
```

### 4.3 Los secretos: `group_vars/all/vault.yml`

Los secretos van en un archivo **cifrado** con Ansible Vault. Se crea a partir del modelo:

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
chmod 600 group_vars/all/vault.yml
nano group_vars/all/vault.yml
```

| Secreto | Qué es | ¿Obligatorio? |
|---|---|---|
| `vault_admin_password` | Contraseña de la cuenta de administrador (10 caracteres o más, que no sea de las comunes). Solo se usa la primera vez | Sí |
| `vault_pin_cerrar_local` | PIN de «Cerrar local», de 4 a 8 cifras. Solo se usa la primera vez | Sí |
| `vault_paypal_client_id`, `vault_paypal_secret` | Claves de la app de PayPal sandbox | No: vacío = sin señal |
| `vault_correo_conexion` | Cadena de conexión de Azure Communication Services | No: vacío = sin correo |

Cuando esté relleno, **cífralo**:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

Pide una **contraseña del vault** dos veces. Apúntala en un sitio seguro: sin
ella no se puede desplegar ni ver los secretos. El archivo queda así:

```
$ANSIBLE_VAULT;1.1;AES256
6339613861…
```

| Quieres… | Orden |
|---|---|
| Ver los secretos | `ansible-vault view group_vars/all/vault.yml` |
| Cambiarlos | `ansible-vault edit group_vars/all/vault.yml` |
| Cambiar la contraseña del vault | `ansible-vault rekey group_vars/all/vault.yml` |

`vault.yml` no se sube nunca a Git (`.gitignore`), ni siquiera cifrado.

### 4.4 Firebase (opcional, para los avisos push)

La cuenta de servicio de Firebase tiene que estar **en la máquina**, en
`files/firebase-cuenta-servicio.json`. Desde **tu ordenador** (no desde la
máquina), en PowerShell o cmd:

```powershell
scp -i $HOME\.ssh\mesalista.pem $HOME\Downloads\mesalista-57287-firebase-adminsdk-xxxxx.json azureuser@mesalista.spaincentral.cloudapp.azure.com:~/despliegue-mesalista-ansible/files/firebase-cuenta-servicio.json
```

Y en la máquina:

```bash
chmod 600 files/firebase-cuenta-servicio.json
```

Ansible la instala con `mesalista push`. Cuando haya terminado y funcione, la
puedes borrar (`rm files/firebase-cuenta-servicio.json`): `config.json` ya
guarda lo necesario. Si la borras, Ansible no vuelve a tocar Firebase.

No sirve `google-services.json`: ese archivo es de la app Android.

---

## 5. Comprobar antes de desplegar

```bash
# 1. Ansible llega a la máquina (aquí, a sí misma) y puede usar sudo
ansible -i inventario-local.yml mesalista -m ping -b
```

Respuesta esperada:

```
mesalista | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

```bash
# 2. Los archivos están bien escritos
ansible-playbook -i inventario-local.yml desplegar.yml --syntax-check --ask-vault-pass
```

Respuesta esperada: `playbook: desplegar.yml`.

```bash
# 3. (Opcional) Qué haría, sin cambiar nada
ansible-playbook -i inventario-local.yml desplegar.yml --check --ask-vault-pass
```

En `--check` no se ejecutan los instaladores: sirve para ver que los secretos
y los parámetros pasan las comprobaciones iniciales. En una máquina donde
MesaLista aún no está instalado, termina con «mesalista estado ha encontrado
cosas que revisar»: es normal, porque todavía no hay nada que comprobar.

---

## 6. Desplegar

```bash
ansible-playbook -i inventario-local.yml desplegar.yml --ask-vault-pass
```

Escribe la contraseña del vault. La primera vez tarda unos **10 minutos**
(Node.js, PostgreSQL, nginx, certificado…). Las siguientes, 2 o 3.

### 6.1 Qué vas a ver

Cada tarea sale con su resultado:

| Marca | Significa |
|---|---|
| `ok` | Ya estaba bien, no ha hecho falta cambiar nada |
| `changed` | Ha cambiado algo (p. ej. ha instalado o actualizado) |
| `skipping` | No tocaba (p. ej. PayPal sin claves, o las mismas de antes) |
| `failed` | Ha fallado: Ansible se para y explica por qué |
| `censored` | Una tarea con secretos: Ansible oculta sus datos a propósito |

Los pasos, en orden:

1. **base**: comprueba los secretos y los parámetros, instala `git` y crea `/srv/mesalista`.
2. **servidor**: baja el repositorio del Servidor en `/srv/mesalista/Servidor` y
   ejecuta `install.sh --no-interactivo`. Se ven sus últimas líneas
   («Salida de install.sh»).
3. **integraciones**: PayPal, correo de Azure y Firebase, solo si hay datos y han cambiado.
4. **web**: baja la WEB en `/srv/mesalista/WEB` y ejecuta `instalar.sh --no-interactivo`.
5. **comprobacion**: `mesalista estado` y una petición a `https://<dominio>/api/v1/config`.

### 6.2 El resumen final

```
PLAY RECAP *********************************************************************
mesalista : ok=31  changed=6  unreachable=0  failed=0  skipped=8  rescued=0  ignored=0
```

**`failed=0`** quiere decir que todo ha ido bien. Los números de `ok`,
`changed` y `skipped` cambian según lo que haya que hacer.

---

## 7. Comprobar el resultado

En la máquina:

```bash
sudo mesalista estado
```

Todo debería salir con ✔ (o – en lo opcional que no hayas configurado).

En el navegador de tu ordenador:

| Dirección | Qué debe salir |
|---|---|
| `https://<dominio>/` | La portada de MesaLista, con candado (certificado de Let's Encrypt) |
| `https://<dominio>/panel/entrar` | Entrar con `mesalista_correo_admin` y `vault_admin_password` |
| `https://<dominio>/api/v1/config` | Un JSON con `min_party`, `max_party`… |

Solo comprobar, sin cambiar nada:

```bash
ansible-playbook -i inventario-local.yml comprobar.yml --ask-vault-pass
```

---

## 8. El día a día

| Quieres… | Qué hacer |
|---|---|
| **Actualizar** MesaLista tras cambios en GitHub | `ansible-playbook -i inventario-local.yml desplegar.yml --ask-vault-pass` (baja la última versión de cada rama) |
| Actualizar el propio proyecto de despliegue | `git pull` en `~/despliegue-mesalista-ansible` |
| Solo el servidor | `… desplegar.yml --ask-vault-pass --tags servidor` |
| Solo la web | `… desplegar.yml --ask-vault-pass --tags web` |
| Cambiar las claves de PayPal o del correo | `ansible-vault edit group_vars/all/vault.yml` y `… --tags integraciones` |
| Cambiar de dominio | Nueva etiqueta DNS en Azure, `mesalista_dominio` en `main.yml` y desplegar todo |
| Desplegar otra rama (p. ej. para probar) | `… -e mesalista_rama_web=mi-rama --tags web` |
| **Empezar de cero** (borra todas las reservas y cuentas) | `… desplegar.yml --ask-vault-pass -e mesalista_borrar_base_de_datos=true` |

(`…` = `ansible-playbook -i inventario-local.yml`.)

La contraseña del administrador y el PIN del vault **solo se usan la primera
vez**, al crear la cuenta y el PIN. Para cambiarlos después: el panel
(*Mi cuenta* y *Administración > Parámetros*), o `sudo mesalista pin`.

---

## 9. Seguridad

- **No escribas secretos en la línea de órdenes** (`-e vault_admin_password=…`):
  quedan en el historial de `bash`. Van siempre en `vault.yml`.
- `vault.yml` **cifrado** y con permisos `600`. No lo copies a otras máquinas sin cifrar.
- La **contraseña del vault** no se guarda en la máquina. Si quieres un archivo
  de contraseña (`--vault-password-file`), que sea con permisos `600` y fuera
  de la carpeta del proyecto. Lo más seguro es escribirla cada vez.
- La cuenta de servicio de Firebase da acceso de administrador al proyecto de
  Firebase: bórrala de `files/` cuando ya esté instalada.
- Ansible oculta los secretos en pantalla (`no_log`) y los instaladores los
  reciben por variables de entorno, no en la línea de órdenes.

---

## 10. Si algo falla

| Mensaje | Qué pasa | Qué hacer |
|---|---|---|
| `Missing sudo password` | Tu usuario necesita contraseña para `sudo` | Añade `-K` (Ansible te la pide) |
| `Attempting to decrypt but no vault secrets found` | Falta la contraseña del vault | Añade `--ask-vault-pass` |
| `Decryption failed` | Contraseña del vault equivocada | Vuelve a escribirla con cuidado |
| `Faltan los secretos` | No existe `group_vars/all/vault.yml` | Apartado 4.3 |
| `vault_admin_password necesita al menos 10 caracteres…` | Contraseña o PIN con mal formato | `ansible-vault edit …` |
| `Revisa group_vars/all/main.yml` | Dominio mal escrito o el correo de ejemplo | Apartado 4.2 |
| `install.sh ha terminado con error: … Falta SERVER_DOMAIN …` | Falta un dato del modo sin preguntas | Revisa `main.yml` y `vault.yml` |
| `install.sh ha terminado con error` (otro) | Fallo al instalar | Las líneas que muestra Ansible; `sudo mesalista logs` |
| Aviso: «No se ha podido obtener el certificado de Let's Encrypt» | El dominio no apunta a la IP, o el puerto 80 está cerrado en Azure | Revisa la etiqueta DNS y las reglas de entrada; vuelve a desplegar |
| `PayPal: PayPal sandbox no acepta esas claves` | Client ID o Secret mal copiados, o de *Live* en vez de *Sandbox* | `ansible-vault edit …` y `--tags integraciones` |
| `Firebase: … no es el archivo JSON de una cuenta de servicio` | Has puesto `google-services.json` | Apartado 4.4 |
| Aviso: «No se llega a https://…/api/v1/config» | La máquina no se alcanza a sí misma por su dominio, o nginx no está bien | Abre la dirección desde tu navegador; si allí funciona, ignóralo |
| `mesalista estado ha encontrado cosas que revisar` | Algo sale con ✘ | Mira las líneas con ✘ en la salida de Ansible |

Registros en la máquina:

```bash
sudo mesalista logs                          # la API
journalctl -u mesalista-web -n 50            # la web
sudo tail -n 50 /var/log/nginx/mesalista-error.log
```

---

## 11. Qué queda en la máquina

| Dónde | Qué |
|---|---|
| `~/despliegue-mesalista-ansible` | Este proyecto (con tu `vault.yml` cifrado) |
| `/srv/mesalista/Servidor`, `/srv/mesalista/WEB` | La copia de los repositorios que baja Ansible (no se editan a mano) |
| `/opt/mesalista-servidor` | La API instalada, con `config/config.json` (claves de PayPal, correo, Firebase) |
| `/opt/mesalista-web` | La web instalada |
| `/etc/mesalista/install.env` | Dominio y correos del servidor |
| `/etc/nginx/sites-available/mesalista` | El sitio de nginx (web + `/api/v1`) |
| `mesalista-api`, `mesalista-web`, `nginx`, `postgresql@16-main` | Los servicios (`systemctl status …`) |

Si ya habías instalado MesaLista a mano (con `~/Servidor` y `~/MesaLista-web`),
Ansible usa sus propias copias en `/srv/mesalista`. Los instaladores conservan
la base de datos, `config.json` y las sesiones abiertas. Las copias antiguas de
tu carpeta personal se pueden borrar cuando todo funcione.

---

## 12. Glosario

| Término | Qué es |
|---|---|
| **Ansible** | Programa que lleva una máquina al estado descrito en unos archivos YAML |
| **Playbook** | Archivo con la lista de pasos (`desplegar.yml`) |
| **Rol** | Grupo de pasos con un propósito (`servidor`, `web`…) |
| **Tarea** | Un paso: instalar un paquete, bajar un repositorio, ejecutar un script |
| **Inventario** | Lista de máquinas y cómo conectarse (`inventario-local.yml`: la propia) |
| **Conexión local** | Ansible actúa sobre la máquina en la que se ejecuta, sin SSH |
| **Variable** | Un parámetro con nombre (`mesalista_dominio`) |
| **Vault** | Archivo de secretos cifrado con una contraseña |
| **Tag** | Etiqueta para ejecutar solo una parte (`--tags web`) |
| **Idempotente** | Ejecutarlo dos veces deja el mismo resultado que una: no estropea ni duplica nada |
| **`become` / sudo** | Ejecutar como administrador (`root`) |
