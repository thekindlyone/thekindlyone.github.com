---
layout: post
title: "Fundraiser Analysis"
description: "Some plotly magic on a dataset"
category: Demo
tags: ["python","beautifulsoup","plotly","data","gmail","shirin","dalvi"]
thumbnail: "http://thekindlyone.github.io/assets/Exploration_files/Exploration_15_0.png"
---
{% include JB/setup %}


# Or how I refused to send 100 manual emails and made some charts instead

Sometime back I started a [fundraiser](https://milaap.org/campaigns/shirin-dalvi-urdu-journalist) for a journalist in trouble. 

Having finally reached the goal, the fundraising website - milaap emailed me saying I should email each donor **individually** to thank them for their contribution. While I am very grateful, sending ~150 manual emails is problematic.

So I decided to automate sending the emails.


## Getting the donor info

Milaap sends me emails everytime someone donates. Lets try to get the info from there. I use the [gmail library](https://github.com/charlierguo/gmail)


```python
from gmail import Gmail
from creds import * # file with my gmail credentials, strings username and password
```


```python
g = Gmail()
g.login(username, password)
```




    True




```python
mails=(g.inbox().mail(sender="admin@milaap.org",prefetch=True) 
       + g.inbox().mail(sender="campaigns@milaap.org",prefetch=True))
```

Parse these emails to get the info


```python
import re

usd_inr = 67.02 # conversion rate of USD to INR


# some helper functions
def amount2inr(currency,amount):
    amount = float(amount.replace(',',''))
    usd=False
    if currency.startswith('$'):
        amount*=usd_inr
        usd=True
    else:
        amount=float(amount)
    return amount,usd

def re_search(exp,body,default=None,group=1):
    try:
        rv = re.search(exp,body).group(group) if group else re.search(exp,body).groups()
    except:
        rv = default
    return rv

def parse_mail(mail):
    mail_exp=r"([a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+)"
    body = mail.body      
    currency,original_amount = re_search('contributed (Rs\.|\$)([0-9\.,]+?) to',
                                         mail.body.replace('\r\n',' '),
                                         group=None)
    amount,usd = amount2inr(currency,original_amount)
    name = re_search('(.+?) has contributed',mail.subject)
    if 'anonymous' in name.lower():
        name=None
        email=None
    else:
        email = re_search(mail_exp,body)
    data = dict(
            email = email,
            name = name ,
            amount = amount,
            timestamp = mail.sent_at.date(),
            usd = usd,
            original_amount = original_amount,
  
    )
    return data
data = [parse_mail(mail) for mail in mails[:-1] if 'contributed' in mail.body]
```

let's quickly email a thank you note to these good folk


```python
import smtplib
from email.MIMEMultipart import MIMEMultipart
from email.MIMEText import MIMEText
template= '''Dear {},
I would like to express my deepest gratitude to you for your donation to the Shirin Dalvi fundraising campaign. 
With help from you, we have reached our goal. :)

Good Day 

Best regards,
Aritra Das
'''.format
server = smtplib.SMTP('smtp.gmail.com', 587)
server.starttls()
fromaddr = username + "@gmail.com"
server.login(fromaddr, password)
for contact in data: 
    if contact['email']:
        toaddr = contact['email']
        name = contact['name']
        msg = MIMEMultipart()
        msg['From'] = fromaddr
        msg['To'] = toaddr
        msg['Subject'] = "Thanks for donating to the campaign"
        body = template(name)
        msg.attach(MIMEText(body, 'plain'))    
        text = msg.as_string()
        server.sendmail(fromaddr, toaddr, text)
server.quit()
```




    (221, '2.0.0 closing connection co6sm6121372pad.23 - gsmtp')



There, thank you notes dispatched.

-----------------------------------------------------------------------------------------
Now that we have all this data, lets plot some charts for fun. Library used: [plot.ly](https://github.com/plotly/plotly.py)


## Cumulative Collections over Time




Initialize plotly


```python
from plotly.graph_objs import Bar, Scatter, Figure, Layout, Pie
import plotly.plotly as py
```

Process the data to get a cumulative series


```python
from itertools import groupby
key=lambda x: x['timestamp']
cum=0.0
xs=[]
ys=[]
for date,contribs in groupby(sorted(data,key=key),key):
    datesum =sum(contrib['amount'] for contrib in contribs)
    cum+=datesum
    xs.append(date)
    ys.append(cum)
```

During the fundraising effort, various media outlets and 2 celebrities endorsed it. Lets mark those.


```python
from datetime import datetime
eventdict=dict(
        tarikh_fatah = datetime(year=2016,month=9,day=12).date(), 
        the_wire =  datetime(year=2016,month=9,day=8).date(),
        newslaundy = datetime(year=2016,month=8,day=12).date(),
        newslaundry2 = datetime(year=2016,month=8,day=19).date(),
        newsminute = datetime(year=2016,month=8,day=27).date(),
        ladiesfinger = datetime(year=2016,month=8,day=20).date(),
        varun_grover = datetime(year=2016,month=8,day=8).date(), 
        )
```

Lets plot this thing now!


```python
from itertools import cycle

colors=cycle('red blue green'.split())
colors2=cycle('red blue green'.split())



shapes=[{'type': 'line',
                        'x0': date,
                        'y0': 0,
                        'x1': date,
                        'y1': ys[-1],
                        'xref': 'x',
                        'yref': 'y',
                         'line': {
                                'width': 1,
                                'color': colors.next(),
                                }
                        } for event,date in sorted(eventdict.iteritems(),key=lambda items:items[1])]


annotations = [dict(text=event,   
                   x=date,                         
                   y=incy*50000,
                   showarrow=False,
                   yanchor='bottom',
                   font = dict(color=colors2.next())
                   )for incy,(event,date) in enumerate(sorted(eventdict.iteritems(),key=lambda items:items[1]))]


fig=Figure(
        data=[Scatter(x=xs,y=ys,)],
        layout=Layout(
                title="Collection of funds",
                xaxis=dict(title="time",),
                yaxis=dict(title="Total funds collected",tickprefix='Rs.'),
                annotations=annotations,                
                shapes=shapes,
            )
       )
py.iplot(fig)
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~thekindlyone/98.embed" height="525px" width="100%"></iframe>



Clearly this thing was dead without Tarek Fatah's endorsement. News publications didn't help at all.               
**Note to self: If you want public attention, get a celeb's attention first.**


## Donations by currency

The fundraising website took both USD and INR. 


```python
total = sum(d['amount'] for d in data)
total_donors = len(data)
total_donated_in_usd = sum(d['amount'] for d in data if d['usd'])
usd_donor_count = sum(1 for d in data if d['usd'])

fig = {
    'data': [{'labels': ['Amount donated in INR','Amount donated in USD'],
              'values': [total-total_donated_in_usd,total_donated_in_usd],
              'type': 'pie',
               'name': '1',
               'domain': {'x': [0, .49]}
             },
             {'labels': ['Number of donations in INR','Number of donations in USD'],
              'values': [total_donors-usd_donor_count,usd_donor_count],
              'type': 'pie',
              'name': '2',
              'domain': {'x': [.52, 1]}
             },
            ],
    'layout': {'title': 'Distribution of Donations by Currency'}
     }
py.iplot(fig)
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~thekindlyone/96.embed" height="525px" width="100%"></iframe>



## Religious Distribution of Donors


This thing was a religious issue. It might be interesting to find out birth-religion distribution of donors.
There is no very good way of finding religion from name, but I found a [website](http://indiachildnames.com/religionof.aspx) that does it. Using [requests](https://github.com/kennethreitz/requests) for http and [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/) for html parsing.


```python
import requests
from bs4 import BeautifulSoup as bs
```


```python
def get_religion(name):
    url='http://indiachildnames.com/religionof.aspx'    
    for i in range(10):
        try:
            r=requests.get(url,params={'name':name})
            break
        except:
            continue
    else:
        print 'error at' ,name
        return None
    soup=bs(r.content,'lxml')
    try:
        religion = '/'.join([a.text[:-2] for a in soup.select('table#castesummary > tr > td')[0].find_all('a')[:-1]])
        return religion
    except :
        return None


# insert religion info in data    
for d in data:
    if d['name']:
        religion = get_religion(d['name'])
        if religion:
            d['religion'] = religion
```


```python
from collections import Counter
c=Counter([d.get('religion','Unknown') for d in data])
labels,values=zip(*[(k,v) for k,v in c.iteritems()])
fig = {
    'data': [{'labels': labels,
              'values': values,
              'type': 'pie'}],
    'layout': {'title': 'Distribution of Donors by Religion'}
     }

py.iplot(fig)
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~thekindlyone/100.embed" height="525px" width="100%"></iframe>





-----------------------------------
{% include JB/setup %}
