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

![Escaneo NMAP](assets/Pasted%20image%2020260924194322.png)

El único servicio relevante es el puerto 80, por lo que se procedió a inspeccionarlo desde el navegador.

### Enumeración web

![Pasted image 20260924221152.png](assets/Pasted%20image%2020260924221152.png)

El sitio publicaba nombres de posibles usuarios del sistema, entre ellos `carlota` y `oscar`. Esa información es directamente aprovechable para un ataque de fuerza bruta contra SSH.

> [!note]- Desglose de comandos
> - `nmap`: herramienta de exploración de red y auditoría utilizada para determinar qué puertos están abiertos en la máquina objetivo.

---

## Acceso Inicial

Con los nombres de usuario identificados, se lanzó un ataque de fuerza bruta contra el servicio SSH utilizando `hydra`.

![Pasted image 20260924195231.png](assets/Pasted%20image%2020260924195231.png)

> [!success] Credenciales obtenidas
> ```text
> [ssh] host: 172.17.0.2   login: carlota   password: babygirl
> ```

Con las credenciales en mano se accedió vía SSH y se verificaron los grupos y permisos del usuario.

![Pasted image 20260924195516.png](assets/Pasted%20image%2020260924195516.png)

Ya dentro del sistema, se enumeró el archivo `/etc/passwd` para identificar otros usuarios.

![Pasted image 20260924195545.png](assets/Pasted%20image%2020260924195545.png)

> [!note]- Desglose de comandos
> - `hydra`: herramienta de fuerza bruta que soporta múltiples protocolos (en este caso, SSH).
> - `cat /etc/passwd`: muestra las cuentas registradas en el sistema, útil para identificar usuarios con shell válida.

Se identificó un segundo usuario: `oscar`. Sin sus credenciales no había forma de avanzar por ese lado, por lo que se procedió a revisar los directorios personales de `carlota`.

---

## Credenciales de Oscar vía Esteganografía

Dentro de `/home/carlota/Desktop` se encontró una carpeta llamada `Vacaciones` con una imagen `imagen.jpg`. Se analizó con `steghide` en busca de contenido oculto.

![Pasted image 20260924201908.png](assets/Pasted%20image%2020260924201908.png)

La herramienta confirmó la existencia de un archivo `secret.txt` embebido. Se procedió a extraerlo.

```bash
steghide extract -sf imagen.jpg
```

![Pasted image 20260924211335.png](assets/Pasted%20image%2020260924211335.png)

El archivo extraído contenía un string en Base64 que, al decodificarse, reveló la contraseña de `oscar`.

![Pasted image 20260924211532.png](assets/Pasted%20image%2020260924211532.png)

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

Se reutilizaron las credenciales para acceder vía SSH y se verificaron los permisos de `sudo`.

![Pasted image 20260924211825.png](assets/Pasted%20image%2020260924211825.png)

El resultado del `sudo -l` mostró una regla crítica:

```text
User oscar may run the following commands on 564fdd50fcd5:
    (ALL) NOPASSWD: /usr/bin/ruby
```

Es decir, `oscar` puede ejecutar `ruby` como `root` sin contraseña. Antes de explotarlo, se revisaron los directorios en busca de más pistas.

![Pasted image 20260924212148.png](assets/Pasted%20image%2020260924212148.png)

El archivo mencionaba revisar el escritorio de `root` en busca de un archivo de texto. Es decir, la flag final está en `/root/Desktop`.

---

## Escalada de Privilegios

Consultando [GTFOBins](https://gtfobins.github.io/gtfobins/ruby/) se identificó la técnica para abusar de `ruby` cuando se ejecuta con privilegios elevados. Ruby permite invocar una shell heredando los privilegios del proceso padre.

```bash
ruby -e 'exec "/bin/sh"'
```

![Pasted image 20260924212539.png](assets/Pasted%20image%2020260924212539.png)

> [!note]- Desglose de comandos
> - `ruby`: intérprete del lenguaje Ruby.
> - `-e`: evalúa el código pasado como argumento.
> - `exec "/bin/sh"`: reemplaza el proceso actual por una shell `/bin/sh`, heredando sus privilegios.

---

## Flag

Ya como `root`, se accedió al directorio indicado por la pista para leer la flag final.

![Pasted image 20260924212800.png](assets/Pasted%20image%2020260924212800.png)

Con esto se confirma el fin de la máquina de forma exitosa.

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
