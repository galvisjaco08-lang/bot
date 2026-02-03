import discord
from discord.ext import commands
import asyncio
import youtube_dl

# --- CONFIGURACIÓN DE AUDIO ---
ytdl_format_options = {
    'format': 'bestaudio/best',
    'restrictfilenames': True,
    'noplaylist': True,
    'nocheckcertificate': True,
    'ignoreerrors': False,
    'logtostderr': False,
    'quiet': True,
    'no_warnings': True,
    'default_search': 'auto',
    'source_address': '0.0.0.0',
}

ytdl = youtube_dl.YoutubeDL(ytdl_format_options)

class YTDLSource(discord.PCMVolumeTransformer):
    def __init__(self, source, *, data, volume=0.5):
        super().__init__(source, volume)
        self.data = data
        self.title = data.get('title')
        self.url = data.get('url')

    @classmethod
    async def from_url(cls, url, *, loop=None, stream=False):
        loop = loop or asyncio.get_event_loop()
        data = await loop.run_in_executor(None, lambda: ytdl.extract_info(url, download=not stream))
        if 'entries' in data:
            data = data['entries'][0]
        filename = data['url'] if stream else ytdl.prepare_filename(data)
        return cls(discord.FFmpegPCMAudio(filename, options='-vn'), data=data)

# --- BOT CONFIGURACIÓN ---
intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(command_prefix='$', intents=intents)

@bot.event
async def on_ready():
    print(f'Bot conectado como: {bot.user}')

# --- TUS COMANDOS ORIGINALES ---
@bot.command()
async def hello(ctx):
    await ctx.send(f'Hola, soy un bot {bot.user}!')

@bot.command()
async def heh(ctx, count_heh = 5):
    await ctx.send("he" * count_heh)

# --- COMANDOS DE MÚSICA REESTRUCTURADOS ---
@bot.command()
async def join(ctx, *, channel: discord.VoiceChannel):
    """Se une a un canal de voz"""
    if ctx.voice_client is not None:
        return await ctx.voice_client.move_to(channel)
    await channel.connect()

@bot.command()
async def play(ctx, *, url):
    """Reproduce música desde YouTube"""
    async with ctx.typing():
        player = await YTDLSource.from_url(url, loop=bot.loop, stream=True)
        ctx.voice_client.play(player, after=lambda e: print(f'Error: {e}') if e else None)
    await ctx.send(f'Reproduciendo ahora: {player.title}')

@bot.command()
async def stop(ctx):
    """Detiene la música y sale del canal"""
    await ctx.voice_client.disconnect()

# --- MEJORA: COMANDO DE AYUDA (Lo que pide tu tarea) ---
@bot.command()
async def ayuda(ctx):
    """Muestra la lista de comandos disponibles"""
    embed = discord.Embed(title="Menú de Ayuda", description="Aquí tienes mis comandos:", color=0x00ff00)
    embed.add_field(name="$hello", value="Saluda al bot", inline=False)
    embed.add_field(name="$heh [número]", value="El bot se ríe", inline=False)
    embed.add_field(name="$join [nombre_canal]", value="El bot entra al canal de voz", inline=False)
    embed.add_field(name="$play [url]", value="Reproduce música de YouTube", inline=False)
    embed.add_field(name="$stop", value="Detiene la música y sale", inline=False)
    await ctx.send(embed=embed)

# RECUERDA: No compartas tu Token en público. 
# Si el código te da error de 'youtube_dl', intenta instalar 'yt-dlp' que es más moderno.
bot.run("TOKEN")
