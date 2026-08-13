import time  
import datetime  
import requests  
  
TOKEN = "8771074603:AAFmIZXie0_kGrcBevMjC88mGvNyLMkMcSQ"  
URL = f"https://api.telegram.org/bot{TOKEN}/"  
  
chat_data = {}  
  
def clear_webhook():  
    try:  
        requests.get(URL + "deleteWebhook", timeout=10)  
    except Exception:  
        pass  
  
def send_message(chat_id, text, reply_markup=None):  
    try:  
        payload = {"chat_id": chat_id, "text": text}  
        if reply_markup:  
            payload["reply_markup"] = reply_markup  
        requests.post(URL + "sendMessage", json=payload, timeout=10)  
    except Exception as e:  
        print("កំហុសក្នុងការផ្ញើសារ:", e)  
  
def edit_message_text(chat_id, message_id, text, reply_markup=None):  
    try:  
        payload = {"chat_id": chat_id, "message_id": message_id, "text": text}  
        if reply_markup:  
            payload["reply_markup"] = reply_markup  
        requests.post(URL + "editMessageText", json=payload, timeout=10)  
    except Exception as e:  
        print("កំហុសក្នុងការកែប្រែសារ:", e)  
  
def send_document(chat_id, file_bytes, filename):  
    try:  
        files = {'document': (filename, file_bytes, 'text/csv')}  
        data = {'chat_id': chat_id}  
        requests.post(URL + "sendDocument", data=data, files=files, timeout=15)  
    except Exception as e:  
        print("កំហុសក្នុងការផ្ញើ Document:", e)  
  
def get_okx_p2p_rates():  
    headers = {  
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'  
    }  
    url = "https://www.okx.com/v3/c2c/tradingOrders/books?quoteCurrency=cny&baseCurrency=usdt&side=sell&paymentMethod=all&userType=all"  
      
    try:  
        res = requests.get(url, headers=headers, timeout=5).json()  
        if res.get("code") == 0 and "data" in res and "sell" in res["data"]:  
            sellers = res["data"]["sell"][:10]  
            rates_list = []  
            for idx, s in enumerate(sellers, 1):  
                price = s.get("price", "6.67")  
                nick = s.get("nickName", "商家")  
                rates_list.append(f"{idx}) {price}   {nick}")  
              
            top_rate = float(sellers[0]["price"]) if sellers else 6.67  
            return "\n".join(rates_list), top_rate  
    except Exception:  
        pass  
      
    default_list = (  
        "1) 6.67   小鱼儿诚信商行\n"  
        "2) 6.68   湘益U商\n"  
        "3) 6.68   三友币行\n"  
        "4) 6.68   六福商贸\n"  
        "5) 6.68   旺集商行\n"  
        "6) 6.68   小川国际\n"  
        "7) 6.68   来洋合众国\n"  
        "8) 6.68   鑫系华夏\n"  
        "9) 6.68   螃蟹横着走\n"  
        "10) 6.68   喜临门商行"  
    )  
    return default_list, 6.67  
  
def get_report_keyboard():  
    return {  
        "inline_keyboard": [  
            [  
                {"text": "📊 刷新账单", "callback_data": "btn_report"},  
                {"text": "📥 导出Excel", "callback_data": "btn_excel"}  
            ],  
            [  
                {"text": "↩️ 撤销上一笔", "callback_data": "btn_undo"},  
                {"text": "🔄 重置账单", "callback_data": "btn_reset"}  
            ]  
        ]  
    }  
  
