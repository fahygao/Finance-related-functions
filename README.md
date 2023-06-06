# Finance related Functions
- [1. Stock Alert](#1-Stock-Alert)
    - [1.1. Request all session last price](#11-Request-all-session-last-price)
    - [1.2. Request a historical close price](#12-Request-a-historical-close-price)
- [2. Automate SFTP](#2-Automate-SFTP)
- [3. Option Expiration](#2-Option-Expiration)


## 1. Stock Alert
refs: 
1. [Bloomberg Handbook](https://github.com/fahygao/Finance-related-functions/blob/main/blpapi-developers-guide-1.38.pdf)
2. [Excel Bloomberg APIs](https://github.com/fahygao/Finance-related-functions/blob/main/BQL%20for%20AIM%20(1)%20(3)%20(1).xlsx)
3. [BQLapi](https://github.com/fahygao/Finance-related-functions/blob/main/Current%20Day%20AIM%20Holdings%20RMON_add%20(2).docx)

Here is a general documentation to show how to pull stock alert from Bloomberg using BQL. 

Motivation: Bloomberg documentation is written in C++, and blpapi do not have examples of how to use the function keys to request real-time price and etc. When I first created the bloomberg api to request the last price, it took me a long time to understand the process of sending the right request. Therefore, I think that it might be helpful to list out my approach and see if it can help others.


Functions
---

### 1.1 Request all session last price
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

### 1.2 Request a historical close price
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
## 2. Automate SFTP
ref: [SFTP from Medium](https://medium.com/@keagileageek/paramiko-how-to-ssh-and-file-transfers-with-python-75766179de73)

SFTP (Secure File Transfer Protocol) is widely used in the finance industry for secure and reliable data transmission between financial institutions, clients, and partners. Manually uploading or downloading file from SFTP may involve human error, such as sending duplicated allocations, downloading the wrong statement and so on. Therefore, automating the process can help prevent the human error and improve the accuracy and efficiency.

```Python  
from datetime import datetime 
import paramiko #use for SFTP
def upload_sftp(filepath_quarry):
    """Upload allocation file to **** sftp"""
    HOSTNAME = "<ftp..com>"
    USERNAME = "<@.com>"
    PASSWORD = "<PASS>"
    name_file = "CUSTOM_NAME"+datetime.now().strftime("%y%m%d_%H%M%S")+".xlsx"

    client=paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(hostname=HOSTNAME, username=USERNAME, password=PASSWORD,allow_agent=False,look_for_keys=False)
    ftp=client.open_sftp()
    ftp.close()
    print("Uploaded the allocation file @",str(datetime.now()))
```

## 3. Option Expiration

Even though Bloomberg MAV can display the '% From Strike;' or '$ From Strike' and we can combine with MLRT (market alert) to play alerts for certain options, it is hard for us to monitor all the changes across many options. This Option expiration program specific focus on giving alerts of the change in options and let traders know if it is in the money, at the money or out of money during the daily decision making process.

We need to combine with the real time stock price using bloomberg api for this program.

Please see [Option Expiration Program]() for details. 


