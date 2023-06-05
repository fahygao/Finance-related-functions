# StockAlert
Here is a general documentation to show how to pull stock alert from Bloomberg using BQL

Motivation: Bloomberg documentation is written in C++, and blpapi do not have examples of how to use the function keys to request real-time price and etc. When I first created the bloomberg api to request the last price, it took me a long time to understand the process of sending the right request. Therefore, I think that it might be helpful to list out my approach and see if it can help others.



Functions
---

**Request the all session last price**
We can also use this to get today's high and low. Please uncomment ```print(msg)``` to see more attributes.

``` Python 
# all session price program

def allsession_lp(ticker):
    """Return the last price after market or pre-market
    ticker: the bloomberg ticker name. 
    return: the last price using all session"""

    SECURITY_DATA = blpapi.Name("securityData")
    FIELD_DATA = blpapi.Name("fieldData")
    subs = SubscriptionList()
    corr=blpapi.CorrelationId(12)
    subs.add(ticker,
     "LAST_ALL_SESSIONS",
     correlationId=corr)
    session=blpapi.Session()
    options = blpapi.SessionOptions()
    options.setServerHost('localhost')
    options.setServerPort(8194)
    session = blpapi.Session(options)
    session.start()
    session.openService("//blp/mktbar")
    session.subscribe(subs)
    last_price_f=0
    while True:
        ev = session.nextEvent()
        for msg in ev:
            try:
#                 print(msg)
                last_price=msg.getElement('LAST_ALL_SESSIONS')[0]
                if type(last_price) == float:
                    last_price_f = last_price
                    session.stop()
                    return last_price_f
            except:
                pass

    session.stop()

    return last_price_f
```

**Request a historical close price**
It is a bit tricky than requesting the all session price, since we need to take UTC into consideration and use different 

``` Python 
def date_convert_UTC(dt):
    """Conver timezone to UTC since Bloomberg stores the traded date in UTC"""
    from dateutil import tz
    # METHOD 1: Hardcode zones:
    from_zone = tz.gettz('America/New_York')
    to_zone = tz.gettz('UTC')
    utc = dt.replace(tzinfo=from_zone)

    # Convert time zone
    central = utc.astimezone(to_zone)
    return central


def close_price(ticker, time_interval: int = 1):
    """Check the close price of a bloomberg ticker for the particular day
    ticker: ticker name, e.g. TSLA US EQUITY
    time_interval: the length of days that you want to check 

    return: closing price of the particular day 
    """
    import time 
    import pytz
    import blpapi
    from datetime import datetime
    date = datetime.now()
    start=pd.to_datetime(date)-pd.Timedelta(days=time_interval)
    end=pd.to_datetime(date)#pd.to_datetime(date)#+pd.Timedelta(days=1)
    start=date_convert_UTC(start)
    end=date_convert_UTC(end)
    SECURITY_DATA = blpapi.Name("securityData")
    FIELD_DATA = blpapi.Name("fieldData")
    session=blpapi.Session()
    options = blpapi.SessionOptions()
    options.setServerHost('localhost')
    options.setServerPort(8194)
    session = blpapi.Session(options)
    session.start()
    session.openService("//blp/refdata")
    service=session.getService("//blp/refdata")
    request=service.createRequest("IntradayBarRequest")
    request.set("security", ticker)
    request.set("startDateTime", start)
    request.set("endDateTime", end)
    request.set("interval", 1)
    request.set("eventType", "TRADE")
    corr=blpapi.CorrelationId(10)
    session.sendRequest(request,correlationId=corr)
    n =0
    close_p =0
    while True:
        ev = session.nextEvent()
        for msg in ev:
            try:
                data=msg.getElement('barData').getElement('barTickData')
                num=data.numValues()
            except:
                pass
        if ev.eventType() == blpapi.Event.RESPONSE:
            break
    close=[]
    for i in range(num):
        """original code"""
        """"""
        close.append(data[i]['close'])
    session.stop()
    return close[-1]
```
