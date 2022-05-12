---
layout:     post
title:      基于python的期权交易系统-交易Gateway
subtitle:   借鉴CTP接口的交易Gateway设计
date:       2022-05-11
author:     YuJun
header-img: img/post-bg-options-insights.webp
catalog: true
tags:
    - python, pyside2, option
---



>对于交易Gateway基类，同样也是借鉴了CTP的交易接口，设计为“调用类(api)”和“回调类(spi)”两个部分。为了连接不同的交易接口（或通道）可以派生出相应的
>子类来进行对接。

## 交易Gateway基类

交易Gateway的api类主要功能是账户登录、登出，订单委托、撤单等。

```python
class CHessTraderApi(QObject):
    """交易api接口基类"""

    def __init__(self):
        super().__init__()
        self._spi = None
        self._MaxOrderRef = datetime.datetime.now().strftime("%m%d%H%M%S") + "000"
        self._CurrentOrderRefBase = int(self._MaxOrderRef[: 10])

        self._brokerid: str = ''
        self._userid: str = ''

        self._lock = threading.Lock()

    @property
    def BrokerID(self):
        return self._brokerid

    @BrokerID.setter
    def BrokerID(self, broker_id: str):
        self._brokerid = broker_id

    @property
    def UserID(self):
        return self._userid

    @UserID.setter
    def UserID(self, user_id: str):
        self._userid = user_id

    def _set_inputorder_ids(self, input_order: CHessInputOrderField):
        """
        设置输入报单的BrokerID,InvestorID,UserID,OrderRef
        :param input_order:
        :return:
        """
        input_order.BrokerID = self.BrokerID
        input_order.InvestorID = self.UserID
        input_order.UserID = self.UserID
        if not (hasattr(input_order, "OrderRef") and input_order.OrderRef > '0'):
            input_order.OrderRef = self.GetAvailableOrderRef()
        input_order.OrderPriceType = CHessOrderPriceType.LimitPrice
        input_order.HedgeFlagType = CHessHedgeFlagType.Speculation

    def GetAvailableOrderRef(self) -> str:
        """获取可用的OrderRef"""

        with self._lock:
            order_ref_base = int(datetime.datetime.now().strftime("%m%d%H%M%S"))
            if order_ref_base == self._CurrentOrderRefBase:
                self._MaxOrderRef = str(int(self._MaxOrderRef) + 1)
            else:
                self._CurrentOrderRefBase = order_ref_base
                self._MaxOrderRef = str(self._CurrentOrderRefBase) + "000"
            return self._MaxOrderRef

    def Init(self):
        """
        初始化
        初始化运行环境，只有调用后，接口才开始工作
        :return:
        """
        pass

    def RegisterSpi(self, spi: CHessTraderSpi):
        """
        注册一个派生自CHessTraderSpi接口类的实例，该实例将完成事件处理
        :param spi: 实现了CHessTraderSpi接口的实例
        :return:
        """
        self._spi = spi

    def ReqUserLogin(self, reqUserLoginField: CHessReqUserLoginField, nRequestID: int):
        """
        用户
        :param reqUserLoginField:
        :param nRequestID:
        :return:
        """
        pass

    def ReqUserLogout(self, userLogout: CHessUserLogoutField, nRequestID: int):
        """
        登出请求
        --------
        :param userLogout: 用户登出请求信息
        :param nRequestID: 请求ID
        :return:
        """
        pass

    def ReqOrderInsert(self, inputOrder: CHessInputOrderField, nRequestID: int):
        """
        报单录入请求，录入错误时对应响应HessOnRspOrderInsert, HessOnErrRtnOrderInsert，正确时对应回报HessOnRtnOrder,
        HessOnRtnTrade。可录入限价单、市价单、条件单等交易所支持的指令，撤单时使用ReqOrderAction.
        --------
        :param inputOrder: 输入报单
        :param nRequestID: 请求ID，对应响应里的nRequestID，无递增规则，由用户自行维护。
        :return:
        """
        self._set_inputorder_ids(inputOrder)

    def ReqOrdersInsert(self, inputOrders: Iterable, nRequestID: int):
        """多个报单录入请求"""
        for inputOrder in inputOrders:
            self.ReqOrder(inputOrder, nRequestID)

    def ReqOrder(self, inputOrder: CHessInputOrderField, nRequestID: int):
        """
        单个报单录入请求入口，对于有单笔最大委托数量的品种会拆分报单
        :param inputOrder: 输入报单
        :param nRequestID: 请求ID
        :return:
        """
        if Utils.SecuType(inputOrder.InstrumentID) == CHessSecuType.Option and \
                inputOrder.Volume > GlobalVar.getVar("OptMaxOrderVol"):
            inputOrders = []
            while inputOrder.Volume > GlobalVar.getVar("OptMaxOrderVol"):
                splitedOrder = copy.deepcopy(inputOrder)
                splitedOrder.OrderRef = self.GetAvailableOrderRef()
                splitedOrder.Volume = GlobalVar.getVar("OptMaxOrderVol")
                inputOrders.append(splitedOrder)

                inputOrder.Volume -= GlobalVar.getVar("OptMaxOrderVol")

            inputOrders.append(inputOrder)
            self.ReqOrdersInsert(inputOrders, nRequestID)
        else:
            self.ReqOrderInsert(inputOrder, nRequestID)

    def ReqOrderAction(self, inputOrderAction: CHessInputOrderActionField, nRequestID: int):
        """
        报单操作请求，错误响应：HessOnRspOrderAction, HessOnErrRtnOrderAction
                   正确响应：HessOnRtnOrder
        --------
        :param inputOrderAction: 输入报单操作
        :param nRequestID: 请求ID，对应响应里的nRequestID，无递增规则，由用户自行维护。
        :return:
        """
        pass
```

