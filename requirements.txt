import discord
from discord.ext import commands, tasks
import sqlite3
import asyncio
from datetime import datetime, timedelta
import pytz
import random

# ══════════════════════════════════════════
#  CONFIG — modifie ces valeurs !
# ══════════════════════════════════════════
TOKEN = "TOKEN"
PREFIX = "!"
STATS_CHANNEL_NAME = "stats"
TIMEZONE = "Europe/Paris"

PIECES_PAR_MESSAGE      = 1
PIECES_PAR_REACTION     = 2
PIECES_PAR_MINUTE_VOCAL = 3

BONUS_TOP_MESSAGES  = 200
BONUS_TOP_REACTIONS = 150
BONUS_TOP_VOCAL     = 150

PRIX_LOTO = 20  # prix d'un ticket de loto
# ══════════════════════════════════════════

intents = discord.Intents.default()
intents.message_content = True
intents.members = True
intents.voice_states = True
intents.reactions = True

bot = commands.Bot(command_prefix=PREFIX, intents=intents)

# ─── Base de données ───────────────────────────────────────────────────────────
def get_db():
    conn = sqlite3.connect("stats.db")
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_db()
    c = conn.cursor()
    c.execute("""CREATE TABLE IF NOT EXISTS weekly_stats (
        user_id INTEGER, guild_id INTEGER, messages INTEGER DEFAULT 0,
        reactions INTEGER DEFAULT 0, voice_secs INTEGER DEFAULT 0,
        week_start TEXT, PRIMARY KEY (user_id, guild_id, week_start))""")
    c.execute("""CREATE TABLE IF NOT EXISTS voice_sessions (
        user_id INTEGER, guild_id INTEGER, joined_at TEXT,
        PRIMARY KEY (user_id, guild_id))""")
    c.execute("""CREATE TABLE IF NOT EXISTS wallet (
        user_id INTEGER, guild_id INTEGER, pieces INTEGER DEFAULT 0,
        PRIMARY KEY (user_id, guild_id))""")
    c.execute("""CREATE TABLE IF NOT EXISTS loto_tickets (
        user_id INTEGER, guild_id INTEGER, week_start TEXT,
        PRIMARY KEY (user_id, guild_id, week_start))""")
    c.execute("""CREATE TABLE IF NOT EXISTS cooldowns (
        user_id INTEGER, guild_id INTEGER, action TEXT, last_used TEXT,
        PRIMARY KEY (user_id, guild_id, action))""")
    conn.commit()
    conn.close()

def get_week_start():
    today = datetime.now(pytz.timezone(TIMEZONE)).date()
    monday = today - timedelta(days=today.weekday())
    return monday.isoformat()

def upsert_stat(user_id, guild_id, field, amount=1):
    week = get_week_start()
    conn = get_db()
    conn.execute("""INSERT INTO weekly_stats (user_id, guild_id, messages, reactions, voice_secs, week_start)
        VALUES (?, ?, 0, 0, 0, ?) ON CONFLICT(user_id, guild_id, week_start) DO NOTHING""",
        (user_id, guild_id, week))
    conn.execute(f"UPDATE weekly_stats SET {field} = {field} + ? WHERE user_id=? AND guild_id=? AND week_start=?",
        (amount, user_id, guild_id, week))
    conn.commit()
    conn.close()

def add_pieces(user_id, guild_id, amount):
    conn = get_db()
    conn.execute("""INSERT INTO wallet (user_id, guild_id, pieces) VALUES (?, ?, ?)
        ON CONFLICT(user_id, guild_id) DO UPDATE SET pieces = pieces + ?""",
        (user_id, guild_id, amount, amount))
    conn.commit()
    conn.close()

def remove_pieces(user_id, guild_id, amount):
    conn = get_db()
    conn.execute("UPDATE wallet SET pieces = MAX(0, pieces - ?) WHERE user_id=? AND guild_id=?",
        (amount, user_id, guild_id))
    conn.commit()
    conn.close()

def get_pieces(user_id, guild_id):
    conn = get_db()
    row = conn.execute("SELECT pieces FROM wallet WHERE user_id=? AND guild_id=?", (user_id, guild_id)).fetchone()
    conn.close()
    return row["pieces"] if row else 0

def format_duration(seconds):
    h = seconds // 3600
    m = (seconds % 3600) // 60
    s = seconds % 60
    if h > 0: return f"{h}h {m}m"
    elif m > 0: return f"{m}m {s}s"
    else: return f"{s}s"

def check_cooldown(user_id, guild_id, action, seconds):
    """Retourne (ok, secondes_restantes)"""
    conn = get_db()
    row = conn.execute("SELECT last_used FROM cooldowns WHERE user_id=? AND guild_id=? AND action=?",
        (user_id, guild_id, action)).fetchone()
    conn.close()
    if row:
        last = datetime.fromisoformat(row["last_used"])
        diff = (datetime.utcnow() - last).total_seconds()
        if diff < seconds:
            return False, int(seconds - diff)
    return True, 0

def set_cooldown(user_id, guild_id, action):
    conn = get_db()
    conn.execute("""INSERT INTO cooldowns (user_id, guild_id, action, last_used) VALUES (?, ?, ?, ?)
        ON CONFLICT(user_id, guild_id, action) DO UPDATE SET last_used=?""",
        (user_id, guild_id, action, datetime.utcnow().isoformat(), datetime.utcnow().isoformat()))
    conn.commit()
    conn.close()

# ─── Vocal ─────────────────────────────────────────────────────────────────────
def save_voice_join(user_id, guild_id):
    conn = get_db()
    conn.execute("INSERT OR REPLACE INTO voice_sessions (user_id, guild_id, joined_at) VALUES (?, ?, ?)",
        (user_id, guild_id, datetime.utcnow().isoformat()))
    conn.commit()
    conn.close()

