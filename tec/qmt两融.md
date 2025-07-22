```python
#encoding:gbk

import pandas as pd
import numpy as np
import time
from datetime import timedelta,datetime

# 定义全局控制开关
buy_flag = True
sell_flag = True

active_code = []

target_rate = {}
target_total_value = 23000

#自定义类 用来保存状态 
class a(object):
	def __init__(self):
		#初始化委托状态记录容器
		self.waiting_dict = {}
		self.all_order_ref_dict = {}
		#撤单间隔 单位秒 超过间隔未成交的委托撤回重报
		self.withdraw_secs = 10
		#定义策略开始结束时间 在两者间时进行下单判断 其他时间跳过
		self.start_time = '093000'
		self.end_time = '150000'
A=a()


def init(C):
	global target_stocks
	A.final_dict = {}
	'''设置交易账号 acount accountType是界面上选的账号 账号类型'''
	A.acct = account
	A.acct_type = accountType
	#定时器 定时触发指定函数
	C.run_time("f","5nSecond","2024-02-21 09:30:00","SH")


def f(C):
	target_stocks = [key for key in target_rate]
	print("target_stocks : ", target_stocks)
	ticks = C.get_full_tick(target_stocks)
	nstocks = len(target_stocks)
	rate_sum = 0.0
	for key in target_rate:
		rate_sum = rate_sum + target_rate[key]
	print("rate_sum: {:.2f}".format(rate_sum))
	for key in target_rate:
		if key not in A.final_dict:
			try:
				ooo_price = ticks[key]["lastPrice"]
				key_rate = target_rate[key]
				tvalue = key_rate / rate_sum * target_total_value
				ooo_vol = max(int(tvalue / ooo_price / 100) * 100, 100)
				A.final_dict[key] = ooo_vol
			except Exception as e:
				print("异常 {} {}".format(key, e))
		else:
			continue
	print("nstocks: {}".format(nstocks))
	ooo_real_total_value = 0
	for key in A.final_dict:
		ooo_real_total_value += A.final_dict[key] * ticks[key]["lastPrice"]
	print("real_total_value: {:.2f}".format(ooo_real_total_value))
	print("target_positions: {}".format(A.final_dict))
	
	'''定义定时触发的函数 入参是ContextInfo对象'''
	#记录本次调用时间戳
	t0 = time.time()
	final_dict=A.final_dict
	#本次运行时间字符串
	now = datetime.now()
	now_timestr = now.strftime("%H%M%S")
	#跳过非交易时间
	if now_timestr < A.start_time or now_timestr > A.end_time:
		return
	#获取账号信息
	acct = get_trade_detail_data(A.acct, A.acct_type, 'account')
	if len(acct) == 0:
		print(A.acct, '账号未登录 停止委托')
		return
	acct = acct[0]
	#获取可用资金
	available_cash = acct.m_dAvailable
	print(now, '可用资金', available_cash)
	#获取持仓信息
	position_list = get_trade_detail_data(A.acct, A.acct_type, 'position')
	#持仓数据 组合为字典
	position_dict = {i.m_strInstrumentID + '.' + i.m_strExchangeID : int(i.m_nVolume) for i in position_list}
	position_dict_available = {i.m_strInstrumentID + '.' + i.m_strExchangeID : int(i.m_nCanUseVolume) for i in position_list}
	#未持有的品种填充持股数0
	not_in_position_stock_dict = {i : 0 for i in final_dict if i not in position_dict}
	position_dict.update(not_in_position_stock_dict)
	#print(position_dict)
	stock_list = list(position_dict.keys())
	print("stock_list : ", stock_list)
	# print(stock_list)
	#获取全推行情
	full_tick = C.get_full_tick(stock_list)

	#print('fulltick', full_tick)
	#更新持仓状态记录
	refresh_waiting_dict(C)
	#撤超时委托
	order_list = get_trade_detail_data(A.acct, 'stock', 'order')
	if '091500'<= now_timestr <= '093000':#指定的范围內不撤单
		pass
	else:
		for order in order_list:
			#非本策略 本次运行记录的委托 不撤
			if order.m_strRemark not in A.all_order_ref_dict:
				continue
			#委托后 时间不到撤单等待时间的 不撤
			if time.time() - A.all_order_ref_dict[order.m_strRemark] < A.withdraw_secs:
				continue
			#对所有可撤状态的委托 撤单
			if order.m_nOrderStatus in [48,49,50,51,52,55,86,255]:
				print(f"超时撤单 停止等待 {order.m_strRemark}")
				cancel(order.m_strOrderSysID,A.acct,'stock',C)
	#下单判断
	for stock in position_dict:
		# 跳过主观交易持仓
		if stock in active_code:
			continue
		#有未查到的委托的品种 跳过下单 防止超单
		if stock in A.waiting_dict:
			print(f"{stock} 未查到或存在未撤回委托 {A.waiting_dict[stock]} 暂停后续报单")
			continue
		if stock in position_dict.keys():
			#print(position_dict[stock],target_vol,'1111')
			#到达目标数量的品种 停止委托
			target_vol = final_dict[stock] if stock in final_dict else 0
			if int(abs(position_dict[stock] - target_vol)) == 0:
				print(stock, C.get_stock_name(stock), '与目标一致')
				continue
			#与目标数量差值小于100股的品种 停止委托
			if abs(position_dict[stock] - target_vol) < 100:
				print(f"{stock} {C.get_stock_name(stock)} 目标持仓{target_vol} 当前持仓{position_dict[stock]} 差额小于100 停止委托")
				continue
			# 允许买入
			if sell_flag:
				#持仓大于目标持仓 卖出
				if position_dict[stock]>target_vol:
					vol = int((position_dict[stock] - target_vol)/100)*100
					if stock not in position_dict_available:
						continue
					vol = min(vol, position_dict_available[stock])
					#获取买一价
					print(stock,'应该卖出')
					if stock not in full_tick:
						print("没有该股票[{}]tick数据，跳过...".format(stock))
						continue
					buy_one_price = full_tick[stock]['bidPrice'][0]
					#买一价无效时 跳过委托
					if not buy_one_price > 0:
						print(f"{stock} {C.get_stock_name(stock)} 取到的价格{buy_one_price}无效，跳过此次推送")
						continue
					print(f"{stock} {C.get_stock_name(stock)} 目标股数{target_vol} 当前股数{position_dict[stock]}")
					msg = f"{now.strftime('%Y%m%d%H%M%S')}_{stock}_sell_{vol}股"
					print(msg)
					#挂单价卖出
					passorder(24,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
					A.waiting_dict[stock] = msg
					A.all_order_ref_dict[msg] = time.time()
			if buy_flag:
				#持仓小于目标持仓 买入
				if position_dict[stock]<target_vol:
					vol = int((target_vol-position_dict[stock])/100)*100
					#获取卖一价
					if stock not in full_tick:
						print("没有该股票[{}]tick数据，跳过...".format(stock))
					sell_one_price = full_tick[stock]['askPrice'][0]
					#卖一价无效时 跳过委托
					if not sell_one_price > 0:
						print(f"{stock} {C.get_stock_name(stock)} 取到的价格{sell_one_price}无效，跳过此次推送")
						continue
					target_value = sell_one_price * vol
					if target_value > available_cash:
						print(f"{stock} 目标市值{target_value} 大于 可用资金{available_cash} 跳过委托")
						continue
					print(f"{stock} {C.get_stock_name(stock)} 目标股数{target_vol} 当前股数{position_dict[stock]}")
					msg = f"{now.strftime('%Y%m%d%H%M%S')}_{stock}_buy_{vol}股"
					print(msg)
					#挂单价买入
					
					#普通账户挂单买入
					#passorder(23,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
					#担保品买入
					passorder(33,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
					#融资买入
					#passorder(27,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
					A.waiting_dict[stock] = msg
					A.all_order_ref_dict[msg] = time.time()
					available_cash -= target_value
	#打印函数运行耗时 定时器间隔应大于该值
	print(f"下单判断函数运行完成 耗时{time.time() - t0}秒")


def refresh_waiting_dict(C):
	"""更新委托状态 入参为ContextInfo对象"""
	#获取委托信息
	order_list = get_trade_detail_data(A.acct,A.acct_type,'order')
	#取出委托对象的 投资备注 : 委托状态
	ref_dict = {i.m_strRemark : int(i.m_nOrderStatus) for i in order_list}
	del_list = []
	for stock in A.waiting_dict:
		if A.waiting_dict[stock] in ref_dict and ref_dict[A.waiting_dict[stock]] in [56, 53, 54]:
			#查到对应投资备注 且状态为成交 / 已撤 / 部撤， 从等待字典中删除
			print(f'查到投资备注 {A.waiting_dict[stock]}，的委托 状态{ref_dict[A.waiting_dict[stock]]} (56已成 53部撤 54已撤)从等待等待字典中删除')
			del_list.append(stock)
		if A.waiting_dict[stock] in ref_dict and ref_dict[A.waiting_dict[stock]] == 57:
			#委托状态是废单的 也停止等待 从等待字典中删除 
			print(f"投资备注为{A.waiting_dict[stock]}的委托状态为废单 停止等待")
			del_list.append(stock)
	for stock in del_list:
		del A.waiting_dict[stock]
```
