### 目标仓位

```python
#encoding:gbk

import pandas as pd
import numpy as np
import time
from datetime import timedelta,datetime

# 定义全局控制开关
buy_flag = True
sell_flag = True

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
	my_targets = {'300059.SZ': 1900, '300856.SZ': 400, '600873.SH': 2500, '600741.SH': 1500, '688556.SH': 800, '300622.SZ': 1700, '601865.SH': 1200, '000014.SZ': 2200, '002737.SZ': 1000, '003028.SZ': 700, '603529.SH': 800, '000538.SZ': 500, '000598.SZ': 4400, '603688.SH': 300, '603801.SH': 1700, '600309.SH': 300, '603899.SH': 700, '600233.SH': 2100, '688239.SH': 700, '600976.SH': 400, '000651.SZ': 600, '003006.SZ': 1600, '600580.SH': 2400, '300827.SZ': 1000, '600690.SH': 1100, '002871.SZ': 3300, '300144.SZ': 2400, '603439.SH': 1700, '600905.SH': 5800, '301119.SZ': 1300, '600674.SH': 1700, '300938.SZ': 800, '002415.SZ': 700, '002968.SZ': 3000, '601166.SH': 1600, '600325.SH': 3900, '603444.SH': 100, '603816.SH': 700, '688059.SH': 400, '603515.SH': 1600, '600886.SH': 1900, '601985.SH': 3100, '688002.SH': 600, '600298.SH': 800, '688078.SH': 900, '600031.SH': 1900, '603156.SH': 1000, '601318.SH': 600, '300395.SZ': 800, '601636.SH': 4000, '300009.SZ': 2700, '002867.SZ': 1400, '600036.SH': 800, '002594.SZ': 100, '600481.SH': 3200, '600760.SH': 700, '600900.SH': 1000, '688618.SH': 600, '600048.SH': 2800, '600258.SH': 1800, '601117.SH': 3800, '603040.SH': 1200, '002683.SZ': 1400, '603027.SH': 1700, '600398.SH': 3100, '600377.SH': 2200, '601888.SH': 300, '688029.SH': 300, '000333.SZ': 400, '002835.SZ': 1900, '001269.SZ': 500, '688516.SH': 300, '688389.SH': 1400, '301367.SZ': 200, '000513.SZ': 700, '002601.SZ': 1400, '601012.SH': 1200, '001222.SZ': 2200, '000338.SZ': 1600, '000895.SZ': 900, '002311.SZ': 600, '002304.SZ': 200, '000858.SZ': 100, '300450.SZ': 1100, '688777.SH': 500, '300861.SZ': 900, '300532.SZ': 1800, '002372.SZ': 1700, '688080.SH': 700, '002690.SZ': 1400, '003011.SZ': 1700, '002555.SZ': 1300, '600030.SH': 1200, '601006.SH': 3600, '601958.SH': 2600, '300776.SZ': 500, '600346.SH': 2100, '300416.SZ': 1900, '600750.SH': 1100, '600600.SH': 300, '300653.SZ': 1100, '002572.SZ': 1600, '300770.SZ': 600, '002568.SZ': 1300, '300693.SZ': 800, '300963.SZ': 2800, '300882.SZ': 1300, '600513.SH': 3100, '600660.SH': 600, '688663.SH': 1200, '603229.SH': 2500, '601088.SH': 700, '002677.SZ': 2800, '601658.SH': 5600, '688566.SH': 1000, '603170.SH': 1800, '300820.SZ': 500}
	'''读取目标仓位 字典格式 品种代码:持仓股数, 可以读本地文件/数据库，当前在代码里写死'''
	A.final_dict = my_targets
	'''设置交易账号 acount accountType是界面上选的账号 账号类型'''
	A.acct = account
	A.acct_type = accountType
	#定时器 定时触发指定函数
	C.run_time("f","5nSecond","2024-02-21 09:30:00","SH")


def f(C):
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
					passorder(23,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
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

### 等权调仓
```python
#encoding:gbk

import pandas as pd
import numpy as np
import time
from datetime import timedelta,datetime 

# 定义全局控制开关
buy_flag = True
sell_flag = True