def flush_voice_time(user_id, guild_id):
    conn = get_db()
    row = conn.execute("SELECT joined_at FROM voice_sessions WHERE user_id=? AND guild_id=?", (user_id, guild_id)).fetchone()
    if row:
        seconds = int((datetime.utcnow() - datetime.fromisoformat(row["joined_at"])).total_seconds())
        minutes = seconds // 60
        conn.execute("DELETE FROM voice_sessions WHERE user_id=? AND guild_id=?", (user_id, guild_id))
        conn.commit()
        conn.close()
        upsert_stat(user_id, guild_id, "voice_secs", seconds)
        if minutes > 0:
            add_pieces(user_id, guild_id, minutes * PIECES_PAR_MINUTE_VOCAL)
    else:
        conn.close()

# ─── Événements ────────────────────────────────────────────────────────────────
@bot.event
async def on_ready():
    init_db()
    print(f"✅ Bot connecté en tant que {bot.user}")
    post_weekly_stats.start()
    for guild in bot.guilds:
        for vc in guild.voice_channels:
            for member in vc.members:
                if not member.bot:
                    save_voice_join(member.id, guild.id)

@bot.event
async def on_message(message):
    if message.author.bot: return
    upsert_stat(message.author.id, message.guild.id, "messages")
    add_pieces(message.author.id, message.guild.id, PIECES_PAR_MESSAGE)
    await bot.process_commands(message)

@bot.event
async def on_reaction_add(reaction, user):
    if user.bot: return
    upsert_stat(user.id, reaction.message.guild.id, "reactions")
    add_pieces(user.id, reaction.message.guild.id, PIECES_PAR_REACTION)

@bot.event
async def on_voice_state_update(member, before, after):
    if member.bot: return
    if before.channel is None and after.channel is not None:
        save_voice_join(member.id, member.guild.id)
    elif before.channel is not None and after.channel is None:
        flush_voice_time(member.id, member.guild.id)

# ─── Bilan hebdomadaire ────────────────────────────────────────────────────────
async def send_weekly_stats(guild):
    conn = get_db()
    sessions = conn.execute("SELECT user_id FROM voice_sessions WHERE guild_id=?", (guild.id,)).fetchall()
    conn.close()
    for s in sessions:
        flush_voice_time(s["user_id"], guild.id)
        save_voice_join(s["user_id"], guild.id)

    week = get_week_start()
    conn = get_db()
    rows = conn.execute("""SELECT user_id, messages, reactions, voice_secs FROM weekly_stats
        WHERE guild_id=? AND week_start=? ORDER BY messages DESC""", (guild.id, week)).fetchall()
    conn.close()
    if not rows: return

    channel = discord.utils.get(guild.text_channels, name=STATS_CHANNEL_NAME)
    if not channel:
        channel = await guild.create_text_channel(STATS_CHANNEL_NAME)

    medals = ["🥇", "🥈", "🥉"]
    embed = discord.Embed(title="📊 Bilan de la semaine", description=f"Semaine du **{week}**",
        color=0x5865F2, timestamp=datetime.utcnow())

    def make_top(sorted_rows, key, fmt=None):
        lines, winner = [], None
        for i, row in enumerate(sorted_rows[:3]):
            val = row[key]
            if val == 0: break
            member = guild.get_member(row["user_id"])
            name = member.display_name if member else f"User {row['user_id']}"
            display = fmt(val) if fmt else str(val)
            lines.append(f"{medals[i]} **{name}** — {display}")
            if i == 0: winner = row["user_id"]
        return "\n".join(lines) or "—", winner

    msg_text,   w_msg   = make_top(sorted(rows, key=lambda r: r["messages"],   reverse=True), "messages",   lambda v: f"{v} messages")
    react_text, w_react = make_top(sorted(rows, key=lambda r: r["reactions"],  reverse=True), "reactions",  lambda v: f"{v} réactions")
    voice_text, w_voice = make_top(sorted(rows, key=lambda r: r["voice_secs"], reverse=True), "voice_secs", format_duration)

    embed.add_field(name="💬 Top Messages",  value=msg_text,   inline=False)
    embed.add_field(name="⚡ Top Réactions", value=react_text, inline=False)
    embed.add_field(name="🎙️ Top Vocal",     value=voice_text, inline=False)

    bonus_lines = []
    for winner, bonus, label in [(w_msg, BONUS_TOP_MESSAGES, "💬"), (w_react, BONUS_TOP_REACTIONS, "⚡"), (w_voice, BONUS_TOP_VOCAL, "🎙️")]:
        if winner:
            add_pieces(winner, guild.id, bonus)
            m = guild.get_member(winner)
            bonus_lines.append(f"{label} **{m.display_name if m else winner}** +{bonus} 🪙")
    if bonus_lines:
        embed.add_field(name="🏆 Bonus pièces pour les top 1 !", value="\n".join(bonus_lines), inline=False)

    # Tirage loto
    conn = get_db()
    loto_rows = conn.execute("SELECT user_id FROM loto_tickets WHERE guild_id=? AND week_start=?", (guild.id, week)).fetchall()
    conn.close()
    if loto_rows:
        gagnant_id = random.choice(loto_rows)["user_id"]
        gain_loto = len(loto_rows) * PRIX_LOTO * 2
        add_pieces(gagnant_id, guild.id, gain_loto)
        m = guild.get_member(gagnant_id)
        name = m.display_name if m else str(gagnant_id)
        embed.add_field(
            name=f"🎟️ Tirage du Loto ! ({len(loto_rows)} tickets)",
            value=f"🎉 **{name}** gagne le jackpot de **{gain_loto}** 🪙 !",
            inline=False
        )

    embed.set_footer(text="Bilan auto chaque dimanche à 23h00")
    await channel.send(embed=embed)

@tasks.loop(minutes=1)
async def post_weekly_stats():
    tz = pytz.timezone(TIMEZONE)
    now = datetime.now(tz)
    if now.weekday() == 6 and now.hour == 23 and now.minute == 0:
        for guild in bot.guilds:
            await send_weekly_stats(guild)

