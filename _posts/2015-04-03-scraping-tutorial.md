---
layout: post
title: "Scraping Tutorial"
description: "Hands on scraping tutorial using python, beautifulsoup and requests with some pillow magic thrown in"
category: lessons
tags: ["python","beautifulsoup","pillow","pil","requests","scraping"]
---
{% include JB/setup %}

##(or how I stopped waiting for each page to load and scraped a webcomic instead.)

So I wanted to read [this webcomic](http://www.paranatural.net/) ,but I have terrible internet, and the pages were taking too long to load. On top of that, having downloaded webcomics before, I did not want html output. I wanted CBRs(Comic Book Rar, RARs renamed) which my android devices can read. Also the images have title text(mouseovers) that need to be stored and displayed with each page. 

###Requirements

* For the web scraping stuff I used [requests](http://docs.python-requests.org/en/latest/) and 
* [BeautifulSoup](http://www.crummy.com/software/BeautifulSoup/) . 
* The image work, ie putting the mouseover text below the image is handled by [Pillow, a superior fork of the Python Imaging Library](http://pillow.readthedocs.org/en/latest/index.html)
* Finally, for the text stuff, I found [this recipe that worked wonders](https://gist.github.com/turicas/1455973). (This is a salient feature of python- just insane third party support) Props to the author. Saved me some work.

This was not a difficult job. Python has gotten me out of much more problematic situations. This article is written as a pitch for selling python to first/second language shoppers.

###Let's size up the Enemy

Before any scraping is done, we have to check out the website and look for patterns. Automation is all about recognizing patterns.   
Looking at the [first page](http://www.paranatural.net/index.php?id=1) and the 'last comic' navigation button link it is clear that the [final id is 289](http://www.paranatural.net/index.php?id=289).
Looking at a few random page sources, it is clear that   
```<div id="comicbody"><a href="/index.php?id=176"><img title="''Irritate your subordinates. It's really fun.'' -Sun Tzu" src="http://www.paranatural.net/comics/2013-06-28-chapter-4-page-18.png" id="comic" border="0" /><br /></a></div>```   
is the ```div``` with all the information we need: The title text(mouseovers) and the image source. 

### Time to write some code
Performing *GET* requests using the requests module is terribly easy. Using [the ```get()``` method of the requests module](http://docs.python-requests.org/en/latest/user/quickstart/#make-a-request), you obtain a response object that has the ```status_code``` and the ```content``` attributes that contain the response status and the html content respectively.   
{% highlight python %}
    In [8]: response=requests.get('http://www.paranatural.net/index.php?id=1')
    In [9]: response.status_code
    Out[9]: 200
    In [10]: response.content
    Out[10]: '<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">\n\n<html xmlns=.....
{% endhighlight %}
Given that network operations can be prone to timeouts and other connectivity issues( more so if you have bad internet), we ought to wrap this getting of pages stuff in a function which retries getting pages multiple times before giving up.
{% highlight python %}
    def get_page(url,max_attempts):
        for i in xrange(max_attempts):
            r = requests.get(url)
            if r.status_code == 200:
                return r.content
        print r.status_code
        return False
{% endhighlight %}
This function takes a ```url``` and an integer ```max_attempts``` and attempts to get the page ```max_attempts``` times returning immediately with the content in case of success(code 200) 

Okay, cool. But now we need to dig out the ```<div id="comicbody">``` from the content.

### Enter BeautifulSoup
[HTML parsing can be tricky if you are not using the right tools.](http://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags/1732454#1732454)   
But BeautifulSoup simplifies the process vastly, making simple web scraping almost trivial. To work with BeautifulSoup, you first make a soup object using the method ```BeautifulSoup(<html string>)``` . To look for a specific div with known id in the entire document you can use the ```find()``` method. the objects returned by this can access child elements using the ```.``` syntax. To get an attribute of the element, you use the ```get()``` attribute.
{% highlight python %}
    In [4]: html=get_page('http://www.paranatural.net/index.php?id=1',5)
    In [5]: soup=BeautifulSoup(html)
    In [6]: comicdiv=soup.find(id='comicbody')
    In [7]: comicdiv
    Out[7]: <div id="comicbody"><a href="/index.php?id=2"><img border="0" id="comic" src="http://www.paranatural.net/comics/2011-04-30-chapter 1.png" title="Twelve-year-old Max moves into the picturesque town of Mayview and is plagued by strange visions and stranger people on his first day of school."/><br/></a></div>
    In [8]: comicdiv.img
    Out[8]: <img border="0" id="comic" src="http://www.paranatural.net/comics/2011-04-30-chapter 1.png" title="Twelve-year-old Max moves into the picturesque town of Mayview and is plagued by strange visions and stranger people on his first day of school."/>
    In [9]: comicdiv.img.get('src')
    Out[9]: u'http://www.paranatural.net/comics/2011-04-30-chapter 1.png'   
    In [10]: comicdiv.img.get('title')
    Out[10]: u'Twelve-year-old Max moves into the picturesque town of Mayview and is plagued by strange visions and stranger people on his first day of school.'

{% endhighlight %}
Lets wrap this in a function so that we don't have to deal with the soupy details while setting up the whole contraption.
{% highlight python %}
    def extract(link): # takes url and returns a tuple of image source and mouseover text strings
        html=get_page(link,5)
        if html:
            soup=BeautifulSoup(html)
            comic=soup.find(id='comicbody').img
            src=comic.get('src')
            mouseover=comic.get('title')
            return(src,mouseover)
        else:
            return False
{% endhighlight %}

But wait! We need to download the images as well. That is the point isn't it?   
No problem. We will just use the ```get_page``` function in another download function which will download the images for us. Once we have the contents of a successful request response, all we need to do is to write the contents to a local file. This is done with the built in [open()](https://docs.python.org/2/library/functions.html#open) function. Content can be written to it using the ```write()``` method on the file descriptor object generated by ```open()```.
{% highlight python %}
    def download(src,localname): #takes in image source and local filename and downloads image to out\localfile
        image=get_page(src,5) # get page, try maximum 5 times
        if not image:
            return False
        f=open('out/'+localname,'wb') # open a file
        f.write(image) # write content to file
        f.close() # close the file
        return True
{% endhighlight %}
The scraping part is officially over. We have all the required functions, and the rest is just calling them in a loop passing to them new urls each time.

But before we put everything together, let us get the mouseover texts into the images so that the comic book reading software can display them.


### A picture is worth 1K words..    
#####(except when it is not)

####The Job:
1. Get the original image.
2. Extend it at the bottom just the right amount to accommodate the mouseover text. 
3. And finally put the text at the bottom.

Parts 1 and 3 can be easily done with pillow, but part 2 can be a little tricky(like 20 mins of experimentation). Fortunately, this is Python and someone on the internet has already spent those 20 minutes. We use [this image_utils module that does exactly what we need](https://gist.github.com/turicas/1455973).
To make it work for pillow(it is written for PIL), take a look at [porting pil code to pillow.](http://pillow.readthedocs.org/en/latest/porting-pil-to-pillow.html) It is just a matter of changing the imports. The example code in the link is quite self explanatory.   
First you get a special ```ImageText``` object initializing it with size and background color tuples(2-tuple for size and 4-tuple for RGBA background color). Then calling the ```write_text_box()``` method on the ```ImageText``` object we can write text to this image.    
Based on the parameters you provide to ```write_text_box()``` you can get it to align, size, and split into lines according to max width any text and put it on the image. The ```image``` attribute of the ```ImageText``` object is the PIL/pillow compatible image object.   
The ```write_text_box()``` method also returns a size tuple of the text box which will be very useful in determining how much to expand the original image.

Reading through [the pillow docs](http://pillow.readthedocs.org/en/latest/index.html) we can see that we can do standard operations like open image, save it, crop it, copy it, paste it etc using the various methods exposed by it.    

####The Plan:
* Get original image size.
* Create temporary ```ImageText``` object of sufficient height and width.
* Write text to temp object using ```write_text_box()``` with box_width = original image's width. (If width is provided, the module automatically manages the number of lines the text has to be split into.)
* Get the height of the text box from the call to ```write_text_box()```
* Make a new blank pillow image with width = original width and height = original height + height of text box + 20pixels(this is the standard offset determined by trial and error by me)
* Crop the temporary image to extract just the text box portion.
* Paste the original image on the new blank image.
* Paste the cropped text box at the bottom of the new image.
* Profit.

![Imgur](http://i.imgur.com/HitfXXO.png)

Code:
{% highlight python %}
    def make_page(infile,outfile,text): # takes in input file name , output file name and text to put in at the bottom
        font = 'Aller_Rg.ttf' #some random font, just find one legible enough
        color = (50, 50, 50) #color of text box
        page=Image.open(infile) # opening original image
        width,original_height= page.size    # size of original image
        temp=ImageText((1000, 500), background=(255, 255, 255, 200)) # making a large temp Image text object to put the text in.
        # get the height of the text box, the second element of the returned tuple+20(offset) 
        height=temp.write_text_box((0,0),text,box_width=width,font_filename=font,font_size=16,color=color)[1]+20 
        textbox=temp.image.crop( (0,0,width,height) ) # crop textbox
        output=Image.new("RGBA", (width,original_height+height),(120,20,20)) # make new blank image with computed dimensions
        output.paste(page,(0,0)) # paste original
        output.paste(textbox,(0,original_height)) # paste textbox
        output.save(outfile) # save file

{% endhighlight %}

### Some loose ends
As you might have noticed we have used an 'out' directory to keep the downloaded files. And we will be using a 'final' directory to store the processed images. It will be awkward if those folders are not present. So let us make them. The [os module](https://docs.python.org/2/library/os.path.html) in the standard library has a lot of convenience functions that help with this path stuff.    

We will be also be using ```os.path.join()``` and ```os.path.basename()``` methods. They are fairly straightforward. join combines path components and basename gigvs you the main file name from a large path. These functions are very very useful if you use windows and are trapped in the eternal tragedy of the slashes. Another helpful function ```os.getcwd()``` gets the current working directory.
{% highlight python %}
    for directory in ['out','final']: # loop through the the directory names you want created
        if not os.path.exists(directory) : # check if they do not exist
            os.makedirs(directory) # if they don't exist, create them
{% endhighlight %}
We also store the output path in a string using ```join``` and ```getcwd```   
```outpath=os.path.join(os.getcwd(),'final')```

### The Mainloop
All the prerequisites have been taken care of and only putting it all together remains.   

Here is the code:
{% highlight python %}
    url='http://paranatural.net/index.php?id='
    for index,link in enumerate( ( url+str(i) for i in range(1,290) ) ): # loop through the urls
        src,mouseover=extract(link) # src gets image source, mouseover gets the mouseover text
        filename=string.zfill(str(index+1),3)+'.'+src[-3:].lower() #generate the filename like 001.png
        download(src,filename) # download to the filename
        print 'downloaded #',index+1,'  ',filename
        make_page(os.path.join('out',filename),os.path.join(outpath,os.path.basename(filename)),mouseover) # process the page
        print 'processed #',index+1,'  ',filename
        sys.stdout.flush() # printing is bufferred by default and this makes the message display as soon as it is encountered
    print '100% complete, all tasks done!' 
{% endhighlight %}
4 peculiar things have been used here. [enumerate](https://docs.python.org/2/library/functions.html#enumerate), [a generator expression](https://docs.python.org/2/howto/functional.html#generator-expressions-and-list-comprehensions), [zfill](https://docs.python.org/2/library/string.html#string.zfill) and [slicing syntax](https://developers.google.com/edu/python/strings) (scroll down to the slicing part)

*Enumerate* is a way to deal with index in ```for..in``` style loops. This is how it works.
{% highlight python %}
    In [4]: l=['a string','another one',5,2.5]
    In [5]: for item in l:
       ...:     print item
       ...:
    a string
    another one
    5
    2.5
    In [6]: for index,item in enumerate(l):
       ...:     print 'index= ',index,' item= ',item
       ...:
    index=  0  item=  a string
    index=  1  item=  another one
    index=  2  item=  5
    index=  3  item=  2.5
{% endhighlight %}
*Zfill* just takes a string and adds 0s in front of it till it reaches supplied width. It is useful for converting a '1' to a '001' which makes it easier for sorting.
{% highlight python %}
    In [8]: for i in range(5):
       ...:     print string.zfill(str(i),4)
       ...:
    0000
    0001
    0002
    0003
    0004

{% endhighlight %}
*A generator expression* is a shorthand way of writing a loop. say you want an iterator(something we can loop over with ```for..in```) that contains the squares of the first n natural numbers. you can do this 
{% highlight python %}    
    In [10]: gen=(number**2 for number in range(1,6) )
    In [11]: for square in gen:
       ....:     print square
       ....:
    1
    4
    9
    16
    25
{% endhighlight %}

*Slicing syntax* is a super convenient way of referring to sections in a list or a string. This is one of the reasons why string work is such a breeze in python.   
Say there is a string 'DeadParrot'   
<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;border:none;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:0px;overflow:hidden;word-break:normal;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:0px;overflow:hidden;word-break:normal;}
</style>
<table class="tg">
  <tr>
    <th class="tg-031e">D</th>
    <th class="tg-031e">e</th>
    <th class="tg-031e">a</th>
    <th class="tg-031e">d</th>
    <th class="tg-031e">P</th>
    <th class="tg-031e">a</th>
    <th class="tg-031e">r</th>
    <th class="tg-031e">r</th>
    <th class="tg-031e">o</th>
    <th class="tg-031e">t</th>
  </tr>
  <tr>
    <td class="tg-031e">0</td>
    <td class="tg-031e">1</td>
    <td class="tg-031e">2</td>
    <td class="tg-031e">3</td>
    <td class="tg-031e">4</td>
    <td class="tg-031e">5</td>
    <td class="tg-031e">6</td>
    <td class="tg-031e">7</td>
    <td class="tg-031e">8</td>
    <td class="tg-031e">9</td>
  </tr>
  <tr>
    <td class="tg-031e">-10</td>
    <td class="tg-031e">-9</td>
    <td class="tg-031e">-8</td>
    <td class="tg-031e">-7</td>
    <td class="tg-031e">-6</td>
    <td class="tg-031e">-5</td>
    <td class="tg-031e">-4</td>
    <td class="tg-031e">-3</td>
    <td class="tg-031e">-2</td>
    <td class="tg-031e">-1</td>
  </tr>
</table>
{% highlight python %}
    In [1]: s='DeadParrot'
    In [2]: s[1:5] #starting index 1 extending till 5 excluding 5
    Out[2]: 'eadP'
    In [3]: s[1:] #omitting index defaults to start or end
    Out[3]: 'eadParrot'
    In [4]: s[5:]
    Out[4]: 'arrot'
    In [5]: s[5:100]  # out of range truncates
    Out[5]: 'arrot'
    In [6]: s[-1] # last element
    Out[6]: 't'
    In [7]: s[-4] # 4th from end
    Out[7]: 'r'
    In [8]: s[-3:] # last three
    Out[8]: 'rot'
    In [9]: 'filename.ext'[-3:] # trick used in mainloop
    Out[9]: 'ext'
{% endhighlight %}
### The full code

{% highlight python %}

    import requests
    from bs4 import BeautifulSoup
    import string #for zfill
    from PIL import ImageFont, Image, ImageDraw, ImageOps
    from image_utils import ImageText
    import os
    import sys
    def get_page(url,max_attempts):
        for i in xrange(max_attempts):
            r = requests.get(url)
            if r.status_code == 200:
                return r.content
        print r.status_code
        return False

    def extract(link):
        html=get_page(link,5)
        if html:
            soup=BeautifulSoup(html)
            comic=soup.find(id='comicbody').img
            src=comic.get('src')
            mouseover=comic.get('title')
            return(src,mouseover)
        else:
            return False

    def download(src,localname):
        image=get_page(src,5)
        if not image:
            return False
        f=open('out/'+localname,'wb')
        f.write(image)
        f.close()
        return True

    def make_page(infile,outfile,text):
        font = 'Aller_Rg.ttf' #some random font, just find one legible enough
        color = (50, 50, 50) #color of text box
        page=Image.open(infile)
        width,original_height= page.size    
        temp=ImageText((1000, 500), background=(255, 255, 255, 200))
        height=temp.write_text_box((0,0),text,box_width=width,font_filename=font,font_size=16,color=color)[1]+20 # +20 y offset. text writer leaves that much space at the top 
        textbox=temp.image.crop( (0,0,width,height) )
        output=Image.new("RGBA", (width,original_height+height),(120,20,20))
        output.paste(page,(0,0))
        output.paste(textbox,(0,original_height))
        output.save(outfile)

    #make required directories
    for directory in ['out','final']:
        if not os.path.exists(directory) :
            os.makedirs(directory) 

    outpath=os.path.join(os.getcwd(),'final')
    url='http://paranatural.net/index.php?id='
    for index,link in enumerate( ( url+str(i) for i in range(1,290) ) ):
        src,mouseover=extract(link)
        filename=string.zfill(str(index+1),3)+'.'+src[-3:].lower()
        download(src,filename)
        print 'downloaded #',index+1,'  ',filename
        make_page(os.path.join('out',filename),os.path.join(outpath,os.path.basename(filename)),mouseover)
        print 'processed #',index+1,'  ',filename
        sys.stdout.flush()
    print '100% complete, all tasks done!'
{% endhighlight %}
that was ~60 lines of python btw ;) if you hadn't noticed.

<div class="center-block">
<img src="http://i.imgur.com/RtgK5B0.png" title="Original Page #1" alt="imgur" width="400" height="475" style="border:2px solid black">

<h4>+ "Twelve-year-old Max moves into the picturesque town of Mayview and is plagued by strange visions and stranger people on his first day of school."(mouseover text) = </h3>

<img src="http://i.imgur.com/YudT4hb.png" title="Processed Image" alt="imgur" width="400" height="475" style="border:2px solid black">
</div>


That is all.

