Tarea 2: Análisis del protocolo RTMP con SRS y VLC

Taller de Redes y Servicios 2026-2 · Universidad Diego Portales

Integrantes: Ricardo Sánchez.

Descripción

Este repositorio contiene el entorno y los resultados del estudio del protocolo **RTMP** (*Real-Time Messaging Protocol*), que transmite audio y video en vivo sobre TCP (puerto 1935).

| Rol | Software | Contenedor |
|---|---|---|
| Servidor | [SRS](https://github.com/ossrs/srs) (imagen `ossrs/srs:5`) | `srs` |
| Cliente | [VLC](https://github.com/videolan/vlc) 3.0.20 en modo consola (`cvlc`) | `vlc` |

El cliente VLC puede publicar un flujo hacia SRS y también reproducirlo. Se captura el tráfico con tshark y se analiza con Wireshark (el análisis completo está en el informe).

Estructura del repositorio

```
.
├── docker-compose.yml     
├── servidor/
│   └── srs.conf           
├── cliente/
│   ├── Dockerfile          
│   └── videos/           
├── capturas/
│   └── rtmp.pcap          
├── informe/
│   ├── informe_tarea2.tex  
│   ├── informe_tarea2.pdf
│   └── img/                
└── README.md
```

Requisitos

- Docker Engine y Docker Compose
- Wireshark (para abrir la captura en el equipo anfitrión)
- Probado en Fedora con SELinux activo (por eso los volúmenes llevan `:Z`)

Puesta en marcha

1. Clonar el repositorio y entrar a la carpeta:
   git clone COMPLETAR_URL_DEL_REPOSITORIO
   cd tarea2
   
3. Crear las carpetas compartidas con permisos de escritura:
   mkdir -p cliente/videos capturas
   chmod 777 cliente/videos capturas
   
4. Validar y levantar los contenedores:
   docker compose config
   docker compose up -d --build
   docker compose ps
   
5. Comprobar que SRS responde:
   curl http://localhost:1985/api/v1/versions


Puertos publicados

| Puerto | Uso |
|---|---|
| 1935/tcp | RTMP |
| 1985/tcp | API HTTP de SRS |
| 8080/tcp | Servidor HTTP (HLS y HTTP-FLV) |

Uso

Se usan tres terminales en la carpeta del proyecto.

1. Crear el video de prueba

El video se incluye en el repositorio. Pero se puede generar dentro del contenedor (640x360 a 25 cuadros por segundo):

docker compose exec vlc ffmpeg -f lavfi -i testsrc=duration=180:size=640x360:rate=25 \
  -f lavfi -i sine=frequency=440:duration=180 -c:v libx264 -c:a aac -pix_fmt yuv420p /tmp/largo.mp4
docker compose exec -u root vlc cp /tmp/largo.mp4 /home/vlc/videos/largo.mp4


Para un video de 30 s (`test.mp4`), usar `duration=30` y cambiar el nombre del archivo.

2. Iniciar la captura (terminal 1)

Siempre antes de generar el tráfico, para no perder el *handshake*:
docker compose exec -u root vlc tshark -i eth0 -w /tmp/rtmp.pcap


### 3. Publicar con VLC (terminal 2)
docker compose exec vlc cvlc videos/largo.mp4 --play-and-exit \
  --sout #transcode{vcodec=h264,acodec=mp4a}:std{access=rtmp,mux=ffmpeg{mux=flv},dst=rtmp://srs/live/stream}'


Alternativa con FFmpeg, que es el método que usa la documentación de SRS:

docker compose exec vlc ffmpeg -re -stream_loop -1 -i videos/test.mp4 -c copy -f flv rtmp://srs/live/stream


4. Verificar y reproducir (terminal 3)


curl http://localhost:1985/api/v1/streams/
curl http://localhost:1985/api/v1/clients/
docker compose exec vlc cvlc rtmp://srs/live/stream --vout dummy --aout dummy -vvv


`--vout dummy` y `--aout dummy` se usan porque el contenedor no tiene pantalla ni audio.

5. Cortar la captura y copiarla

Detener el publicador y luego tshark con `Ctrl+C`, y copiar el archivo:


docker cp vlc:/tmp/rtmp.pcap capturas/rtmp.pcap
sudo chown $USER:$USER capturas/rtmp.pcap


Resultados principales

- VLC **sí logra publicar** por RTMP con `--sout`. Aparecen advertencias del convertidor de video (`Too high level of recursion`) y del muxer FLV (`non monotonically increasing dts`), que descarta algunos cuadros, pero la publicación continúa.
- SRS detecta el flujo `live/stream` con video H.264 (perfil High, 640x360) y audio AAC LC (44 100 Hz, 2 canales).
- SRS identifica al emisor como `fmle-publish` y al reproductor como `rtmp-play`.
- La captura (`rtmp.pcap`, 34 132 paquetes en 773 s) confirma el handshake con C0+C1 de 1537 bytes, S0+S1+S2 de 3073 y C2 de 1536, seguido de `connect`, `createStream`, `publish` y `play`.
- Ambos extremos anuncian un tamaño de chunk de 60 000 bytes (frente a los 128 por defecto de la especificación) y SRS fija una ventana de confirmación de 2 500 000 bytes.
- El emisor usa la secuencia `releaseStream`, `FCPublish`, `createStream` y `publish`, con respuesta `NetStream.Publish.Start`.
- Ninguna conexión cierra con FIN: VLC termina emisor y receptor con RST, y solo el emisor envía `deleteStream`.

Problemas conocidos

| Problema | Solución aplicada |
|---|---|
| `services.volumes must be a mapping` | Reescribir el compose completo y validar con `docker compose config` |
| `Permission denied` al escribir en carpetas montadas | `:Z` en los volúmenes (SELinux), `chmod 777` y captura en `/tmp` con `docker cp` |
| Errores de PulseAudio y D-Bus en VLC | Sin relevancia: el contenedor no tiene audio ni entorno gráfico |
| `curl` sin respuesta en el puerto 1985 | Los contenedores estaban detenidos; levantar con `docker compose up -d` |

Informe y video

- Informe: `informe/informe_tarea2.pdf` (capítulos 1 y 2; el capítulo 3 se entrega en la Tarea 3).
- Video (máximo 8 minutos): me falta el enlace djkshlads.

Referencias

- Adobe Systems, *Adobe's Real Time Messaging Protocol (RTMP) Specification*, 2012. <https://rtmp.veriskope.com/docs/spec>
- SRS: <https://github.com/ossrs/srs> y <https://ossrs.net/lts/en-us/docs/v4/doc/getting-started>
- VLC: <https://github.com/videolan/vlc> y <https://wiki.videolan.org/Documentation:Streaming_HowTo/>
- Docker Compose: <https://docs.docker.com/compose/>
