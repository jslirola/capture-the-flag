# Misión#005

## Introducción:

> ¡Neo, tenemos un problema! Han secuestrado a Morfeo y no sabemos donde lo pueden tener. Necesitamos que investigues y descubras su localización para rescatarle. La única pista que tenemos es una URL que conseguimos. ¿Serás capaz de encontrarle?

## Información adicional:

> URL conseguida: http://34.253.233.243/search/localizacion.php

> Tip: La flag es el nombre del sitio donde se encuentra con el formato UAM{Localización}.

> Tip2: El nombre del sitio en la flag es con "_" en lugar de espacios.

> Tip3: El archivo ".zip" se descomprime con "123mango".

> Tip4: Hay una flag trampa la cuál no tiene localización.

## Solución

Accedemos a la URL que se nos proporciona y nos redirecciona a otra página nada más entrar. 

```html
<meta http-equiv="refresh" content="0; URL=http://34.253.233.243/search/index.php" />
```

Si la detenemos encontramos en el código fuente información útil:

> Para continuar deberéis sacar X información del primer archivo (la cuál está encriptada) y pasársela al segundo archivo: </br>Archivo 1: https://goo.gl/K1dcbG </br>Archivo 2: https://drive.google.com/open?id=1CAz5xxsf9YxGlSWDgOVURsvFmT6A1Swn </br>

**Archivo 1**: *morfeo.jpg*

![Imagen](morfeo.jpg "Imagen de Morfeo")

**Archivo 2**: *web.py*

```python
#!/usr/bin/python3

string = input("Introduce la información que hayas sacado de la imagen: ")

a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p, q, r = string

a = a.lower()
b = b.lower()
c = c.lower()
d = d.lower()
e = e.lower()
f = '://'
g = g.lower()
h = h.lower()
i = i.lower()
j = '.'
k = k.lower()
l = l.lower()
m = '/'
n = n.upper()
o = o.lower()
p = p.upper()
q = q.upper()
r = r.lower()
s = '2'

print (a + b + c + d + e + f + g + h + i + j + k + l + m + n + o + p + q + r + s)

```

Comenzamos revisando metadatos de la imagen:

```bash
➜ exiftool morfeo.jpg 
ExifTool Version Number         : 9.46
File Name                       : morfeo.jpg
Directory                       : .
File Size                       : 33 kB
File Modification Date/Time     : 2018:03:17 14:22:02+01:00
File Access Date/Time           : 2018:03:17 14:22:02+01:00
File Inode Change Date/Time     : 2018:03:17 14:22:03+01:00
File Permissions                : rw-rw-r--
File Type                       : JPEG
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
XMP Toolkit                     : Image::ExifTool 10.75
Creator                         : Pass:UAM
Image Width                     : 425
Image Height                    : 465
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 425x465
```

En la etiqueta *Creator* aparece una contraseña ¿dónde la usamos? ¿Tendrá algo oculto mediante esteganografía? Vamos a probar a extraer información con la herramienta 'steghide':

```bash
➜ steghide extract -sf morfeo.jpg
Enter passphrase: UAM
wrote extracted data to "morf.txt".
```

Efectivamente hemos acertado, ahora tenemos un fichero de texto con un contenido curioso:

```
AABBBBAABBBAABBABBBBBAABA AABBAABBBAABBBA AABBAABABB AABABABABABAAAABAAABAAABA
```

Tenemos que descifrar el texto antes de continuar ya que el script en python no lo reconoce aún.

Tras probar diferentes algoritmos de cifrado llegamos al correcto: Baconian Cipher, el texto plano corresponde a: `HTTPS GOO GL FKQRC`. La herramienta online usada para decodificarlo:

http://rumkin.com/tools/cipher/baconian.php

El siguiente paso es introducir dicha información en el script en python que corresponde con el segundo archivo extraído de la imagen de morfeo.

```bash
➜ Mision5 python3 web.py
Introduce la información que hayas sacado de la imagen: HTTPS GOO GL FKQRC
https://goo.gl/FkQRc2
```

Ahora deberíamos seguir ese enlace acortado de Google ¿Entramos directamente o probamos alguna forma de descubrir el destino de este enlace? Hay un método para descubrir esta información sin abrirlo explícitamente, añadimos `.info` al final de la URL y...

https://goo.gl/FkQRc2.info => https://goo.gl/#analytics/goo.gl/FkQRc2/all_time

Tenemos las estadísticas de dicho enlace: clicks realizados, referidos, navegadores web, países, plataformas y el enlace de destino: https://drive.google.com/open?id=1YmYxO0JGf1xoo9HuNESihe0hQdeXTBVT

Corresponde con un fichero zip bastante pesado (~180MB) en comparación a lo que ya hemos usado. Lo descomprimimos con la contraseña que nos dan en el enunciado `123mango`.

Se trata de un volcado de memoria RAM, lo analizamos con volatility, el primer paso es conseguir el perfil del sistema para aplicar unas funciones u otras:

