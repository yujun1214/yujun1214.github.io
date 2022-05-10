---
layout:     post
title:      基于python的期权交易系统-行情Gateway
subtitle:   借鉴CTP接口的行情Gateway设计
date:       2022-05-08
author:     YuJun
header-img: img/post-bg-options-insights.webp
catalog: true
tags:
    - python, pyside2, option
---



>对于交易系统来说，行情是重要的基础。我们借鉴CTP行情接口，设计行情Gateway基类，并实现了基于新浪接口的行情Gateway。同样的，也可实现其他行情源的
>Gateway。

## 行情Gateway基类

和CTP一样，我们把行情Gateway设计成“调用类(api)”和“回调类(spi)”两个组成部分。api完成登录、登出、订阅、退订等任务，spi处理回调。

话不多说，直接上代码。下面是api的接口：

```python
from PySide2.QtCore import QObject

class CHessMdApi(QObject):
    """行情api接口基类"""
    def __init__(self):
        super().__init__()
        self._spi: CHessMdSpi = None    # 行情spi类
        self._SubscribedSecues = []     # 已订阅的合约代码列表
        self._brokerid: str = ''        # 经纪公司代码
        self._userid: str = ''          # 用户代码

    def Init(self):
        """
        初始化运行环境，只有调用后接口才开始工作
        """
        pass

    def RegisterSpi(self, spi: CHessMdSpi):
        """注册spi回调接口"""
        self._spi = spi

    def ReqUserLogin(self, reqUserLoginField: CHessReqUserLoginField, nRequestID: int):
        """
        用户登录请求
        --------
        :param reqUserLoginField: 用户登录请求信息
        :param nRequestID: 用户操作请求的ID
        """
        pass

    def ReqUserLogout(self, userLogout: CHessUserLogoutField, nRequestID: int):
        """
        登出请求
        --------
        :param userLogout:
        :param nRequestID:
        :return:
        """
        pass

    def SubscribeMarketData(self, lstInstrumentIDs: list, nCount: int):
        """
        订阅行情
        --------
        :param lstInstrumentIDs: 合约ID列表
        :param nCount: 合约数量
        :return:
        """
        pass

    def UnSubscribeMarketData(self, lstInstrumentIDs: list, nCount: int):
        """
        退订行情
        --------
        :param lstInstrumentIDs: 合约ID列表
        :param nCount: 合约数量
        :return:
        """
        pass
```

下面是spi的接口：

```python
import pandas as pd
from PySide2.QtCore import Signal

class CHessMdSpi(QObject):
    """行情spi回调接口基类"""

    # 定义信号
    sigOnRspUserLogin = Signal(CHessRspUserLoginField, CHessRspInfoField, int, bool)
    sigOnRtnDepthMarketDatas = Signal(pd.DataFrame)
    sigOnRtnMarketData = Signal()

    def __init__(self):
        """
        初始化spi接口
        """
        super().__init__()
        self._isLogin = False   # 是否登录成功

    def HessOnRspUserLogin(self, rspUserLoginInfo: CHessRspUserLoginField, rspInfo: CHessRspInfoField, nRequestID: int,
                           bIsLast: bool):
        """
        登录请求响应, 当调用ReqUserLogin后, 该方法被调用
        --------
        :param rspUserLoginInfo: 用户登录应答
        :param rspInfo: 响应信息
        :param nRequestID: 返回用户操作请求的ID, 该ID由用户在操作请求时指定
        :param bIsLast: 指示该次返回是否为针对nRequestID的最后一次返回
        :return:
        """
        if rspInfo.ErrorID != "NO_ERR":
            HessLog.getLogger("HessApiLogger").warning(f"行情api登录失败，userid={rspUserLoginInfo.UserID}，"
                                                       f"{rspInfo.ErrorMsg}")
            self._isLogin = False
        else:
            HessLog.getLogger("HessApiLogger").info(f"行情api登录成功，userid={rspUserLoginInfo.UserID}。")
            self._isLogin = True

    def HessOnRtnDepthMarketData(self, dfDepthMarketData: pd.DataFrame):
        """
        深度行情通知, 当SubscribeMarketData订阅行情后, 行情通知由此推送。
        --------
        :param dfDepthMarketData: 深度行情
        :return:
        """
        pass
```

