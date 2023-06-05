# StockAlert
Here is a general documentation to show how to pull stock alert from Bloomberg using BQL

Motivation: Bloomberg documentation is written in C++, and blpapi do not have examples of how to use the function keys to request real-time price and etc. When I first created the bloomberg api to request the last price, it took me a long time to understand the process of sending the right request. Therefore, I think that it might be helpful to list out my approach and see if it can help others.



Functions
---
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