```bash
➜ volatility -f morfeo.dmp imageinfo

      Suggested Profile(s) : Win8SP0x64, Win81U1x64, Win2012R2x64_18340, Win10x64_14393, Win10x64, Win2016x64_14393, Win10x64_16299, Win2012R2x64, Win2012x64, Win8SP1x64_18340, Win10x64_10586, Win8SP1x64, Win10x64_15063 (Instantiated with Win10x64_15063)
                 AS Layer1 : SkipDuplicatesAMD64PagedMemory (Kernel AS)
                 AS Layer2 : WindowsCrashDumpSpace64 (Unnamed AS)
                 AS Layer3 : FileAddressSpace (/tmp/morfeo.dmp)
                  PAE type : No PAE
                       DTB : 0x187000L
                      KDBG : 0xf800028040a0L
      Number of Processors : 1
 Image Type (Service Pack) : 1
            KPCR for CPU 0 : 0xfffff80002805d00L
         KUSER_SHARED_DATA : 0xfffff78000000000L
       Image date and time : 2018-03-12 20:35:20 UTC+0000
 Image local date and time : 2018-03-12 21:35:20 +0100
```

Teniendo el sistema operativo podemos buscar información relevante en procesos, historial de comandos, conexiones de red, ficheros, etc.

Encontramos algo interesante en la Master File Table (MFT) del sistema:

```bash
➜ mkdir dump
➜ volatility -f morfeo.dmp --profile=Win8SP0x64 -D dump --output-file=resumenMFT.txt mftparser 
Volatility Foundation Volatility Framework 2.6
Outputting to: resumenMFT.txt
Scanning for MFT entries and building directory, this can take a while
```

Abrimos el fichero de texto de salida y buscamos palabras relacionadas con la solución, yo probé con "Morfeo" y acerté:

```
***************************************************************************
***************************************************************************
MFT entry found at offset 0x1289bc00
Attribute: In Use & File
Record Number: 60979
Link count: 1

$STANDARD_INFORMATION
Creation                       Modified                       MFT Altered                    Access Date                    Type
------------------------------ ------------------------------ ------------------------------ ------------------------------ ----
2018-03-12 20:33:51 UTC+0000 2018-03-12 20:33:51 UTC+0000   2018-03-12 20:33:59 UTC+0000   2018-03-12 20:33:51 UTC+0000   Archive

$FILE_NAME
Creation                       Modified                       MFT Altered                    Access Date                    Name/Path
------------------------------ ------------------------------ ------------------------------ ------------------------------ ---------
2018-03-12 20:33:51 UTC+0000 2018-03-12 20:33:51 UTC+0000   2018-03-12 20:33:51 UTC+0000   2018-03-12 20:33:51 UTC+0000   Users\anubis\Desktop\uam.jpg

$OBJECT_ID
Object ID: dec70d8f-3426-e811-9f5d-08002736a682
Birth Volume ID: 80000000-a000-0000-0000-180000000100
Birth Object ID: 81000000-1800-0000-3c68-746d6c3e0d0a
Birth Domain ID: 093c6865-6164-3e0d-0a09-093c7469746c

$DATA
0000000000: 3c 68 74 6d 6c 3e 0d 0a 09 3c 68 65 61 64 3e 0d   <html>...<head>.
0000000010: 0a 09 09 3c 74 69 74 6c 65 3e 43 6f 6f 72 64 65   ...<title>Coorde
0000000020: 6e 61 64 61 73 20 64 65 20 4d 6f 72 66 65 6f 3c   nadas.de.Morfeo<
0000000030: 2f 74 69 74 6c 65 3e 0d 0a 09 3c 2f 68 65 61 64   /title>...</head
0000000040: 3e 0d 0a 09 3c 62 6f 64 79 3e 0d 0a 09 09 3c 68   >...<body>....<h
0000000050: 31 3e 34 30 2e 37 34 38 34 34 30 35 2c 20 2d 37   1>40.7484405,.-7
0000000060: 33 2e 39 38 35 36 36 34 34 3c 2f 68 31 3e 0d 0a   3.9856644</h1>..
0000000070: 09 3c 2f 62 6f 64 79 3e 0d 0a 3c 2f 68 74 6d 6c   .</body>..</html
0000000080: 3e                                                >
```

Las coordenadas (40.7484405 -73.9856644) nos llevan a Wall Street y al Edificio "Empire State", nuestra flag:

https://www.google.es/maps/place/40%C2%B044'54.4%22N+73%C2%B059'08.4%22W/@40.7481377,-73.9863678,378m/data=!3m1!1e3!4m5!3m4!1s0x0:0x0!8m2!3d40.7484405!4d-73.9856644

### Flag: UAM{Empire_State}

Como curiosidad, encontré directamente las coordenadas correctas y no vi nada de la flag falsa que habían puesto en el reto. Tras analizar de nuevo la MFT buscando otras palabras clave dí con ella UAM{N30_i5_4_G0D}:

```
$DATA
0000000000: 3c 68 74 6d 6c 3e 0d 0a 09 3c 68 65 61 64 3e 0d   <html>...<head>.
0000000010: 0a 09 09 3c 74 69 74 6c 65 3e 55 41 4d 20 46 4c   ...<title>UAM.FL
0000000020: 41 47 3c 2f 74 69 74 6c 65 3e 0d 0a 09 3c 2f 68   AG</title>...</h
0000000030: 65 61 64 3e 0d 0a 09 3c 62 6f 64 79 3e 0d 0a 09   ead>...<body>...
0000000040: 09 3c 68 31 3e 55 41 4d 7b 4e 33 30 5f 69 35 5f   .<h1>UAM{N30_i5_
0000000050: 34 5f 47 30 44 7d 3c 2f 68 31 3e 0d 0a 09 3c 2f   4_G0D}</h1>...</
0000000060: 62 6f 64 79 3e 0d 0a 3c 2f 68 74 6d 6c 3e         body>..</html>
```