# ─── Stats ─────────────────────────────────────────────────────────────────────
@bot.command(name="stats")
async def stats_cmd(ctx, member: discord.Member = None):
    target = member or ctx.author
    week = get_week_start()
    conn = get_db()
    row = conn.execute("SELECT messages, reactions, voice_secs FROM weekly_stats WHERE user_id=? AND guild_id=? AND week_start=?",
        (target.id, ctx.guild.id, week)).fetchone()
    conn.close()
    pieces = get_pieces(target.id, ctx.guild.id)
    embed = discord.Embed(title=f"📊 Stats de {target.display_name}", description=f"Semaine du {week}", color=target.color)
    if row:
        embed.add_field(name="💬 Messages",  value=str(row["messages"]), inline=True)
        embed.add_field(name="⚡ Réactions", value=str(row["reactions"]), inline=True)
        embed.add_field(name="🎙️ Vocal",     value=format_duration(row["voice_secs"]), inline=True)
    else:
        embed.add_field(name="Activité", value="Aucune activité cette semaine.", inline=False)
    embed.add_field(name="🪙 Pièces", value=str(pieces), inline=False)
    embed.set_thumbnail(url=target.display_avatar.url)
    await ctx.send(embed=embed)

@bot.command(name="top")
async def top_cmd(ctx):
    await send_weekly_stats(ctx.guild)

@bot.command(name="bilanforce")
@commands.has_permissions(administrator=True)
async def bilan_force(ctx):
    await send_weekly_stats(ctx.guild)
    await ctx.message.add_reaction("✅")

# ─── Monnaie ───────────────────────────────────────────────────────────────────
@bot.command(name="solde")
async def solde_cmd(ctx, member: discord.Member = None):
    target = member or ctx.author
    pieces = get_pieces(target.id, ctx.guild.id)
    embed = discord.Embed(title=f"🪙 Solde de {target.display_name}", description=f"**{pieces}** pièces", color=0xFFD700)
    embed.set_thumbnail(url=target.display_avatar.url)
    await ctx.send(embed=embed)

@bot.command(name="classement")
async def classement_cmd(ctx):
    conn = get_db()
    rows = conn.execute("SELECT user_id, pieces FROM wallet WHERE guild_id=? ORDER BY pieces DESC LIMIT 10", (ctx.guild.id,)).fetchall()
    conn.close()
    medals = ["🥇", "🥈", "🥉"]
    lines = []
    for i, row in enumerate(rows):
        m = ctx.guild.get_member(row["user_id"])
        name = m.display_name if m else f"User {row['user_id']}"
        lines.append(f"{medals[i] if i < 3 else f'#{i+1}'} **{name}** — {row['pieces']} 🪙")
    embed = discord.Embed(title="🏦 Classement des richesses", color=0xFFD700)
    embed.description = "\n".join(lines) if lines else "Personne n'a encore de pièces !"
    await ctx.send(embed=embed)

@bot.command(name="donner")
async def donner_cmd(ctx, member: discord.Member, amount: int):
    if amount <= 0: return await ctx.send("❌ Le montant doit être positif !")
    if member.bot: return await ctx.send("❌ Tu peux pas donner des pièces à un bot !")
    solde = get_pieces(ctx.author.id, ctx.guild.id)
    if solde < amount: return await ctx.send(f"❌ T'as pas assez de pièces ! (Solde : {solde} 🪙)")
    remove_pieces(ctx.author.id, ctx.guild.id, amount)
    add_pieces(member.id, ctx.guild.id, amount)
    await ctx.send(f"✅ **{ctx.author.display_name}** a donné **{amount}** 🪙 à **{member.display_name}** !")

# ══════════════════════════════════════════
#  MINI-JEUX
# ══════════════════════════════════════════

# 🎲 Dés
@bot.command(name="dice")
async def dice_cmd(ctx, member: discord.Member, mise: int):
    if mise <= 0: return await ctx.send("❌ La mise doit être positive !")
    if member.bot or member == ctx.author: return await ctx.send("❌ Choisis un vrai membre !")
    if get_pieces(ctx.author.id, ctx.guild.id) < mise: return await ctx.send("❌ T'as pas assez de pièces !")
    if get_pieces(member.id, ctx.guild.id) < mise: return await ctx.send(f"❌ **{member.display_name}** n'a pas assez de pièces !")
    de1, de2 = random.randint(1, 6), random.randint(1, 6)
    embed = discord.Embed(title="🎲 Duel de dés !", color=0x5865F2)
    embed.add_field(name=ctx.author.display_name, value=f"🎲 {de1}", inline=True)
    embed.add_field(name=member.display_name,     value=f"🎲 {de2}", inline=True)
    if de1 > de2:
        remove_pieces(member.id, ctx.guild.id, mise); add_pieces(ctx.author.id, ctx.guild.id, mise)
        embed.description = f"🏆 **{ctx.author.display_name}** gagne **{mise}** 🪙 !"; embed.color = 0x00FF00
    elif de2 > de1:
        remove_pieces(ctx.author.id, ctx.guild.id, mise); add_pieces(member.id, ctx.guild.id, mise)
        embed.description = f"🏆 **{member.display_name}** gagne **{mise}** 🪙 !"; embed.color = 0xFF0000
    else:
        embed.description = "🤝 Égalité ! Personne ne gagne."
    await ctx.send(embed=embed)