## 新浪行情Gateway

接下来就可以定义新浪行情接口Gateway了。因为不同证券类型从新浪接口返回（股票、股票期权、股指期货）的数据字段不太一样，所以先定义新浪行情Gateway的基类，不同证券
类型的行情接口从这个基类中派生出来：

```python
import threading
from PySide2.QtCore import QTimer, Slot
import datetime
import pandas as pd

class CSinaMDApi(CHessMdApi):
    """新浪行情api接口基类"""

    def __init__(self):
        super().__init__()
        self._SubscribedCodesString = ''  # 已订阅合约代码的连接字符串
        self._value_lock = threading.Lock()  # 互斥锁
        self.timer = QTimer(self)

    @classmethod
    def _StockCode2SinaCode(cls, secu_code: str, is_index: bool = False) -> str:
        """
        将股票代码转换为新浪接口的股票代码
        --------
        :param secu_code: 股票代码, e.g: 510050.SH
        :param is_index: 是否为指数
        :return: sh510050
        """
        secu_code = Utils.MktCode2Symbol(secu_code)
        if is_index:
            if secu_code.startswith('0'):
                return 'sh' + secu_code
            else:
                return 'sz' + secu_code
        else:
            if secu_code.startswith('5') or secu_code.startswith('6'):
                return 'sh' + secu_code
            else:
                return 'sz' + secu_code

    @classmethod
    def _SinaStockCode2MktCode(cls, sina_code: str) -> str:
        """
        将新浪接口的股票代码转换为市场代码
        --------
        :param sina_code: 新浪接口的股票代码, e.g: sh510050
        :param is_index: 是否为指数
        :return: 510050.SH
        """
        if (mkt_symbol := sina_code[: 2]) not in ("sh", "sz"):
            raise ValueError(f"{sina_code} is not a secu code of sina interface.")
        str_code = sina_code[2:]
        return str_code + '.' + mkt_symbol.upper()

    def _ConcatSubCodes(self):
        """
        将已订阅合约的代码连接成字符串，用于获取实时行情
        :return:
        """
        pass

    def Init(self):
        self.timer.timeout.connect(self.GetRealPrice)
        self.timer.start(1000)

    def ReqUserLogin(self, reqUserLoginField: CHessReqUserLoginField, nRequestID: int):
        """
        用户登录请求
        :param reqUserLoginField:
        :param nRequestID:
        :return:
        """
        rspUserLogin = CHessRspUserLoginField()
        rspUserLogin.TradingDay = datetime.date.today()
        rspUserLogin.LoginTime = datetime.datetime.now().time()
        rspUserLogin.BrokerID = reqUserLoginField.BrokerID
        rspUserLogin.UserID = reqUserLoginField.UserID
        rspUserLogin.FrontID = ''
        rspUserLogin.SessionID = ''
        rspUserLogin.MaxOrderRef = 0

        rspInfo = CHessRspInfoField()
        rspInfo.ErrorID = "NO_ERR"
        rspInfo.ErrorMsg = ''

        HessLog.getLogger("HessApiLogger").info(f"登录行情api，userid={self.UserID}.")
        self._spi.HessOnRspUserLogin(rspUserLogin, rspInfo, 0, True)

    def SubscribeMarketData(self, lstInstrumentIDs: list, nCount: int = 1):
        if not isinstance(lstInstrumentIDs, list):
            raise TypeError("'lstInstrumentIDS' should be list.")
        super().SubscribeMarketData(lstInstrumentIDs, nCount)
        for InstrumentID in lstInstrumentIDs:
            if InstrumentID not in self._SubscribedSecues:
                self._SubscribedSecues.append(InstrumentID)
        self._ConcatSubCodes()

    def UnSubscribeMarketData(self, lstInstrumentIDs: list, nCount: int = 1):
        for InstrumentID in lstInstrumentIDs:
            if InstrumentID in self._SubscribedSecues:
                self._SubscribedSecues.remove(InstrumentID)
        self._ConcatSubCodes()

    @Slot()
    def GetRealPrice(self):
        """
        从sina api接口获取已订阅合约的实时行情数据, 并通过spi接口返回
        该函数运行在单独的线程中, 每隔1秒返回一次数据
        :return:
        """
        pass

class CSinaMdSpi(CHessMdSpi):
    """新浪行情spi接口"""
    
    def __init__(self):
        super().__init__()

    def HessOnRtnDepthMarketData(self, dfDepthMarketData: pd.DataFrame):
        GlobalVar.updateMktData(dfDepthMarketData)
        self.sigOnRtnMarketData.emit()
```