target_stocks = ['601012.SH', '300331.SZ', '002507.SZ', '300347.SZ', '000739.SZ', '603182.SH', '301029.SZ', '002372.SZ', '603867.SH', '600233.SH', '603816.SH', '603801.SH', '603908.SH', '601865.SH', '603056.SH', '300995.SZ', '603578.SH', '600690.SH', '600258.SH', '002690.SZ', '603709.SH', '688526.SH', '603600.SH', '300626.SZ', '600036.SH', '002001.SZ', '600905.SH', '000651.SZ', '603899.SH', '605009.SH', '002677.SZ', '300996.SZ', '603305.SH', '688080.SH', '603650.SH', '601117.SH', '002236.SZ', '300009.SZ', '300957.SZ', '688239.SH', '300776.SZ', '002601.SZ', '603229.SH', '688777.SH', '002311.SZ', '600030.SH', '002801.SZ', '001222.SZ', '300395.SZ', '300899.SZ', '002415.SZ', '300586.SZ', '603112.SH', '300929.SZ', '600674.SH', '601318.SH', '300753.SZ', '300986.SZ', '300653.SZ', '601717.SH', '300938.SZ', '300069.SZ', '300316.SZ', '002871.SZ', '603515.SH', '003033.SZ', '600600.SH', '000913.SZ', '301273.SZ', '600379.SH', '605222.SH', '603507.SH', '002594.SZ', '002304.SZ', '301197.SZ', '600513.SH', '002274.SZ', '300144.SZ', '688722.SH', '603866.SH', '601088.SH', '002034.SZ', '600031.SH', '603368.SH', '300535.SZ', '600377.SH', '603057.SH', '300963.SZ', '002497.SZ', '600048.SH', '600760.SH', '300920.SZ', '600584.SH', '600346.SH', '600873.SH', '601985.SH', '002683.SZ', '605133.SH', '600750.SH', '301027.SZ', '600886.SH', '603439.SH', '002916.SZ', '600309.SH', '601006.SH', '603156.SH', '002968.SZ', '300854.SZ', '600660.SH', '000338.SZ', '603027.SH', '300887.SZ', '601607.SH', '300923.SZ', '000858.SZ', '300823.SZ', '002867.SZ', '603444.SH', '301113.SZ', '688078.SH', '300615.SZ', '300947.SZ', '300575.SZ', '603808.SH', '603085.SH', '300700.SZ', '000513.SZ', '603811.SH', '600398.SH', '002049.SZ', '300861.SZ', '600371.SH', '300417.SZ', '600479.SH', '300059.SZ', '000661.SZ', '600754.SH', '002737.SZ', '000915.SZ', '002223.SZ', '000895.SZ', '603529.SH', '300450.SZ', '000538.SZ', '603170.SH', '603966.SH', '600362.SH', '002572.SZ', '000333.SZ', '300416.SZ', '601958.SH', '300775.SZ', '601100.SH', '301119.SZ', '600887.SH', '002555.SZ', '600085.SH', '600531.SH', '600741.SH', '600481.SH', '601166.SH', '000014.SZ', '001211.SZ', '601888.SH', '300413.SZ', '688029.SH', '002258.SZ', '600455.SH', '600900.SH', '600298.SH', '603927.SH', '601658.SH', '001215.SZ', '003006.SZ', '002853.SZ', '603901.SH', '600585.SH', '603355.SH']

target_total_value = 4300000

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
	target_stocks = list(set(target_stocks))
	'''读取目标仓位 字典格式 品种代码:持仓股数, 可以读本地文件/数据库，当前在代码里写死'''
	A.final_dict = {}
	'''设置交易账号 acount accountType是界面上选的账号 账号类型'''
	A.acct = account
	A.acct_type = accountType
	#定时器 定时触发指定函数
	C.run_time("f","5nSecond","2024-02-21 09:30:00","SH")


def f(C):
	
	ticks = C.get_full_tick(target_stocks)
	nstocks = len(target_stocks)
	ooo_value = target_total_value / (nstocks * 1.0)
	for key in target_stocks:
		if key not in A.final_dict:
			try:
				ooo_price = ticks[key]["lastPrice"]
				ooo_vol = max(int(ooo_value / ooo_price / 100) * 100, 100)
				A.final_dict[key] = ooo_vol
			except Exception as e:
				print("异常 {} {}".format(key, e))
		else:
			continue
	print("nstocks: {}".format(nstocks))
	print("single_value: {:.2f}".format(ooo_value))
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
					passorder(23,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
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

### 加权调仓
```python
#encoding:gbk

import pandas as pd
import numpy as np
import time
from datetime import timedelta,datetime

# 定义全局控制开关
buy_flag = True
sell_flag = True

target_rate = {"000333.SZ":50, "601985.SH":30, "600887.SH":20}
target_total_value = 300000

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
					passorder(23,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
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
### 加权调仓（跳过主观交易持仓）
```python
#encoding:gbk

import pandas as pd
import numpy as np
import time
from datetime import timedelta,datetime

# 定义全局控制开关
buy_flag = True
sell_flag = True

active_code = ["600036.SH", "002142.SH"]

target_rate = {'002802.SZ': 0.04, '001234.SZ': 0.04, '300907.SZ': 0.04, '003023.SZ': 0.04, '688517.SH': 0.04, '605189.SH': 0.04, '001209.SZ': 0.04, '300743.SZ': 0.04, '002968.SZ': 0.04, '603908.SH': 0.04, '300515.SZ': 0.04, '002942.SZ': 0.04, '301003.SZ': 0.04, '688314.SH': 0.04, '003018.SZ': 0.04, '300897.SZ': 0.04, '301023.SZ': 0.04, '603681.SH': 0.04, '300950.SZ': 0.04, '301203.SZ': 0.04, '603201.SH': 0.04, '301077.SZ': 0.04, '002825.SZ': 0.04, '001296.SZ': 0.04, '605080.SH': 0.04}
target_total_value = 500000

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
					passorder(23,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
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

### 可转债调仓
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

target_rate = {"113052.SH" : 0.2, "110059.SH" : 0.2, "113042.SH" : 0.2, "113050.SH" : 0.2, "110079.SH" : 0.2}
target_total_value = 20000

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
				ooo_vol = max(int(tvalue / ooo_price / 10) * 10, 10)
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
	# print(full_tick)

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
	print(A.waiting_dict)
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
			#与目标数量差值小于10股的品种 停止委托
			if abs(position_dict[stock] - target_vol) < 10:
				print(f"{stock} {C.get_stock_name(stock)} 目标持仓{target_vol} 当前持仓{position_dict[stock]} 差额小于10 停止委托")
				continue
			# 允许买入
			if sell_flag:
				#持仓大于目标持仓 卖出
				if position_dict[stock]>target_vol:
					vol = int((position_dict[stock] - target_vol)/10)*10
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
					vol = int((target_vol-position_dict[stock])/10)*10
					#获取卖一价
					if stock not in full_tick:
						print("没有该股票[{}]tick数据，跳过...".format(stock))
					sell_one_price = full_tick[stock]['askPrice'][0]
					print(full_tick[stock])
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
					#对手价买入
					print("要买啦: {}".format(stock))
					passorder(23,1101,A.acct,stock,14,-1,vol,'对冲',2,msg,C)
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
