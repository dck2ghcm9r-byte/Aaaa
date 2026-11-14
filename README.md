import discord
from discord.ext import commands
import yt_dlp
import asyncio

# إعداد الصلاحيات (Intents)
intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(command_prefix='!', intents=intents)

# إعدادات yt-dlp لجلب الصوت بأفضل جودة
ytdl_format_options = {
    'format': 'bestaudio/best',
    'noplaylist': True,
    'quiet': True,
}
ffmpeg_options = {
    'before_options': '-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5',
    'options': '-vn',
}

ytdl = yt_dlp.YoutubeDL(ytdl_format_options)

@bot.event
async def on_ready():
    print(f'تم تسجيل الدخول باسم: {bot.user.name}')

@bot.command(name='play', help='تشغيل أغنية من يوتيوب')
async def play(ctx, *, search: str):
    # التأكد من أن العضو داخل قناة صوتية
    if not ctx.author.voice:
        return await ctx.send('❌ لازم تدخل قناة صوتية أول شيء!')

    channel = ctx.author.voice.channel

    # دخول البوت للقناة الصوتية أو الانتقال إليها
    if ctx.voice_client is None:
        await channel.connect()
    elif ctx.voice_client.channel != channel:
        await ctx.voice_client.move_to(channel)

    await ctx.send(f'🔍 جاري البحث عن: **{search}**...')

    # البحث وجلب رابط الصوت المباشر
    loop = asyncio.get_event_loop()
    data = await loop.run_in_executor(None, lambda: ytdl.extract_info(f"ytsearch:{search}", download=False))
    
    if 'entries' in data:
        data = data['entries'][0]

    song_url = data['url']
    title = data['title']

    # إيقاف أي صوت شغال حالياً وتشغيل الجديد
    if ctx.voice_client.is_playing():
        ctx.voice_client.stop()

    source = discord.FFmpegPCMAudio(song_url, **ffmpeg_options)
    ctx.voice_client.play(source)

    await ctx.send(f'🎶 شغّال الآن: **{title}**')

@bot.command(name='stop', help='إيقاف الأغنية وخروج البوت')
async def stop(ctx):
    if ctx.voice_client:
        await ctx.voice_client.disconnect()
        await ctx.send('⏹️ تم إيقاف الصوت وطلع البوت.')
    else:
        await ctx.send('❌ البوت مو موجود في أي قناة صوتية!')

# ضع توكن البوت الخاص بك هنا
bot.run('YOUR_BOT_TOKEN')