这里关于行情的推送有一点需要说明，就是收到行情推送后，不直接广播行情的数据，而是把行情数据更新到行情全局变量中，然后再发送行情已更新的消息，由连接
的槽函数自己去取行情数据。这样可以节省拷贝行情数据的开销，特别是订阅的行情数量大、连接行情推送的槽函数多的情况，效果更明显。

### 新浪股票行情Gateway

新浪股票（含ETF）实时行情的接口为："https://hq.sinajs.cn/list=sh510050"，返回的数据如下：

```
var hq_str_sh510050="上证50ETF,2.687,2.703,2.677,2.699,2.661,2.676,2.677,690934326,1848751782.000,876100,2.676,
129500,2.675,175600,2.674,491900,2.673,918000,2.672,36935,2.677,341500,2.678,493500,2.679,1324900,2.680,653400,
2.681,2022-05-09,15:00:00,00,";

各个字段的含义：
var hq_str_sh510050=证券简称，今日开盘价，昨日收盘价，最近成交价，最高成交价，最低成交价，买入价，卖出价，成交数量，成交金额，买数量1，
买价1，买数量2，买价2，买数量3，买价3，买数量4，买价4，买数量5，买价5，卖数量1，卖价1，卖数量2，卖价2，卖数量3，卖价3，卖数量4，卖价4，
卖数量5，卖价5，行情日期，行情时间，停牌状态
```

新浪股票行情Gateway类：

```python
from requests import get
import pandas as pd
import re

class CSinaStockMdApi(CSinaMDApi):
    """新浪股票行情api接口"""

    def _ConcatSubCodes(self):
        with self._value_lock:
            self._SubscribedCodesString = ','.join([self._StockCode2SinaCode(Utils.MktCode2Symbol(code),
                                                                             Utils.IsIndex(code))
                                                    for code in self._SubscribedSecues])

    def GetRealPrice(self):
        mkt_header = ['InstrumentID', 'InstrumentName', 'OpenPrice', 'PreClosePrice', 'LastPrice', 'HighestPrice',
                      'LowestPrice', 'bid_price', 'ask_price', 'Volume', 'Turnover', 'BidVolume1', 'BidPrice1',
                      'BidVolume2', 'BidPrice2', 'BidVolume3', 'BidPrice3', 'BidVolume4', 'BidPrice4', 'BidVolume5',
                      'BidPrice5', 'AskVolume1', 'AskPrice1', 'AskVolume2', 'AskPrice2', 'AskVolume3', 'AskPrice3',
                      'AskVolume4', 'AskPrice4', 'AskVolume5', 'AskPrice5', 'TradingDay', 'UpdateTime',
                      'suspend_status']
        while True:
            with self._value_lock:
                url = "https://hq.sinajs.cn/list={code_str}".format(code_str=self._SubscribedCodesString)
            data = get(url, headers=SinaApiConst.HQ_REQUEST_HEADERS).content.decode('gbk')
            # 解析返回数据中的股票代码
            str_code_pattern = re.compile(r'hq_str_(.*?)=')
            stock_codes = str_code_pattern.findall(data)
            # 解析返回数据中的行情数据
            str_mkt_pattern = re.compile(r'=\"(.*?)\";')
            mkt_datas = str_mkt_pattern.findall(data)
            stock_mkt_datas = [[Utils.Symbol2MktCode(code)] + mkt_data.rstrip(',').split(',') for code,
                               mkt_data in zip(stock_codes, mkt_datas)]
            df_mkt_data = pd.DataFrame(stock_mkt_datas, columns=mkt_header, dtype=float)
            df_mkt_data["bid_volume"] = df_mkt_data["BidVolume1"]
            df_mkt_data["ask_volume"] = df_mkt_data["AskVolume"]
            df_mkt_data["UpdateTime"] = \
                df_mkt_data.apply(lambda x: x["TradingDay"] + ' ' + x["UpdateTime"], axis=1)
            self._spi.HessOnRtnDepthMarketData(df_mkt_data)
```

