import discord
from discord.ext import commands
import datetime
import asyncio

# --- 設定區 ---
TOKEN = 'MTQ5Mzk5NjY0MDU1MjQ4NTAwNQ.Gd8oml.iLGV0dPgblRcpuSgvnb-iNRovJj3m9QFZhjVD0' 
WORKER_ROLE_ID = 1494029120982618112
LOG_CHANNEL_ID = 1494030234599686174
CATEGORY_ID = 1493235782188798102
MAX_WORKERS = 3

class NUNBot(commands.Bot):
    def __init__(self):
        intents = discord.Intents.default()
        intents.members = True
        intents.message_content = True
        super().__init__(command_prefix="!", intents=intents, help_command=None)

    async def setup_hook(self):
        # 註冊持久化選單，確保機器人重啟後舊訊息依然有效
        self.add_view(PersistentOrderMenu())

# --- 1. 結單視圖 ---
class CloseChannelView(discord.ui.View):
    def __init__(self, customer_id, worker_ids, order_type, start_time):
        super().__init__(timeout=None)
        self.customer_id = customer_id
        self.worker_ids = worker_ids  # 這是一個 ID 列表
        self.order_type = order_type
        self.start_time = start_time

    @discord.ui.button(label="完成訂單並結單", style=discord.ButtonStyle.danger, emoji="⛔", custom_id="nun:close_btn")
    async def close_order(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.defer()

        if interaction.user.id not in self.worker_ids and not interaction.user.guild_permissions.administrator:
            return await interaction.followup.send("❌ 只有參與此單的打手可以結單！", ephemeral=True)

        log_channel = interaction.guild.get_channel(LOG_CHANNEL_ID)
        duration = datetime.datetime.now() - self.start_time
        workers_mention = " ".join([f"<@{wid}>" for wid in self.worker_ids])

        log_embed = discord.Embed(title="📕 NUN 結單紀錄", color=discord.Color.red(), timestamp=datetime.datetime.now())
        log_embed.add_field(name="📋 項目", value=f"**{self.order_type}**", inline=False)
        log_embed.add_field(name="👤 顧客", value=f"<@{self.customer_id}>", inline=True)
        log_embed.add_field(name="⚔️ 參與打手", value=workers_mention, inline=False)
        log_embed.add_field(name="⏱️ 總耗時", value=f"`{str(duration).split('.')[0]}`", inline=False)

        if log_channel:
            try:
                await log_channel.send(embed=log_embed)
            except discord.Forbidden:
                print(f"❌ 無法在 Log 頻道發言，請檢查權限！")

        await interaction.followup.send("✅ 結單完成！頻道將在 5 秒後刪除。")
        await asyncio.sleep(5)
        await interaction.channel.delete()

# --- 2. 打手接單視圖 (3人限制) ---
class WorkerConfirmView(discord.ui.View):
    def __init__(self, customer, order_type):
        super().__init__(timeout=None)
        self.customer = customer
        self.order_type = order_type
        self.accepted_workers = [] # 儲存打手對象
        self.channel = None        # 私人頻道物件
        self.close_view = None     # 結單 View 物件

    @discord.ui.button(label="我要接單 (0/3)", style=discord.ButtonStyle.success, emoji="✅", custom_id="nun:accept_btn")
    async def accept(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.defer()

        # 基礎檢查
        worker_role = interaction.guild.get_role(WORKER_ROLE_ID)
        if worker_role not in interaction.user.roles:
            return await interaction.followup.send("❌ 你沒有打手身份組！", ephemeral=True)
        if interaction.user in self.accepted_workers:
            return await interaction.followup.send("❌ 你已經接過這單了！", ephemeral=True)
        if interaction.user.id == self.customer.id:
            return await interaction.followup.send("❌ 你不能接自己的單！", ephemeral=True)

        self.accepted_workers.append(interaction.user)
        count = len(self.accepted_workers)

        # 處理第一位接單者 (建立頻道)
        if count == 1:
            guild = interaction.guild
            category = guild.get_channel(CATEGORY_ID)
            overwrites = {
                guild.default_role: discord.PermissionOverwrite(read_messages=False),
                self.customer: discord.PermissionOverwrite(read_messages=True, send_messages=True),
                interaction.user: discord.PermissionOverwrite(read_messages=True, send_messages=True),
                guild.me: discord.PermissionOverwrite(read_messages=True, send_messages=True, manage_channels=True)
            }
            
            self.channel = await guild.create_text_channel(
                name=f"🟢-{self.order_type}-{self.customer.name}",
                overwrites=overwrites,
                category=category if isinstance(category, discord.CategoryChannel) else None
            )
            
            self.close_view = CloseChannelView(self.customer.id, [interaction.user.id], self.order_type, datetime.datetime.now())
            welcome = discord.Embed(title="🚀 派單通訊頻道", description=f"顧客：{self.customer.mention}\n打手：{interaction.user.mention}", color=discord.Color.green())
            await self.channel.send(content=f"{self.customer.mention} {interaction.user.mention}", embed=welcome, view=self.close_view)
        
        # 處理後續加入者 (更新權限)
        else:
            await self.channel.set_permissions(interaction.user, read_messages=True, send_messages=True)
            self.close_view.worker_ids.append(interaction.user.id)
            await self.channel.send(f"⚔️ 打手 {interaction.user.mention} 已加入支援！")

        # 更新按鈕文字與狀態
        button.label = f"我要接單 ({count}/{MAX_WORKERS})"
        if count >= MAX_WORKERS:
            button.disabled = True
            button.style = discord.ButtonStyle.secondary
            button.label = "接單已額滿"

        # 更新原本的嵌入訊息
        embed = interaction.message.embeds[0]
        embed.set_field_at(0, name="目前打手", value=", ".join([m.mention for m in self.accepted_workers]), inline=False)
        await interaction.message.edit(embed=embed, view=self if count < MAX_WORKERS else None)
        
        await interaction.followup.send(f"✅ 接單成功！請前往 {self.channel.mention}", ephemeral=True)

# --- 3. 持久化主選單 ---
class PersistentOrderMenu(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.select(
        placeholder="點擊此處選擇服務項目...",
        options=[
            discord.SelectOption(label="純變賣單", emoji="💰", description="開啟變賣服務"),
            discord.SelectOption(label="賭狗單", emoji="🎲", description="開啟賭狗服務"),
            discord.SelectOption(label="護航單", emoji="🛡️", description="開啟護航服務")
        ],
        custom_id="nun:main_select"
    )
    async def select_callback(self, interaction: discord.Interaction, select: discord.ui.Select):
        # 先 defer，確保不會出現三個訊息的第一步
        await interaction.response.defer(ephemeral=True)

        order_type = select.values[0]
        
        embed = discord.Embed(title=f"⏳ 待接單 - {order_type}", color=discord.Color.orange(), timestamp=datetime.datetime.now())
        embed.add_field(name="目前打手", value="等待支援中...", inline=False)
        embed.add_field(name="發起顧客", value=interaction.user.mention, inline=True)
        
        # 發送接單按鈕訊息 (全頻道可見)
        await interaction.channel.send(
            content=f"🔔 <@&{WORKER_ROLE_ID}> 有新單需求！(上限 {MAX_WORKERS} 人)",
            embed=embed, 
            view=WorkerConfirmView(interaction.user, order_type)
        )
        
        # 只有顧客看得到的確認訊息
        await interaction.followup.send(f"✅ 已成功發起 **{order_type}** 需求，請稍候。", ephemeral=True)

# --- 指令與啟動 ---
bot = NUNBot()

@bot.command()
@commands.has_permissions(administrator=True)
async def start_system(ctx):
    """(管理員) 初始化派單系統選單"""
    # 刪除觸發指令，保持頻道整潔
    await ctx.message.delete()
    
    embed = discord.Embed(
        title="🎮 NUN 俱樂部派單系統",
        description="請從下方選單選擇服務類型，系統將自動通知在線打手。",
        color=discord.Color.blue()
    )
    embed.set_footer(text="NUN Club Management System")
    await ctx.send(embed=embed, view=PersistentOrderMenu())

@bot.event
async def on_ready():
    print(f'✅ 機器人 {bot.user} 已啟動')
    print(f'✅ 多人接單功能 (上限 {MAX_WORKERS} 人) 已就緒')

if __name__ == "__main__":
    bot.run(TOKEN)
