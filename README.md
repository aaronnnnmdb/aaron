import telebot
import yt_dlp
import os
import qrcode
import requests
from io import BytesIO

# Token del bot
TOKEN = '7993524400:AAHV54CX2CNPuNoCGPe0Jr1whEcvP9gok1Y'
bot = telebot.TeleBot(TOKEN)

# Carpeta para guardar los archivos descargados
DOWNLOAD_FOLDER = '/data/data/com.termux/files/home/Download/'
if not os.path.exists(DOWNLOAD_FOLDER):
    os.makedirs(DOWNLOAD_FOLDER)

# Función para descargar música o video con progreso
def descargar(link, formato, chat_id, message_id):
    def progreso(d):
        if d['status'] == 'downloading':
            percent = d['downloaded_bytes'] / d['total_bytes'] * 100
            mb_downloaded = d['downloaded_bytes'] / (1024 * 1024)
            mb_total = d['total_bytes'] / (1024 * 1024)
            text = f"Descargando {mb_downloaded:.1f}MB de {mb_total:.1f}MB ({percent:.1f}%)"
            try:
                bot.edit_message_text(text, chat_id, message_id)
            except Exception as e:
                print(f"Error al actualizar el mensaje: {e}")
        elif d['status'] == 'finished':
            bot.edit_message_text("Descarga completa. Enviando archivo...", chat_id, message_id)

    ydl_opts = {
        'format': 'bestaudio/best' if formato == 'mp3' else 'bestvideo+bestaudio',
        'outtmpl': f'{DOWNLOAD_FOLDER}%(title)s.%(ext)s',
        'progress_hooks': [progreso]
    }

    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.download([link])
            info_dict = ydl.extract_info(link, download=False)
            file_name = ydl.prepare_filename(info_dict)
            file_path = file_name.replace('.webm', f'.{formato}')
            return file_path
    except Exception as e:
        print(f"Error durante la descarga: {e}")
        return None

# Función para generar un código QR
def generar_qr(enlace):
    qr = qrcode.QRCode(
        version=1,
        error_correction=qrcode.constants.ERROR_CORRECT_L,
        box_size=10,
        border=4,
    )
    qr.add_data(enlace)
    qr.make(fit=True)

    img = qr.make_image(fill='black', back_color='white')
    img_io = BytesIO()
    img.save(img_io, 'PNG')
    img_io.seek(0)
    return img_io

# Función para obtener información sobre una IP
def obtener_info_ip(ip):
    try:
        response = requests.get(f'https://ipinfo.io/{ip}/json')
        data = response.json()
        return data
    except Exception as e:
        print(f"Error al obtener información de la IP: {e}")
        return None

# Función para acortar URLs usando TinyURL
def acortar_url(url):
    try:
        response = requests.get(f'http://tinyurl.com/api-create.php?url={url}')
        short_url = response.text
        return short_url
    except Exception as e:
        print(f"Error al acortar la URL: {e}")
        return 'Ocurrió un error al acortar la URL.'

# Comando /start
@bot.message_handler(commands=['start'])
def send_welcome(message):
    bienvenida = (
        "Hola! Bienvenido a este bot creado por @aaronnnmdb!\n"
        "Este bot fue creado solo con fines de ayudar.\n"
        "\nComandos:\n"
        "\n/miusic (Link del vídeo) Mp3 o Mp4\n"
        "│\n"
        "╰─> Ejemplo: /miusic https://youtube.com Mp3\n"
        "    Help: Mp3 si quieres música, Mp4 si quieres video.\n"
        "_____________________________________________________________\n"
        "\n/locip (IP)\n"
        "│\n"
        "╰─> Te dará información de la IP.\n"
        "_____________________________________________________________\n"
        "\n/qrcode (link)\n"
        "│\n"
        "╰─> Te dará una imagen QR que te direcciona al link.\n"
        "_____________________________________________________________\n"
        "\n/shorturl (link)\n"
        "│\n"
        "╰─> Recortará un enlace."
    )
    bot.reply_to(message, bienvenida)