# 🎰 Slots
@bot.command(name="slots")
async def slots_cmd(ctx, mise: int):
    if mise <= 0: return await ctx.send("❌ La mise doit être positive !")
    if get_pieces(ctx.author.id, ctx.guild.id) < mise: return await ctx.send("❌ T'as pas assez de pièces !")
    symboles = ["🍒", "🍋", "🍊", "⭐", "💎", "7️⃣"]
    r = [random.choice(symboles) for _ in range(3)]
    remove_pieces(ctx.author.id, ctx.guild.id, mise)
    if r[0] == r[1] == r[2]:
        gain = mise * (10 if r[0] == "💎" else 5 if r[0] == "7️⃣" else 3)
        label = "💎 JACKPOT" if r[0] == "💎" else "7️⃣ SUPER WIN" if r[0] == "7️⃣" else "✅ Triple"
        add_pieces(ctx.author.id, ctx.guild.id, gain); resultat = f"{label} ! Tu gagnes **{gain}** 🪙 !"
    elif r[0] == r[1] or r[1] == r[2]:
        add_pieces(ctx.author.id, ctx.guild.id, mise); resultat = f"👍 Double ! Tu récupères ta mise **{mise}** 🪙."
    else:
        resultat = f"❌ Perdu ! Tu perds **{mise}** 🪙."
    embed = discord.Embed(title="🎰 Machine à sous", description=f"[ {r[0]} | {r[1]} | {r[2]} ]\n\n{resultat}", color=0xFFD700)
    embed.set_footer(text=f"Solde : {get_pieces(ctx.author.id, ctx.guild.id)} 🪙")
    await ctx.send(embed=embed)

# 🪙 Pile ou Face
@bot.command(name="pileouface")
async def pof_cmd(ctx, choix: str, mise: int):
    choix = choix.lower()
    if choix not in ["pile", "face"]: return await ctx.send("❌ Choisis `pile` ou `face` !")
    if mise <= 0: return await ctx.send("❌ La mise doit être positive !")
    if get_pieces(ctx.author.id, ctx.guild.id) < mise: return await ctx.send("❌ T'as pas assez de pièces !")
    resultat = random.choice(["pile", "face"])
    if choix == resultat:
        add_pieces(ctx.author.id, ctx.guild.id, mise); msg = f"🪙 C'est **{resultat}** ! Tu gagnes **{mise}** 🪙 !"; color = 0x00FF00
    else:
        remove_pieces(ctx.author.id, ctx.guild.id, mise); msg = f"🪙 C'est **{resultat}** ! Tu perds **{mise}** 🪙..."; color = 0xFF0000
    embed = discord.Embed(title="🪙 Pile ou Face", description=msg, color=color)
    embed.set_footer(text=f"Solde : {get_pieces(ctx.author.id, ctx.guild.id)} 🪙")
    await ctx.send(embed=embed)

# ❓ Quiz
QUESTIONS = [
    {"q": "Quelle est la capitale de la France ?",               "r": "paris"},
    {"q": "Combien font 7 x 8 ?",                                "r": "56"},
    {"q": "Quel est le plus grand océan du monde ?",             "r": "pacifique"},
    {"q": "En quelle année a eu lieu la Révolution française ?", "r": "1789"},
    {"q": "Quelle planète est la plus proche du soleil ?",       "r": "mercure"},
    {"q": "Combien de côtés a un hexagone ?",                    "r": "6"},
    {"q": "Quel animal est le symbole de l'Australie ?",         "r": "kangourou"},
    {"q": "Quelle est la langue la plus parlée au monde ?",      "r": "mandarin"},
    {"q": "Combien de joueurs dans une équipe de foot ?",        "r": "11"},
    {"q": "Quel est le pays le plus grand du monde ?",           "r": "russie"},
    {"q": "Combien de continents y a-t-il sur Terre ?",          "r": "7"},
    {"q": "Quelle est la capitale de l'Espagne ?",               "r": "madrid"},
    {"q": "Combien font 12 x 12 ?",                              "r": "144"},
    {"q": "Quel est l'animal le plus rapide du monde ?",         "r": "guepard"},
    {"q": "Dans quel pays se trouve la Tour de Pise ?",          "r": "italie"},
]
quiz_en_cours = {}

@bot.command(name="quiz")
async def quiz_cmd(ctx):
    if ctx.channel.id in quiz_en_cours: return await ctx.send("❌ Un quiz est déjà en cours !")
    question = random.choice(QUESTIONS)
    quiz_en_cours[ctx.channel.id] = question["r"]
    gain = 50
    embed = discord.Embed(title="❓ Quiz !", description=f"**{question['q']}**\n\nSois le premier à répondre et gagne **{gain}** 🪙 !\n*30 secondes...*", color=0x5865F2)
    await ctx.send(embed=embed)
    def check(m): return m.channel == ctx.channel and not m.author.bot and m.content.lower().strip() == quiz_en_cours.get(ctx.channel.id, "")
    try:
        msg = await bot.wait_for("message", check=check, timeout=30)
        del quiz_en_cours[ctx.channel.id]
        add_pieces(msg.author.id, ctx.guild.id, gain)
        await ctx.send(f"🎉 Bravo **{msg.author.display_name}** ! Bonne réponse ! +**{gain}** 🪙")
    except asyncio.TimeoutError:
        quiz_en_cours.pop(ctx.channel.id, None)
        await ctx.send(f"⏰ Temps écoulé ! La réponse était : **{question['r']}**")

# 🧮 Calcul
calcul_en_cours = {}

@bot.command(name="calcul")
async def calcul_cmd(ctx):
    if ctx.channel.id in calcul_en_cours: return await ctx.send("❌ Un calcul est déjà en cours !")
    a, b = random.randint(1, 50), random.randint(1, 50)
    op = random.choice(["+", "-", "x"])
    if op == "+": reponse = a + b
    elif op == "-": reponse = a - b
    else: reponse = a * b
    calcul_en_cours[ctx.channel.id] = str(reponse)
    gain = 30
    embed = discord.Embed(title="🧮 Calcul rapide !", description=f"**{a} {op} {b} = ?**\n\nSois le premier à répondre et gagne **{gain}** 🪙 !\n*20 secondes...*", color=0x00BFFF)
    await ctx.send(embed=embed)
    def check(m): return m.channel == ctx.channel and not m.author.bot and m.content.strip() == calcul_en_cours.get(ctx.channel.id, "")
    try:
        msg = await bot.wait_for("message", check=check, timeout=20)
        del calcul_en_cours[ctx.channel.id]
        add_pieces(msg.author.id, ctx.guild.id, gain)
        await ctx.send(f"🎉 Bravo **{msg.author.display_name}** ! +**{gain}** 🪙")
    except asyncio.TimeoutError:
        calcul_en_cours.pop(ctx.channel.id, None)
        await ctx.send(f"⏰ Temps écoulé ! La réponse était : **{reponse}**")

