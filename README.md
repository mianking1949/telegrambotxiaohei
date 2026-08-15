import io
import re
import sqlite3
import pandas as pd
import requests
from datetime import datetime, date
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application, CommandHandler, MessageHandler, CallbackQueryHandler, filters, ContextTypes
)

# ==================== 配置项 ====================
BOT_TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"  # 替换为你的 Telegram Bot Token (从 @BotFather 获取)
ADMIN_IDS = [587654321]  # 替换为你的纯数字 Telegram User ID (使用 @userinfobot 获取)

# TronGrid API Key (建议配置，避免免费接口请求频率受限)
TRONGRID_API_KEY = "YOUR_TRONGRID_API_KEY"

# TRC-20 USDT 合约地址
USDT_TRC20_CONTRACT = "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t"

# 正则表达式匹配 TRX 地址 (T开头，base58字符，长度34)
PATTERN_TRX = r"^T[a-km-zA-HJ-NP-Z1-9]{33}$"


# ==================== 1. 数据库与权限系统 (SQLite) ====================

def init_db():
    conn = sqlite3.connect("bot_user_data.db")
    cursor = conn.cursor()
    # 用户表：记录 ID、权限等级 (user/vip/admin)、每日查询统计
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        role TEXT DEFAULT 'user',
        daily_count INTEGER DEFAULT 0,
        last_query_date TEXT
    )
    """)
    conn.commit()
    conn.close()

def check_and_update_perm(user_id: int) -> tuple[bool, str]:
    """检查用户权限与每日额度"""
    if user_id in ADMIN_IDS:
        return True, "admin"

    conn = sqlite3.connect("bot_user_data.db")
    cursor = conn.cursor()
    
    today_str = str(date.today())
    cursor.execute("SELECT role, daily_count, last_query_date FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()

    if not row:
        cursor.execute("INSERT INTO users (user_id, role, daily_count, last_query_date) VALUES (?, 'user', 1, ?)", (user_id, today_str))
        conn.commit()
        conn.close()
        return True, "user"

    role, count, last_date = row
    
    if role == "banned":
        conn.close()
        return False, "❌ 您的账号已被系统封禁，无法使用此功能。"

    # 重置新一天的计数
    if last_date != today_str:
        count = 0

    limit = 100 if role == "vip" else 5  # 普通用户每天 5 次，VIP 100 次

    if count >= limit:
        conn.close()
        return False, f"⚠️ 您今天的查询额度已用完 ({count}/{limit} 次)。如需提升额度请联系管理员开通 VIP。"

    # 更新计数
    cursor.execute("UPDATE users SET daily_count = ?, last_query_date = ? WHERE user_id = ?", (count + 1, today_str, user_id))
    conn.commit()
    conn.close()
    return True, role


# ==================== 2. 实时汇率系统 (CoinGecko API) ====================

def get_trx_usdt_prices():
    """获取 TRX 和 USDT 的最新价格 (USD / CNY)"""
    url = "https://api.coingecko.com/api/v3/simple/price?ids=tron,tether&vs_currencies=usd,cny"
    try:
        res = requests.get(url, timeout=5).json()
        return {
            "TRX": res.get("tron", {"usd": 0.12, "cny": 0.86}),
            "USDT": res.get("tether", {"usd": 1.0, "cny": 7.2}),
        }
    except Exception:
        # 异常时的备用保底汇率
        return {
            "TRX": {"usd": 0.12, "cny": 0.86},
            "USDT": {"usd": 1.0, "cny": 7.2},
        }


# ==================== 3. TRX 数据查询 + CSV 报告准备 ====================

def format_time(ms_timestamp):
    if not ms_timestamp: 
        return "未知"
    if ms_timestamp > 1e11: 
        ms_timestamp /= 1000
    return datetime.fromtimestamp(ms_timestamp).strftime("%Y-%m-%d %H:%M:%S")

def query_tron(address: str):
    headers = {"Accept": "application/json"}
    if TRONGRID_API_KEY and TRONGRID_API_KEY != "YOUR_TRONGRID_API_KEY":
        headers["TRON-PRO-API-KEY"] = TRONGRID_API_KEY

    payload = {"address": address, "visible": True}
    
    # 1. 账户基本信息
    res = requests.post("https://api.trongrid.io/wallet/getaccount", json=payload, headers=headers).json()
    trx_bal = res.get("balance", 0) / 1_000_000
    create_time = format_time(res.get("create_time"))
    
    # 质押 TRX
    frozen_v2 = res.get("frozenV2", [])
    staked_trx = sum(item.get("amount", 0) for item in frozen_v2) / 1_000_000

    # USDT 余额
    usdt_bal = 0.0
    for item in res.get("trc20", []):
        if USDT_TRC20_CONTRACT in item:
            usdt_bal = float(item[USDT_TRC20_CONTRACT]) / 1_000_000
            break

    # 2. 能量和带宽
    res_data = requests.post("https://api.trongrid.io/wallet/getaccountresource", json=payload, headers=headers).json()
    free_bw = max(0, res_data.get("freeNetLimit", 600) - res_data.get("freeNetUsed", 0))
    staked_bw = max(0, res_data.get("NetLimit", 0) - res_data.get("NetUsed", 0))
    available_energy = max(0, res_data.get("EnergyLimit", 0) - res_data.get("EnergyUsed", 0))

    # 3. 汇率与资产折算
    prices = get_trx_usdt_prices()
    total_usd = ((trx_bal + staked_trx) * prices["TRX"]["usd"]) + (usdt_bal * prices["USDT"]["usd"])
    total_cny = ((trx_bal + staked_trx) * prices["TRX"]["cny"]) + (usdt_bal * prices["USDT"]["cny"])

    # 4. 获取 USDT 交易明细并准备 CSV 数据
    tx_url = f"https://api.trongrid.io/v1/accounts/{address}/transactions/trc20?limit=50&contract_address={USDT_TRC20_CONTRACT}"
    tx_data = requests.get(tx_url, headers=headers).json().get("data", [])
    
    usdt_in_count = 0
    usdt_out_count = 0
    recent_tx_lines = []
    export_list = []
    last_active = format_time(tx_data[0].get("block_timestamp")) if tx_data else "无近期交易"

    for tx in tx_data:
        val = float(tx.get("value", 0)) / 1_000_000
        t_str = format_time(tx.get("block_timestamp"))
        is_in = tx.get("to") == address
        direction = "转入" if is_in else "转出"
        
        if is_in:
            usdt_in_count += 1
            if len(recent_tx_lines) < 5:
                recent_tx_lines.append(f"{t_str} 转入 {val:,.2f} USDT")
        else:
            usdt_out_count += 1
            if len(recent_tx_lines) < 5:
                recent_tx_lines.append(f"{t_str} 转出 {val:,.2f} USDT")

        # 添加至导出数据列表
        export_list.append({
            "交易哈希": tx.get("transaction_id"),
            "时间": t_str,
            "类型": direction,
            "金额": val,
            "代币": "USDT",
            "对方地址": tx.get("from") if is_in else tx.get("to")
        })

    tx_history_text = "\n".join(recent_tx_lines) if recent_tx_lines else "近期无 USDT 交易记录"

    msg = f"""👤账户类型: 普通账户