### 新浪股指期货行情Gateway

新浪股指期货实时行情接口为：”https://hq.sinajs.cn/list=CFE_RE_IH2205“，返回的数据如下：

```
var hq_str_CFF_RE_IH2205="2700.600,2710.800,2668.000,2683.800,31848,85632757.200,38082.000,2683.800,0.000,2985.800,
2443.000,0.000,0.000,2711.400,2714.400,40529.000,2685.800,1,0.000,0,0.000,0,0.000,0,0.000,0,2686.000,2,0.000,0,0.000,
0,0.000,0,0.000,0,2022-05-09,15:00:00,100,1,,,,,,,,,2688.795,上证50指数期货2205";

各个字段的含义（我们只取第1, 2, 3, 4, 5, 6, 7, 8, 10, 11, 14, 15, 16, 17, 18, 27, 28, 37, 38及最后一个字段）：
var hq_str_CFE_RF_IH2205=开盘价，最高价，最低价，最新价，成交量，成交额（未乘以乘数），持仓量，收盘价，涨停价，跌停价，前收盘价，
前结算价，前持仓量，买价1，卖数量1，卖价1，卖数量1，交易日，更新时间，合约简称
```

新浪股指期货行情Gateway：

```python
from requests import get
import pandas as pd
import re

class CSinaIdxFutureMdApi(CSinaMDApi):
    """新浪股指期货行情api接口"""

    def _ConcatSubCodes(self):

        with self._value_lock:
            self._SubscribedCodesString = ','.join(["CFF_RE_" + Utils.MktCode2Symbol(code)
                                                    for code in self._SubscribedSecues])

    def GetRealPrice(self):

        mkt_header = ["InstrumentID", "OpenPrice", "HighestPrice", "LowestPrice", "LastPrice", "Volume", "Turnover",
                      "OpenInterest", "ClosePrice", "UpperLimitPrice", "LowerLimitPrice", "PreClosePrice",
                      "PreSettlementPrice", "PreOpenInterest", "BidPrice1", "BidVolume1", "AskPrice1", "AskVolume1",
                      "TradingDay", "UpdateTime", "InstrumentName"]
        while True:
            with self._value_lock:
                url = "https://hq.sinajs.cn/list={code_str}".format(code_str=self._SubscribedCodesString)
            data = get(url, headers=SinaApiConst.HQ_REQUEST_HEADERS).content.decode("gbk")
            # 解析返回数据中的期指代码
            str_code_pattern = re.compile(r"CFF_RE_(.*?)=")
            idxFt_codes = str_code_pattern.findall(data)
            # 解析返回数据中的行情数据
            str_mkt_pattern = re.compile(r'=\"(.*?)\";')
            mkt_datas = str_mkt_pattern.findall(data)
            idxFt_mkt_datas = [[Utils.Symbol2MktCode(code)] + mkt_data.split(',')
                               for code, mkt_data in zip(idxFt_codes, mkt_datas)]
            used_columns = [0, 1, 2, 3, 4, 5, 6, 7, 8, 10, 11, 14, 15, 16, 17, 18, 27, 28, 37, 38, -1]
            df_idxFt_mkt_data = pd.DataFrame(idxFt_mkt_datas, dtype=float).iloc[:, used_columns]
            df_idxFt_mkt_data.columns = mkt_header

            df_idxFt_mkt_data["bid_price"] = df_idxFt_mkt_data["BidPrice1"]
            df_idxFt_mkt_data["bid_volume"] = df_idxFt_mkt_data["BidVolume1"]
            df_idxFt_mkt_data["ask_price"] = df_idxFt_mkt_data["AskPrice1"]
            df_idxFt_mkt_data["ask_volume"] = df_idxFt_mkt_data["AskVolume1"]
            df_idxFt_mkt_data["UpdateTime"] = \
                df_idxFt_mkt_data.apply(lambda x: x["TradingDay"] + ' ' + x["UpdateTime"], axis=1)
            self._spi.HessOnRtnDepthMarketData(df_idxFt_mkt_data)
```