# 🎯 Devinette
DEVINETTES = [
    {"q": "J'ai des dents mais je ne mords pas. Qu'est-ce que je suis ?",       "r": "peigne"},
    {"q": "Plus je sèche, plus je suis mouillée. Qu'est-ce que je suis ?",      "r": "serviette"},
    {"q": "Je parle sans bouche et j'entends sans oreilles. Qu'est-ce que je suis ?", "r": "echo"},
    {"q": "On me jette quand on m'utilise, on me garde quand on ne m'utilise pas. Qu'est-ce que je suis ?", "r": "ancre"},
    {"q": "Je suis toujours devant toi mais ne peut jamais être vu. Qu'est-ce que je suis ?", "r": "futur"},
    {"q": "Plus je suis grande, moins on me voit. Qu'est-ce que je suis ?",     "r": "obscurite"},
    {"q": "J'ai une tête et une queue mais pas de corps. Qu'est-ce que je suis ?", "r": "piece"},
    {"q": "Je vole sans ailes et pleure sans yeux. Qu'est-ce que je suis ?",    "r": "nuage"},
    {"q": "Je monte et descends sans bouger. Qu'est-ce que je suis ?",          "r": "escalier"},
    {"q": "On me coupe mais je ne saigne pas. On me lit mais je ne suis pas un livre. Qu'est-ce que je suis ?", "r": "journal"},
]
devinette_en_cours = {}

@bot.command(name="devinette")
async def devinette_cmd(ctx):
    if ctx.channel.id in devinette_en_cours: return await ctx.send("❌ Une devinette est déjà en cours !")
    d = random.choice(DEVINETTES)
    devinette_en_cours[ctx.channel.id] = d["r"]
    gain = 40
    embed = discord.Embed(title="🎯 Devinette !", description=f"**{d['q']}**\n\nSois le premier à trouver et gagne **{gain}** 🪙 !\n*60 secondes...*", color=0x9B59B6)
    await ctx.send(embed=embed)
    def check(m): return m.channel == ctx.channel and not m.author.bot and m.content.lower().strip() == devinette_en_cours.get(ctx.channel.id, "")
    try:
        msg = await bot.wait_for("message", check=check, timeout=60)
        del devinette_en_cours[ctx.channel.id]
        add_pieces(msg.author.id, ctx.guild.id, gain)
        await ctx.send(f"🎉 Bravo **{msg.author.display_name}** ! Bonne réponse ! +**{gain}** 🪙")
    except asyncio.TimeoutError:
        devinette_en_cours.pop(ctx.channel.id, None)
        await ctx.send(f"⏰ Temps écoulé ! La réponse était : **{d['r']}**")

# 🎟️ Loto
@bot.command(name="loto")
async def loto_cmd(ctx):
    week = get_week_start()
    conn = get_db()
    existing = conn.execute("SELECT 1 FROM loto_tickets WHERE user_id=? AND guild_id=? AND week_start=?",
        (ctx.author.id, ctx.guild.id, week)).fetchone()
    conn.close()
    if existing: return await ctx.send("❌ T'as déjà un ticket cette semaine ! Le tirage a lieu dimanche à 23h.")
    if get_pieces(ctx.author.id, ctx.guild.id) < PRIX_LOTO:
        return await ctx.send(f"❌ T'as pas assez de pièces ! Un ticket coûte **{PRIX_LOTO}** 🪙.")
    remove_pieces(ctx.author.id, ctx.guild.id, PRIX_LOTO)
    conn = get_db()
    conn.execute("INSERT INTO loto_tickets (user_id, guild_id, week_start) VALUES (?, ?, ?)", (ctx.author.id, ctx.guild.id, week))
    conn.commit()
    nb = conn.execute("SELECT COUNT(*) as n FROM loto_tickets WHERE guild_id=? AND week_start=?", (ctx.guild.id, week)).fetchone()["n"]
    conn.close()
    jackpot = nb * PRIX_LOTO * 2
    embed = discord.Embed(title="🎟️ Ticket de Loto acheté !", color=0xFFD700,
        description=f"**{ctx.author.display_name}** a acheté un ticket pour **{PRIX_LOTO}** 🪙 !\n\n🏆 Jackpot actuel : **{jackpot}** 🪙\n👥 Participants : **{nb}**\n⏰ Tirage : dimanche à 23h")
    await ctx.send(embed=embed)

# 🎰 Roulette
@bot.command(name="roulette")
async def roulette_cmd(ctx, couleur: str, mise: int):
    couleur = couleur.lower()
    if couleur not in ["rouge", "noir", "vert"]:
        return await ctx.send("❌ Choisis `rouge`, `noir` ou `vert` !\n*Vert = x14 mais très rare !*")
    if mise <= 0: return await ctx.send("❌ La mise doit être positive !")
    if get_pieces(ctx.author.id, ctx.guild.id) < mise: return await ctx.send("❌ T'as pas assez de pièces !")

    # 18 rouges, 18 noirs, 1 vert (0)
    tirage = random.choices(["rouge", "noir", "vert"], weights=[18, 18, 1])[0]
    emojis = {"rouge": "🔴", "noir": "⚫", "vert": "🟢"}
    remove_pieces(ctx.author.id, ctx.guild.id, mise)

    if couleur == tirage:
        gain = mise * (14 if tirage == "vert" else 2)
        add_pieces(ctx.author.id, ctx.guild.id, gain)
        msg = f"La bille s'arrête sur {emojis[tirage]} **{tirage}** !\n🏆 Tu gagnes **{gain}** 🪙 !"
        color = 0x00FF00
    else:
        msg = f"La bille s'arrête sur {emojis[tirage]} **{tirage}** !\n❌ Tu perds **{mise}** 🪙..."
        color = 0xFF0000

    embed = discord.Embed(title="🎡 Roulette", description=msg, color=color)
    embed.set_footer(text=f"Solde : {get_pieces(ctx.author.id, ctx.guild.id)} 🪙")
    await ctx.send(embed=embed)