🔍查询地址: `{address}`
⏰创建时间: {create_time}
🌟最后活跃: {last_active}
➖➖➖➖➖➖➖➖
💰 TRX  余额：{trx_bal:,.2f}
💰 TRX  质押：{staked_trx:,.2f}
💰USDT余额：{usdt_bal:,.2f}
🔋能量：{available_energy:,}
📡质押带宽：{staked_bw:,}
📡免费带宽：{free_bw:,}
💵 **预估总资产**: ${total_usd:,.2f} USD (≈ ￥{total_cny:,.2f} CNY)
➖➖➖➖➖➖➖➖➖➖➖➖➖➖➖➖➖➖➖➖
⤴️USDT支出笔数：{usdt_out_count}  ⤵️USDT收入笔数：{usdt_in_count}
{tx_history_text}"""
    
    return msg, export_list


# ==================== 4. Telegram 交互句柄 ====================

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 你好！请直接发送 TRX 地址（以 T 开头的 34 位字符串），我将为你查询账户详情。")

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    text = update.message.text.strip()

    # 正则识别 TRX 地址
    if re.match(PATTERN_TRX, text):
        # 权限与每日额度校验
        passed, role_or_msg = check_and_update_perm(user_id)
        if not passed:
            await update.message.reply_text(role_or_msg)
            return

        loading = await update.message.reply_text("🔍 正在查询波场链上数据...")

        try:
            reply_msg, export_data = query_tron(text)
            
            # 暂存交易数据以供 CSV 导出
            context.user_data[f"export_{text}"] = export_data
            
            # 挂载 Telegram 导出按钮
            keyboard = [[InlineKeyboardButton("📥 导出 CSV 交易报告", callback_data=f"csv_{text}")]]
            reply_markup = InlineKeyboardMarkup(keyboard)

            await loading.edit_text(reply_msg, parse_mode="Markdown", reply_markup=reply_markup)
        except Exception as e:
            await loading.edit_text(f"❌ 查询失败，请检查地址或稍后再试。\n错误信息: {str(e)}")


async def handle_csv_export(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """处理用户点击导出 CSV 按钮的请求"""
    query = update.callback_query
    await query.answer()

    address = query.data.replace("csv_", "")
    tx_list = context.user_data.get(f"export_{address}")

    if not tx_list:
        await query.message.reply_text("⚠️ 导出的数据超时失效，请重新发送地址查询后再导出。")
        return

    # 生成 CSV 字节流
    df = pd.DataFrame(tx_list)
    csv_buffer = io.BytesIO()
    df.to_csv(csv_buffer, index=False, encoding="utf-8-sig")
    csv_buffer.seek(0)

    # 发送文件给用户
    await query.message.reply_document(
        document=csv_buffer,
        filename=f"TRX_Report_{address[:8]}.csv",
        caption=f"📄 地址 `{address}` 的 50 笔 USDT 转账明细已生成。"
    )


# ==================== 5. 管理员控制指令 ====================

async def add_vip(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """管理员开通 VIP 命令: /addvip <user_id>"""
    if update.effective_user.id not in ADMIN_IDS:
        return
    if not context.args:
        await update.message.reply_text("使用方式: `/addvip 12345678`", parse_mode="Markdown")
        return
    
    target_id = int(context.args[0])
    conn = sqlite3.connect("bot_user_data.db")
    cursor = conn.cursor()
    cursor.execute("INSERT INTO users (user_id, role) VALUES (?, 'vip') ON CONFLICT(user_id) DO UPDATE SET role='vip'", (target_id,))
    conn.commit()
    conn.close()
    
    await update.message.reply_text(f"✅ 已成功将用户 `{target_id}` 设为 VIP (每日查询额度提升至 100 次)！", parse_mode="Markdown")

async def ban_user(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """管理员封禁命令: /ban <user_id>"""
    if update.effective_user.id not in ADMIN_IDS:
        return
    if not context.args:
        await update.message.reply_text("使用方式: `/ban 12345678`", parse_mode="Markdown")
        return

    target_id = int(context.args[0])
    conn = sqlite3.connect("bot_user_data.db")
    cursor = conn.cursor()
    cursor.execute("INSERT INTO users (user_id, role) VALUES (?, 'banned') ON CONFLICT(user_id) DO UPDATE SET role='banned'", (target_id,))
    conn.commit()
    conn.close()
    
    await update.message.reply_text(f"🚫 已将用户 `{target_id}` 加入黑名单并封禁。", parse_mode="Markdown")


# ==================== 启动主程序 ====================

def main():
    init_db()  # 初始化数据库
    app = Application.builder().token(BOT_TOKEN).build()

    # 注册管理员命令
    app.add_handler(CommandHandler("addvip", add_vip))
    app.add_handler(CommandHandler("ban", ban_user))
    app.add_handler(CommandHandler("start", start))

    # 注册消息与按钮回调
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    app.add_handler(CallbackQueryHandler(handle_csv_export, pattern=r"^csv_"))

    print("🤖 TRX 查询机器人运行中 (已包含管理员控制、权限分级、汇率折算与数据导出功能)...")
    app.run_polling()

if __name__ == "__main__":
    main()