### 新浪股票期权行情Gateway

新浪股票期权实时行情接口为：“https://hq.sinajs.cn/list=CON_OP_10004163”，返回的数据如下：

```
var hq_str_CON_OP_10004163="70,0.0421,0.0421,0.0422,1,149089,-27.29,2.7000,0.0579,0.0529,0.3282,0.0001,0.0426,82,
0.0425,86,0.0424,33,0.0423,26,0.0422,1,0.0421,70,0.0420,33,0.0419,91,0.0418,56,0.0417,11,2022-05-09 15:00:00,0,
E 01,EBS,510050,50ETF购5月2700,31.43,0.0549,0.0367,189902,85957119.00,M,0.0579,C,2022-05-25,16,1,0,0.0421";

各个字段的含义：
var hq_str_CON_OP_10004163=买量，买价，最新价，卖价，卖量，持仓量，涨幅，行权价，昨收价，开盘价，涨停价，跌停价, 
申卖 价五，申卖量五，申卖价四，申卖量四，申卖价三，申卖量三，申卖价二，申卖量二，申卖价一，申卖量一，申买价一，
申买量一，申买价二，申买量二，申买价三，申买量三，申买价四，申买量四，申买价五，申买量五，行情时间，主力合约标识，状态码， 
标的证券类型，标的股票，期权合约简称，振幅(38)，最高价，最低价，成交量，成交额，分红调整标志，昨结算价，认购认沽标志，
到期日，剩余天数，虚实值标志，内在价值，时间价值
```

新浪股票期权实时隐波、希腊字母接口为：“https://hq.sinajs.cn/list=CON_SO_10004163”，返回的数据如下：

```
var hq_str_CON_SO_10004163="50ETF购5月2700,,,,189902,0.4615,2.7049,-0.7123,0.2226,0.2261,0.0549,0.0367,
510050C2205M02700,2.7000,0.0421,0.0501,M";

各个字段的含义：
var hq_str_CON_SO_10004163=期权合约简称,,,,成交量,Delta,Gamma,Theta,Vega,隐含波动率,最高价,最低价,交易代码,行权价,最新价,理论价值
```

新浪股票期权行情Gateway：