# 🃏 Blackjack
def valeur_carte(carte):
    if carte in ["J", "Q", "K"]: return 10
    if carte == "A": return 11
    return int(carte)

def generer_main():
    cartes = ["2","3","4","5","6","7","8","9","10","J","Q","K","A"]
    return [random.choice(cartes), random.choice(cartes)]

def total_main(main):
    total = sum(valeur_carte(c) for c in main)
    if total > 21 and "A" in main:
        total -= 10
    return total

def afficher_main(main):
    return " | ".join(main)

blackjack_en_cours = {}

@bot.command(name="blackjack", aliases=["bj"])
async def blackjack_cmd(ctx, mise: int):
    if ctx.author.id in blackjack_en_cours: return await ctx.send("❌ T'as déjà une partie en cours !")
    if mise <= 0: return await ctx.send("❌ La mise doit être positive !")
    if get_pieces(ctx.author.id, ctx.guild.id) < mise: return await ctx.send("❌ T'as pas assez de pièces !")

    remove_pieces(ctx.author.id, ctx.guild.id, mise)
    main_joueur = generer_main()
    main_bot = generer_main()
    blackjack_en_cours[ctx.author.id] = {"main": main_joueur, "bot": main_bot, "mise": mise, "guild": ctx.guild.id}

    embed = discord.Embed(title="🃏 Blackjack", color=0x2ECC71)
    embed.add_field(name="Ta main", value=f"{afficher_main(main_joueur)} (total : **{total_main(main_joueur)}**)", inline=False)
    embed.add_field(name="Main du bot", value=f"{main_bot[0]} | ❓", inline=False)
    embed.add_field(name="Actions", value="Tape `tirer` pour une carte ou `rester` pour s'arrêter", inline=False)

    if total_main(main_joueur) == 21:
        gain = int(mise * 2.5)
        add_pieces(ctx.author.id, ctx.guild.id, gain)
        del blackjack_en_cours[ctx.author.id]
        embed.description = f"🎉 **BLACKJACK !** Tu gagnes **{gain}** 🪙 !"
        embed.color = 0xFFD700
        return await ctx.send(embed=embed)

    await ctx.send(embed=embed)

    def check(m): return m.author == ctx.author and m.channel == ctx.channel and m.content.lower() in ["tirer", "rester"]

    while ctx.author.id in blackjack_en_cours:
        try:
            msg = await bot.wait_for("message", check=check, timeout=30)
        except asyncio.TimeoutError:
            blackjack_en_cours.pop(ctx.author.id, None)
            return await ctx.send(f"⏰ Temps écoulé ! Tu perds ta mise de **{mise}** 🪙.")

        data = blackjack_en_cours.get(ctx.author.id)
        if not data: break

        if msg.content.lower() == "tirer":
            data["main"].append(random.choice(["2","3","4","5","6","7","8","9","10","J","Q","K","A"]))
            total = total_main(data["main"])
            if total > 21:
                del blackjack_en_cours[ctx.author.id]
                embed2 = discord.Embed(title="🃏 Blackjack", color=0xFF0000,
                    description=f"Ta main : {afficher_main(data['main'])} (total : **{total}**)\n\n💥 Bust ! Tu perds **{mise}** 🪙.")
                return await ctx.send(embed=embed2)
            embed2 = discord.Embed(title="🃏 Blackjack", color=0x2ECC71)
            embed2.add_field(name="Ta main", value=f"{afficher_main(data['main'])} (total : **{total}**)", inline=False)
            embed2.add_field(name="Actions", value="Tape `tirer` ou `rester`", inline=False)
            await ctx.send(embed=embed2)
        else:  # rester
            del blackjack_en_cours[ctx.author.id]
            total_j = total_main(data["main"])
            total_b = total_main(data["bot"])
            # Le bot tire jusqu'à 17
            while total_b < 17:
                data["bot"].append(random.choice(["2","3","4","5","6","7","8","9","10","J","Q","K","A"]))
                total_b = total_main(data["bot"])

            embed2 = discord.Embed(title="🃏 Blackjack — Résultat", color=0x5865F2)
            embed2.add_field(name="Ta main", value=f"{afficher_main(data['main'])} (total : **{total_j}**)", inline=True)
            embed2.add_field(name="Main du bot", value=f"{afficher_main(data['bot'])} (total : **{total_b}**)", inline=True)

            if total_b > 21 or total_j > total_b:
                gain = mise * 2
                add_pieces(ctx.author.id, ctx.guild.id, gain)
                embed2.description = f"🏆 Tu gagnes ! +**{gain}** 🪙"; embed2.color = 0x00FF00
            elif total_j == total_b:
                add_pieces(ctx.author.id, ctx.guild.id, mise)
                embed2.description = f"🤝 Égalité ! Tu récupères ta mise **{mise}** 🪙."
            else:
                embed2.description = f"❌ Le bot gagne ! Tu perds **{mise}** 🪙."; embed2.color = 0xFF0000

            embed2.set_footer(text=f"Solde : {get_pieces(ctx.author.id, ctx.guild.id)} 🪙")
            await ctx.send(embed=embed2)
            break

