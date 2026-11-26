# MesaLista · despliegue con Ansible

Instala o actualiza **MesaLista** en una máquina virtual **Ubuntu Server 24.04
de Azure**, por SSH, con una sola orden:

- el **Servidor** ([LXAST-MesaLista/Servidor](https://github.com/LXAST-MesaLista/Servidor)):
  API en Node.js, PostgreSQL, nginx y HTTPS con Let's Encrypt;
- la **WEB** ([LXAST-MesaLista/WEB](https://github.com/LXAST-MesaLista/WEB)): web de clientes
  y panel del restaurante;
- y los servicios externos: **PayPal** sandbox (señal de los grupos), **correo**
  de Azure Communication Services (restablecer la contraseña) y **Firebase**
  (avisos push de la app).

```
Tu ordenador (Ansible)  ──SSH──►  Máquina de Azure (Ubuntu 24.04)
                                    ├── git clone Servidor → despliegue/install.sh --no-interactivo
                                    ├── mesalista paypal / correo / push
                                    ├── git clone WEB      → despliegue/instalar.sh --no-interactivo
                                    └── mesalista estado
```

Ansible **no rehace la instalación**: baja los repositorios y ejecuta sus
propios instaladores sin preguntas, con los datos de `group_vars/`. Así solo hay
una forma de instalar MesaLista, a mano o con Ansible. Se puede ejecutar tantas
veces como haga falta: actualiza el código y no borra datos.

## Carpetas

| Archivo | Qué hay |
|---|---|
| `inventario.yml` | La máquina: dominio o IP, usuario SSH y clave |
| `inventario-local.yml` | Para ejecutar Ansible **dentro** de la propia máquina (conexión local) |
| `group_vars/all/main.yml` | Parámetros preconfigurados: dominio, correos, repositorios, ramas… |
| `group_vars/all/vault.yml.example` | Modelo de los secretos (contraseña del administrador, PIN, PayPal, correo de Azure) |
| `group_vars/all/vault.yml` | Tus secretos, **cifrados** con `ansible-vault` (no se sube a Git) |
| `files/firebase-cuenta-servicio.json` | Cuenta de servicio de Firebase, si quieres avisos push (no se sube a Git) |
| `desplegar.yml` | Despliegue completo |
| `comprobar.yml` | Solo comprobar el estado, sin cambiar nada |
| `roles/base` | Comprobaciones, git y la carpeta del código |
| `roles/servidor` | Servidor (`install.sh`) y servicios externos (`integraciones.yml`) |
| `roles/web` | Web (`instalar.sh`) |
| `roles/comprobacion` | `mesalista estado` y una prueba de la API desde fuera |
| `docs/` | Manual para desplegar desde dentro de la máquina, y `integracion-tecnologias.md` (todas las tecnologías y cómo se integran; copia del repositorio Servidor), en `.md` y `.pdf` |

**¿Ya estás dentro de la máquina por SSH?** Puedes ejecutar Ansible allí mismo,
sin WSL: [docs/manual-despliegue-en-la-maquina.md](docs/manual-despliegue-en-la-maquina.md)
(también en PDF). Este README explica la otra forma: desde tu ordenador.

## 1. Antes de empezar

**En Azure:**

1. Máquina virtual **Ubuntu Server 24.04 LTS** con IP pública **estática** y
   **etiqueta DNS** (p. ej. `mesalista.spaincentral.cloudapp.azure.com`).
2. Grupo de seguridad de red: **80 y 443** abiertos a todos; **22** solo desde tu IP.
3. Tu clave SSH (`.pem`) para entrar como `azureuser` (o el usuario que pusiste).

**En tu ordenador**, Ansible solo funciona en Linux o macOS. En **Windows**, usa **WSL**:

```powershell
wsl --install -d Ubuntu
```

Reinicia si te lo pide, abre «Ubuntu» desde el menú Inicio y sigue dentro de
esa terminal:

```bash
sudo apt update && sudo apt install -y ansible git
ansible --version                      # ansible-core 2.14 o posterior
```

Copia la clave SSH dentro de WSL (en `/mnt/c/…` los permisos no se pueden
ajustar y SSH la rechaza):

```bash
mkdir -p ~/.ssh
cp /mnt/c/Users/<tu-usuario>/.ssh/mesalista.pem ~/.ssh/
chmod 600 ~/.ssh/mesalista.pem
ssh -i ~/.ssh/mesalista.pem azureuser@mesalista.spaincentral.cloudapp.azure.com   # acepta la huella con «yes» y sal con exit
```

También sirve **Azure Cloud Shell** (en el portal de Azure), que ya trae Ansible.

## 2. Configurar

```bash
git clone https://github.com/LXAST-MesaLista/despliegue-mesalista-ansible.git
cd despliegue-mesalista-ansible
```

**Si trabajas en WSL dentro de `/mnt/c/…`**, Ansible ignora `ansible.cfg` por
seguridad (la carpeta tiene permisos de escritura para todos). Clona el
proyecto en tu carpeta de WSL (`~`), o ejecuta antes `export ANSIBLE_CONFIG=$PWD/ansible.cfg`.

### 2.1 La máquina: `inventario.yml`

```yaml
ansible_host: mesalista.spaincentral.cloudapp.azure.com   # o la IP pública
ansible_user: azureuser
ansible_ssh_private_key_file: ~/.ssh/mesalista.pem
```

### 2.2 Los parámetros: `group_vars/all/main.yml`

| Variable | Qué es | Valor de partida |
|---|---|---|
| `mesalista_dominio` | Dominio público (web y API) | `mesalista.spaincentral.cloudapp.azure.com` |
| `mesalista_correo_letsencrypt` | Correo para los avisos del certificado | **Cámbialo** (`cambiar@ejemplo.com` no vale) |
| `mesalista_correo_admin` | Cuenta de administrador de MesaLista | `admin@mesalista.com` |
| `mesalista_repo_servidor` / `mesalista_rama_servidor` | Repositorio y rama del Servidor | `LXAST-MesaLista/Servidor`, `main` |
| `mesalista_repo_web` / `mesalista_rama_web` | Repositorio y rama de la WEB | `LXAST-MesaLista/WEB`, `main` |
| `mesalista_carpeta_codigo` | Dónde se bajan los repositorios en la máquina | `/srv/mesalista` |
| `mesalista_instalar_web` | Instalar también la web | `true` |
| `mesalista_correo_remitente` | Remitente del correo de Azure (`DoNotReply@….azurecomm.net`) | vacío |
| `mesalista_firebase_cuenta_servicio` | Dónde está la cuenta de servicio de Firebase | `files/firebase-cuenta-servicio.json` |
| `mesalista_borrar_base_de_datos` | **Borra todo** y crea la base de datos de nuevo | `false` |

### 2.3 Los secretos: `group_vars/all/vault.yml`

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
nano group_vars/all/vault.yml          # rellena los valores
ansible-vault encrypt group_vars/all/vault.yml   # elige una contraseña y guárdala bien
```

| Secreto | Qué es | Si se deja vacío |
|---|---|---|
| `vault_admin_password` | Contraseña del administrador (10 caracteres o más). Solo se usa al crear la cuenta | Obligatorio |
| `vault_pin_cerrar_local` | PIN de «Cerrar local» (4 a 8 cifras). Solo se usa la primera vez | Obligatorio |
| `vault_paypal_client_id`, `vault_paypal_secret` | Claves de la app de PayPal sandbox | Sin pago de señal |
| `vault_correo_conexion` | Cadena de conexión de Azure Communication Services | Sin correo |

Para verlo o cambiarlo después: `ansible-vault edit group_vars/all/vault.yml`.

### 2.4 Firebase (opcional)

Consola de Firebase > *Configuración del proyecto* > *Cuentas de servicio* >
*Generar nueva clave privada*, y guarda el archivo como
`files/firebase-cuenta-servicio.json`. Ansible lo copia a la máquina, lo instala
con `mesalista push` y lo borra de la máquina al terminar.

## 3. Desplegar

```bash
ansible-playbook desplegar.yml --ask-vault-pass
```

La primera vez tarda unos 10 minutos. Al final sale el resultado de
`mesalista estado` y una prueba de `https://<dominio>/api/v1/config` desde tu ordenador.

| Quieres… | Orden |
|---|---|
| Ver qué haría, sin cambiar nada | `ansible-playbook desplegar.yml --ask-vault-pass --check` (solo las tareas que lo admiten) |
| Solo el servidor | `ansible-playbook desplegar.yml --ask-vault-pass --tags servidor` |
| Solo PayPal, correo y Firebase | `ansible-playbook desplegar.yml --ask-vault-pass --tags integraciones` |
| Solo la web | `ansible-playbook desplegar.yml --ask-vault-pass --tags web` |
| Solo comprobar | `ansible-playbook comprobar.yml --ask-vault-pass` |
| Actualizar tras un cambio en GitHub | La misma orden de desplegar: baja la última versión de cada rama |
| Empezar de cero (**borra todas las reservas y cuentas**) | `ansible-playbook desplegar.yml --ask-vault-pass -e mesalista_borrar_base_de_datos=true` |

## 4. Qué hace, paso a paso

1. **base**: comprueba que están los secretos y que los parámetros tienen buena
   forma, avisa si la máquina no es Ubuntu 24.04, instala `git` y crea `/srv/mesalista`.
2. **servidor**:
   1. `git clone` / `git pull` de la rama del Servidor en `/srv/mesalista/Servidor`.
   2. `install.sh --no-interactivo --dominio <dominio>` con los datos por variables
      de entorno: Node.js, PostgreSQL (roles, base de datos, esquema, permisos),
      la API como servicio, nginx, cortafuegos, certificado de Let's Encrypt,
      tareas programadas y, la primera vez, la cuenta de administrador y el PIN.
   3. **integraciones**: compara PayPal, el correo y Firebase con lo que ya tiene
      el servidor y, solo si cambian, los configura con `mesalista paypal`,
      `mesalista correo` y `mesalista push`. Estos comandos comprueban las claves
      con PayPal, Azure o Google antes de guardarlas.
3. **web**: `git clone` / `git pull` de la WEB en `/srv/mesalista/WEB` e
   `instalar.sh --no-interactivo`: la web como servicio y el sitio de nginx
   completo (web + `/api/v1`).
4. **comprobacion**: `mesalista estado` y una petición HTTPS desde tu ordenador.

Los secretos nunca salen en pantalla (`no_log`) ni en la línea de órdenes de la
máquina: llegan a los instaladores por variables de entorno.

## 5. Si algo falla

| Mensaje | Qué hacer |
|---|---|
| `UNREACHABLE` / `Permission denied (publickey)` | Revisa `ansible_host`, `ansible_user` y la clave (`chmod 600`). Prueba antes con `ssh` a mano |
| `Host key verification failed` | Conéctate una vez con `ssh` y acepta la huella |
| `Faltan los secretos` | Crea `vault.yml` (2.3) y usa `--ask-vault-pass` |
| `Attempting to decrypt but no vault secrets found` | Falta `--ask-vault-pass` |
| `Revisa group_vars/all/main.yml` | Pon tu correo en `mesalista_correo_letsencrypt` |
| `install.sh ha terminado con error` | Mira las últimas líneas que muestra Ansible; en la máquina, `sudo mesalista logs` |
| Aviso «No se llega a https://…» | Puertos 80 y 443 en el grupo de seguridad de red de Azure, y que el dominio apunte a la IP |
| `PayPal: PayPal sandbox no acepta esas claves` | Client ID o Secret mal copiados, o de otra app (Live en vez de Sandbox) |
| `Firebase: … no es el archivo JSON de una cuenta de servicio` | Has puesto `google-services.json` (de la app). Hace falta la cuenta de servicio |

En la máquina: `sudo mesalista estado`, `sudo mesalista logs`,
`journalctl -u mesalista-web -n 50` y `/var/log/nginx/mesalista-error.log`.

## 6. Qué se ha añadido en Servidor y WEB para esto

- **Servidor**, `despliegue/install.sh --no-interactivo` (o `ML_NO_INTERACTIVO=1`).
  No pregunta nada: los datos llegan en `SERVER_DOMAIN`, `LE_EMAIL`,
  `ADMIN_EMAIL`, `ML_ADMIN_PASSWORD`, `ML_CLOSE_PIN` y, si se quiere,
  `MAIL_CONNECTION` y `MAIL_SENDER`. Si falta uno obligatorio, para con un
  mensaje claro. Con `--reset-db` pide además `ML_CONFIRMAR_RESET=BORRAR`.
- **Servidor**, `mesalista paypal` y `mesalista correo` sin preguntas:
  `ML_PAYPAL_CLIENT_ID` / `ML_PAYPAL_SECRET` y `ML_MAIL_CONNECTION` /
  `ML_MAIL_SENDER` (y `ML_MAIL_TEST` para un correo de prueba).
- **WEB**, `despliegue/instalar.sh --no-interactivo`. Además, Apache (de una
  instalación anterior) se apaga **antes** de instalar nginx: si ocupaba los
  puertos 80 y 443, la instalación de nginx fallaba.

Sin la opción, los dos instaladores siguen funcionando como siempre, con preguntas.