交易Gateway的spi类的主要功能是处理交易接口的推送信息，包括登录、登出的返回信息，报单回报以及成交回报。在spi类里会定义相应的信号，处理完收到的推送
信息后，会发送相应的信号。

```python
class CHessTraderSpi(QObject):
    """交易spi接口"""

    # 定义信号

    sigOnRspUserLogin = Signal(CHessRspUserLoginField, CHessRspInfoField, int, bool)
    sigOnRspUserLogout = Signal(CHessUserLogoutField, CHessRspInfoField, int, bool)
    sigOnRspOrderInsert = Signal(CHessInputOrderField, CHessRspInfoField, int, bool)
    sigOnRspOrderAction = Signal(CHessInputOrderActionField, CHessRspInfoField, int, bool)
    sigOnRtnOrder = Signal(CHessOrderField)
    sigOnRtnTrade = Signal(CHessTradeField)

    def __init__(self, api):
        """
        初始化spi接口
        --------
        :param api: CHessTradeApi类
        """
        super().__init__()
        self._api = api
        self.orderbook_header = ['BrokerID', 'InvestorID', 'ExchangeID', 'InstrumentID', 'OrderPriceType', 'Direction',
                                 'LimitPrice', 'VolumeTotalOriginal', 'OpenClose', 'HedgeFlagType', 'OrderStatus',
                                 'OrderSubmitStatus', 'VolumeTraded', 'VolumeTotal', 'TradedValue', 'InsertDate',
                                 'InsertTime', 'UpdateTime', 'CancelTime', 'RequestID', 'OrderSysID', 'FrontID',
                                 'SessionID', 'StatusMsg']
        # 报单簿, 类型为pd.DataFrame, index为OrderRef
        # 该报单簿维护所有本接口发出去的订单

        self.dfOrderBook = pd.DataFrame(columns=self.orderbook_header)
        # 佣金表, {ExchangeID_i: {SecuType_i: {'ratio': r, 'type': CHessCommissionRatioType}}}

        self._dictCommissionTable: dict = None

    @property
    def CommissionTable(self) -> dict:
        """返回佣金表"""
        return self._dictCommissionTable

    @CommissionTable.setter
    def CommissionTable(self, commission_table: dict) -> None:
        """设置佣金表"""
        self._dictCommissionTable = commission_table

    def HessOnRspUserLogin(self, rspUserLoginInfo: CHessRspUserLoginField, rspInfo: CHessRspInfoField, nRequestID: int,
                           bIsLast: bool):
        """
        登录请求响应，当执行ReqUserLogin后，该方法被调用
        --------
        :param rspUserLoginInfo: 用户登录应答
        :param rspInfo: 响应信息
        :param nRequestID: 返回用户操作请求的ID，该ID由用户在操作请求时指定
        :param bIsLast: 指示该次返回是否为针对nRequestID的最后一次返回
        :return:
        """
        if rspInfo.ErrorID == "NO_ERR":
            str_log = "账户{UserID}登录成功。".format(UserID=rspUserLoginInfo.UserID)
        else:
            str_log = "账户{UserID}登录失败，{ErrorMsg}".format(UserID=rspUserLoginInfo.UserID,
                                                         ErrorMsg=rspInfo.ErrorMsg)
        HessLog.getLogger("HessApiLogger").critical(str_log)
        self.sigOnRspUserLogin.emit(rspUserLoginInfo, rspInfo, nRequestID, bIsLast)

    def HessOnRspUserLogout(self, rspUserLogoutInfo: CHessUserLogoutField, rspInfo: CHessRspInfoField, nRequestID: int,
                            bIsLast: bool):
        """
        登出请求响应，当调用ReqUserLogout后，该方法被调用
        --------
        :param rspUserLogoutInfo: 用户登出请求
        :param rspInfo: 响应信息
        :param nRequestID: 返回用户操作请求的ID，该ID由用于在操作请求时指定
        :param bIsLast: 指示该次返回是否为针对nRequestID的最后一次返回
        :return:
        """
        self.sigOnRspUserLogout.emit(rspUserLogoutInfo, rspInfo, nRequestID, bIsLast)

    def HessOnRspOrderInsert(self, inputOrder: CHessInputOrderField, rspInfo: CHessRspInfoField, nRequestID: int,
                             bIsLast: bool):
        """
        报单录入请求响应，当执行ReqOrderInsert后有字段填写不对之类的报错，通过此接口返回
        --------
        :param inputOrder: 输入报单
        :param rspInfo: 响应信息
        :param nRequestID: 返回用户操作请求的ID，该ID由用于在操作请求时指定
        :param bIsLast: 指示该次返回是否为针对nRequestID的最后一次返回
        :return:
        """
        self._LogRspOrderInsert(inputOrder, rspInfo)
        self.sigOnRspOrderInsert.emit(inputOrder, rspInfo, nRequestID, bIsLast)

    def HessOnErrRtnOrderInsert(self, inputOrder: CHessInputOrderField, rspInfo: CHessRspInfoField):
        """
        报单录入错误回报, 当执行ReqOrderInsert后有字段填写不对之类的错误, 通过此接口返回
        --------
        :param inputOrder: 输入报单
        :param rspInfo: 响应信息
        :return:
        """
        self.sigOnErrRtnOrderInsert.emit(inputOrder, rspInfo)

    def HessOnRspOrderAction(self, inputOrderAction: CHessInputOrderActionField, rspInfo: CHessRspInfoField,
                             nRequestID: int, bIsLast: bool):
        """
        报单操作请求响应，当执行ReqOrderAction后有字段填写不对之类的报错，则通过此接口返回
        --------
        :param inputOrderAction: 输入报单操作
        :param rspInfo: 响应信息
        :param nRequestID: 返回用户操作请求的ID，该ID由用于在操作请求时指定
        :param bIsLast: 指示该次返回是否为针对nRequestID的最后一次返回
        :return:
        """
        self._LogRspOrderAction(inputOrderAction, rspInfo)
        self.sigOnRspOrderAction.emit(inputOrderAction, rspInfo, nRequestID, bIsLast)

    def HessOnRtnOrder(self, rtnOrder: CHessOrderField):
        """
        报单通知，当执行ReqOrderInsert后并报出后，收到返回则调用此接口
        --------
        :param rtnOrder: 报单
        :return:
        """
        self._LogRtnOrder(rtnOrder)
        self._update_orderbook(rtnOrder)
        self.sigOnRtnOrder.emit(rtnOrder)

    def HessOnRtnTrade(self, rtnTrade: CHessTradeField):
        """
        成交通知，报单发出后有成交则通过此接口返回
        --------
        :param rtnTrade: 成交回报
        :return:
        """
        rtnTrade.Commission = self._CalcCommission(rtnTrade)
        self._LogRtnTrade(rtnTrade)
        self._update_rtnTrade(rtnTrade)
        self.sigOnRtnTrade.emit(rtnTrade)

    def _CalcCommission(self, trade_field: CHessTradeField) -> float:
        """
        计算佣金
        --------
        :param trade_field: 成交回报
        :return:
        """

        secu_type = Utils.SecuType(trade_field.InstrumentID)
        if (trade_field.Direction, trade_field.OpenClose) not in \
                self._dictCommissionTable[trade_field.ExchangeID][secu_type]["commission"]:
            return 0.0

        fCommissionRatio = self._dictCommissionTable[trade_field.ExchangeID][secu_type]["commission"][(trade_field.Direction, trade_field.OpenClose)]
        if self._dictCommissionTable[trade_field.ExchangeID][secu_type]["type"] == CHessCommissionRatioType.ByMoney:
            fCommission = trade_field.Amount * fCommissionRatio
        elif self._dictCommissionTable[trade_field.ExchangeID][secu_type]["type"] == CHessCommissionRatioType.ByVolume:
            fCommission = trade_field.Volume * fCommissionRatio
        else:
            fCommission = 0.0
        return fCommission

    def _update_orderbook(self, rtnOrder: CHessOrderField):
        """
        更新订单簿
        --------
        :param rtnOrder: 报单
        :return:
        """
        order_ref = rtnOrder.OrderRef
        if order_ref not in self.dfOrderBook.index:
            new_order = pd.Series(name=order_ref)
            new_order['BrokerID'] = rtnOrder.BrokerID
            new_order['InvestorID'] = rtnOrder.InvestorID
            new_order['ExchangeID'] = rtnOrder.ExchangeID
            new_order['InstrumentID'] = rtnOrder.InstrumentID
            new_order['OrderPriceType'] = rtnOrder.OrderPriceType
            new_order['Direction'] = rtnOrder.Direction
            new_order['LimitPrice'] = rtnOrder.LimitPrice
            new_order['VolumeTotalOriginal'] = rtnOrder.VolumeTotalOriginal
            new_order['OpenCloser'] = rtnOrder.OpenClose
            new_order['HedgeFlagType'] = rtnOrder.HedgeFlagType
            new_order['OrderStatus'] = rtnOrder.OrderStatus
            new_order['OrderSubmitStatus'] = rtnOrder.OrderSubmitStatus
            new_order['VolumeTraded'] = rtnOrder.VolumeTraded
            new_order['VolumeTotal'] = rtnOrder.VolumeTotal
            new_order['TradedValue'] = 0
            new_order['InsertDate'] = rtnOrder.InsertDate
            new_order['InsertTime'] = rtnOrder.InsertTime
            new_order['UpdateTime'] = rtnOrder.UpdateTime
            new_order['CancelTime'] = rtnOrder.CancelTime
            new_order['RequestID'] = rtnOrder.RequestID
            new_order['OrderSysID'] = rtnOrder.OrderSysID
            new_order['FrontID'] = rtnOrder.FrontID
            new_order['SessionID'] = rtnOrder.SessionID
            new_order['StatusMsg'] = rtnOrder.StatusMsg

            self.dfOrderBook = self.dfOrderBook.append(other=new_order, ignore_index=False)
        else:
            # 正常交易、撤单成功或被交易所拒单

            self.dfOrderBook.loc[order_ref, 'OrderStatus'] = rtnOrder.OrderStatus
            self.dfOrderBook.loc[order_ref, 'OrderSubmitStatus'] = rtnOrder.OrderSubmitStatus
            self.dfOrderBook.loc[order_ref, 'VolumeTraded'] = rtnOrder.VolumeTraded
            self.dfOrderBook.loc[order_ref, 'VolumeTotal'] = rtnOrder.VolumeTotal
            self.dfOrderBook.loc[order_ref, 'UpdateTime'] = rtnOrder.UpdateTime
            self.dfOrderBook.loc[order_ref, 'CancelTime'] = rtnOrder.CancelTime
            self.dfOrderBook.loc[order_ref, 'OrderSysID'] = rtnOrder.OrderSysID
            self.dfOrderBook.loc[order_ref, 'StatusMsg'] = rtnOrder.StatusMsg

    def _update_rtnTrade(self, rtnTrade: CHessTradeField):
        """
        更新成交回报
        --------
        :param rtnTrade: 成交回报
        :return:
        """
        order_ref = rtnTrade.OrderRef
        if order_ref in self.dfOrderBook.index:
            self.dfOrderBook.loc[order_ref, 'TradedValue'] += rtnTrade.Amount
```

以上就是交易Gateway基类的设计，在下一篇中我们会基于此派生实现一个模拟交易的Gateway类，用于策略的模拟交易。