# ⚔️ Duel RPG
duel_en_attente = {}

@bot.command(name="duel")
async def duel_cmd(ctx, member: discord.Member, mise: int):
    if member.bot or member == ctx.author: return await ctx.send("❌ Choisis un vrai membre !")
    if mise <= 0: return await ctx.send("❌ La mise doit être positive !")
    if get_pieces(ctx.author.id, ctx.guild.id) < mise: return await ctx.send("❌ T'as pas assez de pièces !")
    if get_pieces(member.id, ctx.guild.id) < mise: return await ctx.send(f"❌ **{member.display_name}** n'a pas assez de pièces !")

    embed = discord.Embed(title="⚔️ Défi de duel !",
        description=f"**{ctx.author.display_name}** défie **{member.mention}** pour **{mise}** 🪙 !\n\nTape `accepter` pour relever le défi ou `refuser` pour décliner.\n*30 secondes...*",
        color=0xFF6B00)
    await ctx.send(embed=embed)

    def check(m): return m.author == member and m.channel == ctx.channel and m.content.lower() in ["accepter", "refuser"]
    try:
        msg = await bot.wait_for("message", check=check, timeout=30)
    except asyncio.TimeoutError:
        return await ctx.send(f"⏰ **{member.display_name}** n'a pas répondu. Duel annulé.")

    if msg.content.lower() == "refuser":
        return await ctx.send(f"🏳️ **{member.display_name}** a refusé le duel.")

    # Combat RPG
    pv1, pv2 = 100, 100
    log = []
    attaquant, defenseur = ctx.author, member
    pv_att, pv_def = pv1, pv2

    for round_num in range(1, 11):
        degats = random.randint(10, 30)
        esquive = random.random() < 0.2
        if esquive:
            log.append(f"Round {round_num}: **{defenseur.display_name}** esquive l'attaque !")
        else:
            pv_def -= degats
            log.append(f"Round {round_num}: **{attaquant.display_name}** inflige **{degats}** dégâts → **{defenseur.display_name}** ({max(0,pv_def)} PV)")
        if pv_def <= 0: break
        attaquant, defenseur = defenseur, attaquant
        pv_att, pv_def = pv_def, pv_att

    gagnant = ctx.author if pv_def <= 0 and defenseur == member else member
    perdant = member if gagnant == ctx.author else ctx.author

    remove_pieces(perdant.id, ctx.guild.id, mise)
    add_pieces(gagnant.id, ctx.guild.id, mise)

    embed2 = discord.Embed(title="⚔️ Résultat du duel !", color=0xFF6B00,
        description="\n".join(log[-5:]) + f"\n\n🏆 **{gagnant.display_name}** remporte **{mise}** 🪙 !")
    await ctx.send(embed=embed2)

