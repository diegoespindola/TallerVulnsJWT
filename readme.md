# Taller de Vulnerabilidades en jwt

Un taller mas sobre vulnerabilidades en JWT, este es practico, paso a paso y en español.

Toda la teoria y conocimientos previos seran explicados en el taller en vivo. 

Resolveremos los laboratorios de JWT de Portswigger Academy pero sin BurpSuite.

## Laboratorio

Para realizar el taller paso a paso debemos tener lo siguiente:

### Webs

 - Necesitamos una cuenta activa en la plataforma de [https://portswigger.net/](https://portswigger.net/users/register)
 - Usaremos https://token.dev/ para revisar los jwt.
 - Para transformar los certificados usaremos <https://8gwifi.org/jwkconvertfunctions.jsp>

### Diccionarios

#### SecLists
Usaremos un archivo de la coleccion  [SecLists](https://github.com/danielmiessler/SecLists) de Daniel Miessler.

`git clone https://github.com/danielmiessler/SecLists.git`

#### jwt-secrets
Usaremos el archivo de claves filtradas de jwt [jwt-secrets](https://github.com/wallarm/jwt-secrets/) de Wallarm.

`git clone https://github.com/wallarm/jwt-secrets.git`


### Herramientas

Las siguientes herramientas deben estar instaladas

- #### CAIDO

	Debemos crear una cuenta en <https://caido.io/> y descargar el instalador para nuestro sistema operativo

- #### GIT

	Debemos instalar git segun las siguientes [instrucciones](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

- #### Python3

	Instalaremos Python 3 segun las siguientes [instrucciones](https://www.python.org/downloads/)

- #### Ffuf
	instalaremos Ffuf segun las instrucciones en la url <https://github.com/ffuf/ffuf>

- #### HashCat

	sudo apt install hashcat -y

- #### JWT-TOOL 

	Instalaremos [JWT_TooL](https://github.com/ticarpi/jwt_tool)

	`git clone https://github.com/ticarpi/jwt_tool`

	`python3 -m pip install -r requirements.txt`

## Vulnerabilidades

### No valida firma

Este ataque aprovecha que algunas implementaciones de librerias de JWT tienen distintos metodos para validar el token y obtener los claims, entonces algunos desarroladores obtienen los claims antes de validar la firma del token. la que podria ser invalida.

[Laboratorio](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature)

- Tal como dice el laboratorio, hay que loguearse con las credenciales wiener:peter todo esto pormedio de CAIDO
- Con lo anterior obtenemos un JWT y lo llevamos a [token.dev](https://token.dev/)
- Podemos observar que trae un claim llamado "sub" con el login y segun las indicaciones del lab debemos loguearnos con el usuario **administrator**
- Modificamos el claim reemplazando wiener por administrator
- La herramienta esta usando una firma propia, no es la firma del sitio, por lo que al validar el token con la clave original deberia ser rechazado
- Logueados como wiener vamos a la seccion de administrador que indica el lab https://xx.web-security-academy.net/admin 
- Desde CAIDO enviamos ese request a **Replay***
- Reemplazamos el token por el generado y click a **Send**
- para automatizar el reemplazo del token vamos a **Match & Replace**
	- Creamos una nueva regla
		- Section : Request Header
		- Matcher : String
		- Replacer: String
	- activar
- Navegamos


### Alg none

Esta vulnerabilidad se aprovecha que algunas implementaciones utilizan siempre el algoritmo indicado en la cabecera para validar el token. 

[Laboratorio](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification)

### Claves debiles

Un token con una clave debil puede ser facilmente falsificado, obteniendo la clave por fuerza bruta.

[Laboratorio](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key)

- Navegar a la url del lab
- Obtener el jwt desde CAIDO
- Usar un ataque de  diccionario para obtener la clave publica del jwt


	`hashcat -a 0 -m 16500 <TOKEN> ../jwt-secrets/jwt.secrets.list --show`

	- `-a 0` ataque modo Straight, es decir linea a linea
	- `-m 16500` ataque a jwt
	- `<TOKEN>` el jwt obtenido con CAIDO
	- `../jwt-secrets/jwt.secrets.list` diccionario de claves comunes

- Usar jwt_tool para generar un token falsificado valido

	`python3 ./jwt_tool.py -p <password> <JWT> -T -S hs256`
	
	- `-p <password>` la contraseña obtenida con hashcat
	- `-T` Tampering (modificacion) del jwt
	- `-S hs256` algoritmo que usara para firmar el jwt falso
- Usar el token resultante en la Web a traves de match&replace de CAIDO



### jwk injection

Inyectaremos la clave para validar el token directamente en el header del token

[Laboratorio](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection)





### Key confusion attack

Este ataque se aprovecha que algunas librerias usan siempre el algoritmo que viene en el Header par validar los tokens, entonces se cambia el algoritmo de cifrado desde uno asincronico(clave de cifrado y validacion distintas) como el RS256 a uno sincronico(clace de firmado distinta a la clave de validacion) como es el HS256, basicamente el sistema deberia validar con la misma clave el token firmado con la clave publica, entonces firmamos el jwt con el algoritmo sincronico y usamos la misma clave publica (de validacion) para firmar el token. en resumen el sistema se confunde y usa la clave publica como clave privada.

#### Taller:

 Usaremos el siguiente [Laboratorio de portswigger](https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion)

- Abrimos la url del laboratorio con CAIDO
	- Ingresamos con las credenciales wiener:peter
- Buscamos el token entregado por el servidor, vemos que usa KID si tenemos suerte  deberia haber un archivo con la clave publica con ese id.
- Usamos ffuf para ver si hay algun archivo esta expuesto con las claves publicas 

	`ffuf -u https://lab1.web-security-academy.net/FUZZ -w ~/tools/diccionarios/SecLists/Discovery/Web-Content/common.txt -mc 200`

- Con lo anterior descubrimos un archivo https://lab1.web-security-academy.net/jwks.json

- Transformamos el jwk a formato pem en <https://8gwifi.org/jwkconvertfunctions.jsp> solo el elemento dentro de "keys"

- Usamos el PEM para firmar nuestro tokenmodificado en <https://token.dev/> 
*Base64 encoded off*
	- Pegamos nuestro token en la seccion JWT String
	- Modificamos el Header, en el campo alg reemplazamos el valor RS256 por HS256
	- Modificamos el Payload, en el campo sub reemplazamos el valor "wiener" por "administrator".
	- Pegamos el texto del archivo PEM en el campo Signing Key *Base64 encoded off*.
	- Y obtenemos nuestro token firmado.

- Para usar nuestro nuevo token primero enviaremos en CAIDO a Replay el request de "/my-account"
	- Reemplazamos el valor del parametro id, en la url, en vez de "wiener" va "administrator".
	- Reemplazamos el token por el nuestro.
	- Le damos a Send.
	- si encontramos en el response el texto "Your username is: administrator"entonces nuestro tokenfunciono ok.

- Para usar el token y resolver el laboratorios creamos 2 reglas en "match & replace" de CAIDO, donde reemplazaremos automaticamente el token real por el nuestro y reemplazaremos automaticamente el nombre de usuario de "wiener" por "administrator" y luego navegamos desde el chrome de CAIDO.