```python
from requests import get
import pandas as pd
import re

class CSinaStockOptionMdApi(CSinaMDApi):
    """新浪股票期权行情api接口"""

    def _ConcatSubCodes(self):
        with self._value_lock:
            self._SubscribedCodesString = ','.join(['CON_OP_' + Utils.MktCode2Symbol(code)
                                                    for code in self._SubscribedSecues])

    def GetOptionGreeks(self) -> pd.DataFrame:
        """
        获取期权隐含波动率、希腊字母等信息
        :return:
        """
        greeks_header = ["InstrumentID", "InstrumentName", "Volume", "Delta", "Gamma", "Theta", "Vega", "ImpVol",
                         "HighestPrice", "LowestPrice", "TradeID", "strike", "LastPrice", "theoricvalue"]
        with self._value_lock:
            url = "https://hq.sinajs.cn/list={code_str}".format(code_str=self._SubscribedCodesString.replace("OP", "SO"))
        data = get(url, headers=SinaApiConst.HQ_REQUEST_HEADERS).content.decode("gbk")
        # 解析返回数据中的期权代码 #
        str_code_pattern = re.compile(r'SO_(.*?)=')
        opt_codes = str_code_pattern.findall(data)
        # 解析返回数据中的希腊字母、隐波数据 #
        str_greek_pattern = re.compile(r'=\"(.*?)\";')
        greek_datas = str_greek_pattern.findall(data)
        opt_greek_datas = [[Utils.Symbol2MktCode(code)] + greek_data.replace(",,,,", ",").split(",")[: -1]
                           for code, greek_data in zip(opt_codes, greek_datas)]
        df_greek_data = pd.DataFrame(opt_greek_datas, columns=greeks_header, dtype=float)
        df_greek_data["Vega"] = df_greek_data["Vega"] / 100
        df_greek_data["Theta"] = df_greek_data["Theta"] / 365
        return df_greek_data

    def GetRealPrice(self):
        mkt_header = ['InstrumentID', 'bid_volume', 'bid_price', 'LastPrice', 'ask_price', 'ask_volume', 'OpenInterest',
                      'increase', 'strike', 'PreClosePrice', 'OpenPrice', 'UpperLimitPrice', 'LowerLimitPrice',
                      'AskPrice5', 'AskVolume5', 'AskPrice4', 'AskVolume4', 'AskPrice3', 'AskVolume3', 'AskPrice2',
                      'AskVolume2', 'AskPrice1', 'AskVolume1', 'BidPrice1', 'BidVolume1', 'BidPrice2', 'BidVolume2',
                      'BidPrice3', 'BidVolume3', 'BidPrice4', 'BidVolume4', 'BidPrice5', 'BidVolume5', 'UpdateTime',
                      'iszl', 'status', 'underlyingtype', 'underlyingcode', 'InstrumentName', 'swing', 'HighestPrice',
                      'LowestPrice', 'Volume', 'Turnover', 'adjflag', 'PreSettlementPrice', 'callputflag', 'expireday',
                      'remainderDays', 'inoutmoneyflag', 'intrinsicvalue', 'timevalue']
        while True:
            with self._value_lock:
                url = "https://hq.sinajs.cn/list={code_str}".format(code_str=self._SubscribedCodesString)
            data = get(url, headers=SinaApiConst.HQ_REQUEST_HEADERS).content.decode('gbk')
            # 解析返回数据中的期权代码
            str_code_pattern = re.compile(r'OP_(.*?)=')
            opt_codes = str_code_pattern.findall(data)
            # 解析返回数据中的行情数据
            str_mkt_pattern = re.compile(r'=\"(.*?)\";')
            mkt_datas = str_mkt_pattern.findall(data)
            opt_mkt_datas = [[Utils.Symbol2MktCode(code)] + mkt_data.split(',')
                             for code, mkt_data in zip(opt_codes, mkt_datas)]
            df_mkt_data = pd.DataFrame(opt_mkt_datas, columns=mkt_header, dtype=float)
            df_greek_data = self.GetOptionGreeks()
            df_mkt_data = pd.merge(left=df_mkt_data,
                                   right=df_greek_data[["InstrumentID", "Delta", "Gamma", "Theta", "Vega", "ImpVol"]],
                                   how="outer", on="InstrumentID")
            df_mkt_data["TradingDay"] = df_mkt_data["UpdateTime"].apply(lambda x: x[:10])
            self._spi.HessOnRtnDepthMarketData(df_mkt_data)
```

至此，行情Gateway就讲完了，接下来的文章我们将会介绍“交易Gateway”。