# 🔫 Braquage
@bot.command(name="braquage")
async def braquage_cmd(ctx, member: discord.Member):
    if member.bot or member == ctx.author: return await ctx.send("❌ Choisis un vrai membre !")

    ok, reste = check_cooldown(ctx.author.id, ctx.guild.id, "braquage", 3600)
    if not ok: return await ctx.send(f"❌ T'es encore en cooldown ! Attends encore **{reste//60}min {reste%60}s**.")

    solde_victime = get_pieces(member.id, ctx.guild.id)
    if solde_victime < 50: return await ctx.send(f"❌ **{member.display_name}** n'a pas assez de pièces à voler (min 50 🪙) !")

    set_cooldown(ctx.author.id, ctx.guild.id, "braquage")
    succes = random.random() < 0.4  # 40% de chance de réussir

    if succes:
        vol = random.randint(20, min(200, solde_victime // 2))
        remove_pieces(member.id, ctx.guild.id, vol)
        add_pieces(ctx.author.id, ctx.guild.id, vol)
        embed = discord.Embed(title="🔫 Braquage réussi !",
            description=f"**{ctx.author.display_name}** a volé **{vol}** 🪙 à **{member.display_name}** ! 💰",
            color=0x00FF00)
    else:
        penalite = random.randint(30, 100)
        remove_pieces(ctx.author.id, ctx.guild.id, penalite)
        embed = discord.Embed(title="🚔 Braquage raté !",
            description=f"**{ctx.author.display_name}** s'est fait attraper et paie une amende de **{penalite}** 🪙 ! 🚓",
            color=0xFF0000)

    embed.set_footer(text=f"Solde : {get_pieces(ctx.author.id, ctx.guild.id)} 🪙 | Cooldown : 1h")
    await ctx.send(embed=embed)

# 🃏 Poker (simplifié — duel de mains)
VALEURS_POKER = ["2","3","4","5","6","7","8","9","10","J","Q","K","A"]
COULEURS_POKER = ["♠","♥","♦","♣"]

def generer_main_poker():
    return [(random.choice(VALEURS_POKER), random.choice(COULEURS_POKER)) for _ in range(5)]

def score_main_poker(main):
    valeurs = [v for v, c in main]
    couleurs = [c for v, c in main]
    indices = sorted([VALEURS_POKER.index(v) for v in valeurs], reverse=True)
    flush = len(set(couleurs)) == 1
    suite = (max(indices) - min(indices) == 4 and len(set(indices)) == 5)
    counts = {v: valeurs.count(v) for v in set(valeurs)}
    combos = sorted(counts.values(), reverse=True)

    if flush and suite: return 8, "Quinte Flush"
    if combos == [4, 1]: return 7, "Carré"
    if combos == [3, 2]: return 6, "Full House"
    if flush: return 5, "Couleur"
    if suite: return 4, "Suite"
    if combos == [3, 1, 1]: return 3, "Brelan"
    if combos == [2, 2, 1]: return 2, "Double Paire"
    if combos == [2, 1, 1, 1]: return 1, "Paire"
    return 0, "Carte Haute"

def afficher_main_poker(main):
    return " ".join(f"{v}{c}" for v, c in main)

@bot.command(name="poker")
async def poker_cmd(ctx, member: discord.Member, mise: int):
    if member.bot or member == ctx.author: return await ctx.send("❌ Choisis un vrai membre !")
    if mise <= 0: return await ctx.send("❌ La mise doit être positive !")
    if get_pieces(ctx.author.id, ctx.guild.id) < mise: return await ctx.send("❌ T'as pas assez de pièces !")
    if get_pieces(member.id, ctx.guild.id) < mise: return await ctx.send(f"❌ **{member.display_name}** n'a pas assez de pièces !")

    main1 = generer_main_poker()
    main2 = generer_main_poker()
    score1, combo1 = score_main_poker(main1)
    score2, combo2 = score_main_poker(main2)

    embed = discord.Embed(title="🃏 Poker — Duel de mains !", color=0x2ECC71)
    embed.add_field(name=f"{ctx.author.display_name}", value=f"{afficher_main_poker(main1)}\n**{combo1}**", inline=True)
    embed.add_field(name=f"{member.display_name}", value=f"{afficher_main_poker(main2)}\n**{combo2}**", inline=True)

    if score1 > score2:
        remove_pieces(member.id, ctx.guild.id, mise)
        add_pieces(ctx.author.id, ctx.guild.id, mise)
        embed.description = f"🏆 **{ctx.author.display_name}** gagne **{mise}** 🪙 !"
        embed.color = 0x00FF00
    elif score2 > score1:
        remove_pieces(ctx.author.id, ctx.guild.id, mise)
        add_pieces(member.id, ctx.guild.id, mise)
        embed.description = f"🏆 **{member.display_name}** gagne **{mise}** 🪙 !"
        embed.color = 0xFF0000
    else:
        embed.description = "🤝 Égalité ! Personne ne gagne."

    await ctx.send(embed=embed)

# 🎟️ Grattage
@bot.command(name="grattage")
async def grattage_cmd(ctx, mise: int):
    if mise <= 0: return await ctx.send("❌ La mise doit être positive !")
    if get_pieces(ctx.author.id, ctx.guild.id) < mise: return await ctx.send("❌ T'as pas assez de pièces !")
    remove_pieces(ctx.author.id, ctx.guild.id, mise)

    symboles = ["🍒", "💎", "⭐", "🍋", "7️⃣", "🔔"]
    grille = [[random.choice(symboles) for _ in range(3)] for _ in range(3)]

    # Vérifier les lignes gagnantes
    gains = 0
    lignes_gagnantes = []
    for i, ligne in enumerate(grille):
        if ligne[0] == ligne[1] == ligne[2]:
            if ligne[0] == "💎": mult = 10
            elif ligne[0] == "7️⃣": mult = 7
            elif ligne[0] == "⭐": mult = 5
            else: mult = 3
            gains += mise * mult
            lignes_gagnantes.append(i)

    if gains > 0:
        add_pieces(ctx.author.id, ctx.guild.id, gains)

    affichage = ""
    for i, ligne in enumerate(grille):
        row = " | ".join(ligne)
        affichage += f"{row} {'✅' if i in lignes_gagnantes else ''}\n"

    embed = discord.Embed(title="🎟️ Ticket Grattage", color=0xFFD700 if gains > 0 else 0x888888)
    embed.description = f"```\n{affichage}```"
    if gains > 0:
        embed.add_field(name="🎉 Gagné !", value=f"Tu remportes **{gains}** 🪙 !")
    else:
        embed.add_field(name="😢 Perdu !", value=f"Tu perds **{mise}** 🪙.")
    embed.set_footer(text=f"Solde : {get_pieces(ctx.author.id, ctx.guild.id)} 🪙")
    await ctx.send(embed=embed)

# ─── Aide ──────────────────────────────────────────────────────────────────────
@bot.command(name="aide")
async def aide_cmd(ctx):
    embed = discord.Embed(title="📖 Commandes du bot", color=0x5865F2)
    embed.add_field(name="📊 Stats", value="`!stats [@membre]`\n`!top`\n`!bilanforce` (admin)", inline=False)
    embed.add_field(name="🪙 Monnaie", value="`!solde [@membre]`\n`!classement`\n`!donner @membre montant`", inline=False)
    embed.add_field(name="🎮 Mini-jeux solo", value=(
        "`!slots mise` — Machine à sous\n"
        "`!pileouface pile/face mise` — Double ou rien\n"
        "`!roulette rouge/noir/vert mise` — Roulette\n"
        "`!blackjack mise` — Blackjack contre le bot\n"
        "`!grattage mise` — Ticket grattage\n"
        "`!loto` — Ticket loto (tirage dimanche)\n"
        "`!quiz` — Question culture générale\n"
        "`!calcul` — Calcul rapide\n"
        "`!devinette` — Devinette"
    ), inline=False)
    embed.add_field(name="⚔️ Mini-jeux multijoueur", value=(
        "`!dice @membre mise` — Duel de dés\n"
        "`!duel @membre mise` — Combat RPG\n"
        "`!poker @membre mise` — Poker\n"
        "`!braquage @membre` — Voler des pièces (cooldown 1h)"
    ), inline=False)
    embed.add_field(name="💰 Gains automatiques", value=(
        f"+{PIECES_PAR_MESSAGE} 🪙 par message\n"
        f"+{PIECES_PAR_REACTION} 🪙 par réaction\n"
        f"+{PIECES_PAR_MINUTE_VOCAL} 🪙 par minute vocal\n"
        f"+{BONUS_TOP_MESSAGES}/{BONUS_TOP_REACTIONS}/{BONUS_TOP_VOCAL} 🪙 bonus top semaine"
    ), inline=False)
    await ctx.send(embed=embed)

# ─── Lancement ─────────────────────────────────────────────────────────────────
bot.run(TOKEN)