def build_report(chat_id):  
    data = chat_data[chat_id]  
    count_in = len(data["in_records"])  
    count_out = len(data["out_records"])  
  
    in_list_str = "\n".join([r["str"] for r in data["in_records"]]) if data["in_records"] else "无"  
    out_list_str = "\n".join([r["str"] for r in data["out_records"]]) if data["out_records"] else "无"  
  
    tot_amt_val = data["total_amount"]  
    tot_amt = int(tot_amt_val) if tot_amt_val.is_integer() else f"{tot_amt_val:.2f}"  
      
    tot_in_usdt = data["total_in_usdt"]  
    fee_rate = data["fee_rate"]  
      
    fee_usdt = tot_in_usdt * (fee_rate / 100.0)  
    should_pay_usdt = tot_in_usdt - fee_usdt  
      
    tot_out_usdt = data["total_out_usdt"]  
    remaining_usdt = should_pay_usdt - tot_out_usdt  
  
    limit_msg = ""  
    if data["usdt_limit"] is not None:  
        max_limit = data["usdt_limit"]  
        if tot_in_usdt >= max_limit:  
            limit_msg = f"\n\n⚠️ 已超额! ({tot_in_usdt:.2f} / {max_limit:.2f} U)"  
        else:  
            limit_msg = f"\n\n📊 限额: {tot_in_usdt:.2f} / {max_limit:.2f} U"  
  
    fee_info = f" ({fee_rate}% 手续费: {fee_usdt:.2f} U)" if fee_rate > 0 else ""  
  
    text = (  
        f"众合管家\n\n"  
        f"入账({count_in}笔)：\n"  
        f"{in_list_str}\n\n"  
        f"下发({count_out}笔)：\n"  
        f"{out_list_str}\n\n"  
        f"总入款：{tot_amt}\n"  
        f"USDT汇率：{data['rate']}\n\n"  
        f"应下发：{tot_amt} | {should_pay_usdt:.2f} U{fee_info}\n"  
        f"总下发：{tot_out_usdt:.2f} U\n"  
        f"未下发：{tot_amt} | {remaining_usdt:.2f} U"  
        f"{limit_msg}"  
    )  
    return text  
  
def check_and_auto_reset(chat_id):  
    today_str = datetime.datetime.now().strftime("%Y-%m-%d")  
    data = chat_data[chat_id]  
      
    if data["last_date"] != today_str:  
        if data["in_records"] or data["out_records"]:  
            data["history"].append({  
                "date": data["last_date"],  
                "report": build_report(chat_id)  
            })  
            send_message(chat_id, f"🔄 跨天自动重置：已保存 {data['last_date']} 的账单至历史账单！")  
  
        data["in_records"] = []  
        data["out_records"] = []  
        data["total_amount"] = 0.0  
        data["total_in_usdt"] = 0.0  
        data["total_out_usdt"] = 0.0  
        data["last_date"] = today_str  
  