# Comando /miusic
@bot.message_handler(commands=['miusic'])
def enviar_musica(message):
    try:
        msg_parts = message.text.split()

        if len(msg_parts) < 3:
            bot.reply_to(message, "Uso: /miusic <link> <mp3/mp4>")
            return

        link = msg_parts[1]
        formato = msg_parts[2].lower()

        if formato not in ['mp3', 'mp4']:
            bot.reply_to(message, "Formato no válido. Usa 'mp3' o 'mp4'.")
            return

        sent_message = bot.reply_to(message, "Iniciando descarga...")
        bot.reply_to(message, "Descargando... Por favor espera.")

        # Descargar el archivo
        archivo = descargar(link, formato, message.chat.id, sent_message.message_id)

        if archivo is None or not os.path.exists(archivo):
            bot.reply_to(message, "Ocurrió un error al descargar el archivo.")
            return

        # Enviar el archivo al usuario
        with open(archivo, 'rb') as f:
            if formato == 'mp3':
                bot.send_audio(message.chat.id, f)
            elif formato == 'mp4':
                bot.send_video(message.chat.id, f)

        # Borrar el archivo después de enviarlo
        os.remove(archivo)

    except Exception as e:
        bot.reply_to(message, f"Ocurrió un error: {str(e)}")

# Comando /qrcode
@bot.message_handler(commands=['qrcode'])
def enviar_qr(message):
    try:
        msg_parts = message.text.split(maxsplit=1)

        if len(msg_parts) < 2:
            bot.reply_to(message, "Uso: /qrcode <enlace>")
            return

        enlace = msg_parts[1]
        qr_img_io = generar_qr(enlace)

        bot.send_photo(message.chat.id, qr_img_io, caption="Aquí está tu código QR.")

    except Exception as e:
        bot.reply_to(message, f"Ocurrió un error: {str(e)}")

# Comando /locip
@bot.message_handler(commands=['locip'])
def localizar_ip(message):
    try:
        msg_parts = message.text.split(maxsplit=1)

        if len(msg_parts) < 2:
            bot.reply_to(message, "Uso: /locip <IP>")
            return

        ip = msg_parts[1]
        info = obtener_info_ip(ip)

        if info is None:
            bot.reply_to(message, "Ocurrió un error al obtener la información de la IP.")
            return

        detalles = (
            f"**Información de la IP**\n\n"
            f"**IP:** {info.get('ip', 'No disponible')}\n"
            f"**Hostname:** {info.get('hostname', 'No disponible')}\n"
            f"**Ciudad:** {info.get('city', 'No disponible')}\n"
            f"**Región:** {info.get('region', 'No disponible')}\n"
            f"**País:** {info.get('country', 'No disponible')}\n"
            f"**Código del País:** {info.get('country_code', 'No disponible')}\n"
            f"**Ubicación:** {info.get('loc', 'No disponible')}\n"
            f"**Organización:** {info.get('org', 'No disponible')}\n"
        )

        bot.send_message(message.chat.id, detalles, parse_mode='Markdown')
            except Exception as e:
        bot.reply_to(message, f"Ocurrió un error: {str(e)}")

# Comando /shorturl
@bot.message_handler(commands=['shorturl'])
def acortar_link(message):
    try:
        msg_parts = message.text.split(maxsplit=1)

        if len(msg_parts) < 2:
            bot.reply_to(message, "Uso: /shorturl <URL>")
            return

        url = msg_parts[1]
        short_url = acortar_url(url)

        bot.reply_to(message, f"URL corta: {short_url}")

    except Exception as e:
        bot.reply_to(message, f"Ocurrió un error: {str(e)}")

# Iniciar el bot
if __name__ == '__main__':
    bot.polling()
   