def main():  
    clear_webhook()  
    last_update_id = None  
    print("Bot កំពុងដំណើរការ...")  
  
    while True:  
        try:  
            for c_id in list(chat_data.keys()):  
                check_and_auto_reset(c_id)  
  
            url = URL + "getUpdates?timeout=10"  
            if last_update_id:  
                url += f"&offset={last_update_id}"  
  
            res = requests.get(url, timeout=15).json()  
  
            if "result" in res:  
                for update in res["result"]:  
                    last_update_id = update["update_id"] + 1  
  
                    if "callback_query" in update:  
                        cq = update["callback_query"]  
                        cq_id = cq["id"]  
                        chat_id = cq["message"]["chat"]["id"]  
                        message_id = cq["message"]["message_id"]  
                        data_action = cq["data"]  
  
                        if chat_id not in chat_data:  
                            chat_data[chat_id] = {  
                                "in_records": [], "out_records": [], "history": [],  
                                "total_amount": 0.0, "total_in_usdt": 0.0, "total_out_usdt": 0.0,  
                                "rate": 0.0, "fee_rate": 0.0, "usdt_limit": None,  
                                "last_date": datetime.datetime.now().strftime("%Y-%m-%d")  
                            }  
  
                        if data_action == "btn_report":  
                            edit_message_text(chat_id, message_id, build_report(chat_id), reply_markup=get_report_keyboard())  
                        elif data_action == "btn_excel":  
                            today_str = datetime.datetime.now().strftime("%Y-%m-%d")  
                            filename = f"账单_{today_str}.csv"  
                            csv_content = "\ufeff类型,时间,金额,汇率,USDT\n"  
                            for r in chat_data[chat_id]["in_records"]:  
                                csv_content += f"入账,{r['time']},{r['amount']},{r['rate']},{r['usdt']:.2f}\n"  
                            for r in chat_data[chat_id]["out_records"]:  
                                csv_content += f"下发,{r['time']},-,-,{r['usdt']:.2f}\n"  
                            send_document(chat_id, csv_content.encode('utf-8-sig'), filename)  
                        elif data_action == "btn_undo":  
                            if chat_data[chat_id]["in_records"]:  
                                last_rec = chat_data[chat_id]["in_records"].pop()  
                                chat_data[chat_id]["total_amount"] -= last_rec["amount"]  
                                chat_data[chat_id]["total_in_usdt"] -= last_rec["usdt"]  
                                edit_message_text(chat_id, message_id, build_report(chat_id), reply_markup=get_report_keyboard())  
                            else:  
                                requests.post(URL + "answerCallbackQuery", json={"callback_query_id": cq_id, "text": "⚠️ 当前没有可撤销的入账记录。"})  
                        elif data_action == "btn_reset":  
                            r_temp = chat_data[chat_id]["rate"]  
                            l_temp = chat_data[chat_id]["usdt_limit"]  
                            f_temp = chat_data[chat_id]["fee_rate"]  
                            chat_data[chat_id]["in_records"] = []  
                            chat_data[chat_id]["out_records"] = []  
                            chat_data[chat_id]["total_amount"] = 0.0  
                            chat_data[chat_id]["total_in_usdt"] = 0.0  
                            chat_data[chat_id]["total_out_usdt"] = 0.0  
                            edit_message_text(chat_id, message_id, build_report(chat_id), reply_markup=get_report_keyboard())  
  
                        requests.post(URL + "answerCallbackQuery", json={"callback_query_id": cq_id})  
                        continue  
  
                    if "message" in update and "text" in update["message"]:  
                        chat_id = update["message"]["chat"]["id"]  
                        text = update["message"]["text"].strip()  
                          
                        if "@" in text:  
                            text = text.split("@")[0]  
  
                        today_str = datetime.datetime.now().strftime("%Y-%m-%d")  
  
                        if chat_id not in chat_data:  
                            chat_data[chat_id] = {  
                                "in_records": [], "out_records": [], "history": [],  
                                "total_amount": 0.0, "total_in_usdt": 0.0, "total_out_usdt": 0.0,  
                                "rate": 0.0, "fee_rate": 0.0, "usdt_limit": None,  
                                "last_date": today_str  
                            }  
  
                        check_and_auto_reset(chat_id)  
  
                        if text.upper() == "Z0":  
                            top10_str, top1_rate = get_okx_p2p_rates()  
                            if chat_data[chat_id]["rate"] == 0:  
                                chat_data[chat_id]["rate"] = top1_rate  
                                  
                            curr_rate = chat_data[chat_id]["rate"]  
                            curr_fee = chat_data[chat_id]["fee_rate"]  
                            tot_amt_val = chat_data[chat_id]["total_amount"]  
                            tot_amt = int(tot_amt_val) if tot_amt_val.is_integer() else f"{tot_amt_val:.2f}"  
                            tot_in_usdt = chat_data[chat_id]["total_in_usdt"]  
                            calc_usdt_str = "0" if tot_in_usdt == 0 else f"{tot_in_usdt:.2f}"  
                              
                            z0_response = (  
                                f"欧易所有商家实时交易汇率top10\n\n{top10_str}\n\n"  
                                f"本群汇率：实时汇率 {curr_rate}\n本群费率：{int(curr_fee) if curr_fee.is_integer() else curr_fee}\n\n"  
                                f"💰 {tot_amt} / {curr_rate} = {calc_usdt_str}"  
                            )  
                            send_message(chat_id, z0_response)  
                            continue  
  
                        if text.startswith("设置汇率") or text.startswith("汇率") or text.lower().startswith("rate"):  
                            parts = text.split()  
                            if len(parts) > 1:  
                                try:  
                                    new_rate = float(parts[1])  
                                    chat_data[chat_id]["rate"] = new_rate  
                                    send_message(chat_id, f"✅ 已设置汇率：{new_rate}\n\n" + build_report(chat_id), reply_markup=get_report_keyboard())  
                                except ValueError:  
                                    pass  
                            continue  
  
                        if text.startswith("设置手续费") or text.startswith("手续费") or text.startswith("费率"):  
                            parts = text.split()  
                            if len(parts) > 1:  
                                try:  
                                    new_fee = float(parts[1])  
                                    chat_data[chat_id]["fee_rate"] = new_fee  
                                    send_message(chat_id, f"✅ 已设置手续费：{new_fee}%\n\n" + build_report(chat_id), reply_markup=get_report_keyboard())  
                                except ValueError:  
                                    pass  
                            continue  
  
                        if text in ["开始", "/start", "重置", "reset"]:  
                            chat_data[chat_id]["in_records"] = []  
                            chat_data[chat_id]["out_records"] = []  
                            chat_data[chat_id]["total_amount"] = 0.0  
                            chat_data[chat_id]["total_in_usdt"] = 0.0  
                            chat_data[chat_id]["total_out_usdt"] = 0.0  
                            send_message(chat_id, "✅ 已本群开始作业，祝各位业绩千万！", reply_markup=get_report_keyboard())  
                            continue  
  
                        if text in ["+0", "账单", "របាយការណ៍"]:  
                            send_message(chat_id, build_report(chat_id), reply_markup=get_report_keyboard())  
                            continue  
  
                        if text.lower() in ["导出", "excel", "导出excel"]:  
                            filename = f"账单_{today_str}.csv"  
                            csv_content = "\ufeff类型,时间,金额,汇率,USDT\n"  
                            for r in chat_data[chat_id]["in_records"]:  
                                csv_content += f"入账,{r['time']},{r['amount']},{r['rate']},{r['usdt']:.2f}\n"  
                            for r in chat_data[chat_id]["out_records"]:  
                                csv_content += f"下发,{r['time']},-,-,{r['usdt']:.2f}\n"  
                            send_document(chat_id, csv_content.encode('utf-8-sig'), filename)  
                            continue  
  
                        if text.startswith("+") and text != "+0":  
                            try:  
                                expr = text[1:].strip()  
                                if "/" in expr:  
                                    parts = expr.split('/')  
                                    amount_part = parts[0].strip()  
                                    rate_part = parts[1].strip()  
                                    rate = float(rate_part)  
                                else:  
                                    amount_part = expr  
                                    if chat_data[chat_id]["rate"] > 0:  
                                        rate = chat_data[chat_id]["rate"]  
                                    else:  
                                        _, top1_r = get_okx_p2p_rates()  
                                        rate = top1_r  
                                        chat_data[chat_id]["rate"] = top1_r  
                                    rate_part = str(rate)  
  
                                amount = float(amount_part)  
                                usdt = amount / rate  
                                current_time = datetime.datetime.now().strftime("%H:%M:%S")  
  
                                record_str = f"{current_time} {amount_part}/{rate_part}={usdt:.2f}"  
                                chat_data[chat_id]["in_records"].append({  
                                    "time": current_time, "amount": amount, "rate": rate, "usdt": usdt, "str": record_str  
                                })  
                                chat_data[chat_id]["total_amount"] += amount  
                                chat_data[chat_id]["total_in_usdt"] += usdt  
  
                                send_message(chat_id, build_report(chat_id), reply_markup=get_report_keyboard())  
                            except Exception:  
                                pass  
  
        except Exception as e:  
            print("កំហុសប្រព័ន្ធ:", e)  
  
        time.sleep(1)  
  
if __name__ == '__main__':  
    main()  
