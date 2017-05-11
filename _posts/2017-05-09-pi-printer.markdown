---
layout: post
title:  "Powered 3D-Printed XKCD Printer"
date:   2017-05-08 21:20:27 -0700
categories: pi python hardware 3dprint
---

{% include image.html url="http://192.168.1.108:4000/assets/images/pi/1.jpg" description="I did it!" %}

# Pi Zero W
![Pi Zero W](http://192.168.1.108:4000/assets/images/pi/pi.jpg)
Sometime last month, I discovered that [Raspberry Pi Zero W](https://www.raspberrypi.org/products/pi-zero-w/) was a thing. Holy crap! It's a Pi Zero, but it has built in Wi-Fi and Bluetooth so you can avoid the [dongle-hub demonspawn](http://duinorasp.hansotten.com/wp-content/uploads/2015/12/IMG_4632.jpg) that defeats the whole purpose of having a tiny device in the first place

# Inventing an excuse to buy a cool thing
I realized I had a magnificent hammer before me, yet no nail. There were loads of projects online, and it didn't take long to get inspired by this [instant camera](https://learn.adafruit.com/instant-camera-using-raspberry-pi-and-thermal-printer/overview) that uses a thermal printer. The idea of a tiny printer intrigued me (although I cannot imagine why--after all, normal sized printers are mundane garbage you find at garage sales but also in the literal garbage), and so I had yet another hammer, but still no nail. 

Pi Zero, built in Wi-Fi, thermal printer... I guess I could have it... print something... from... the Internet? But it only prints in black and white, so ideally I should print something that has no color to begin with. [XKCD](https://xkcd.com/) is black and white, yeah? Perfect. Done.

# Parts List
* [Raspberry Pi Zero W](http://www.microcenter.com/product/475267/zero_wireless_development_board) - $10
* [Adafruit Tiny Thermal Receipt Printer - TTL Serial / USB](https://www.adafruit.com/product/2751) - $50 - Supports USB, and they provide a Python library.
* [Barrel jack](https://www.adafruit.com/product/368) - $2
* [5V Power supply](https://www.adafruit.com/product/276) - $8 - Apparently the printer can use over 1.5A of power, but the Pi Zero W only uses ~120mA at idle. The most CPU-intensive part of the project is resizing the image, which won't even happen while the printer is running.
* [USB OTG Adapter](https://www.amazon.com/gp/product/B015GZOHKW) - $4 for 5 - USB A to Micro USB, for connecting the thermal printer to the Pi
* Micro USB cable to cannibalize
* Soldering iron, wire strippers, hot glue
* [Monoprice Maker Select 3D Printer](https://www.monoprice.com/product?p_id=13860) with Hatchbox PLA filament. I can't recommend this printer enough--I've had zero issues with it. Out of the box, I just had to level the bed and calibrate my e-steps and it's been smooth sailing the past year and a half.

It seemed like every website was sold out of the Pi Zero W, but luckily, my local Micro Center had assloads. They kept them in a locked drawer, and had a pretty clever pricing scheme: $10 for one, $15 each if you buy more than one. Death to scalpers! Normal humans unite! 

"I'm only buying one. I'm sure not paying $5 extra; I can always come back later for a second," I stupidly said to myself after having already spent well over $5 on gas and car depreciation getting to and from Micro Center.


# Python is OP
My thermal printer was still in the mail, because apparently in this post-Amazon-Prime era, every other website still charges you for standard shipping that takes a week, so I started coding.

I had heard that Python makes shit easy, but I was unaware of the reality of the situation. I had to download an XKCD image, resize it, change it to a binary bitmap, and print it. Surprise! There are libraries for all three of those things!
1. For doing XKCD crap, I used the aptly-named [XKCD](https://pypi.python.org/pypi/xkcd/) library
2. For doing image crap, I used the aptly-named Image package of the awkwardly-named [PIL](http://www.pythonware.com/products/pil/) library
3. For thermal printing crap, I used the aptly-named [Python-Thermal-Printer](https://github.com/adafruit/Python-Thermal-Printer) library from Adafruit

## Getting a comic
{% highlight python %}
import xkcd as xkcd
comic = xkcd.getLatestComic()
comic.download(output="./Images", outputFile='{}.png'.format(comic.number))
{% endhighlight %}

Huh, that was a little too easy. 

## Rotating and scaling and black and white
This ended up being the only remotely complicated code in the entire project.

### Rotating
The printer has a max width of 384 pixels, and obviously I wanted to print the comics in portrait mode every time.

I only want to rotate when it's wider than it is tall, obviously:
{% highlight python %}
from PIL import Image
with Image.open('./Images/{}.png'.format(comic.number)) as image:
    width, height = image.size
    if width > height:
        image = image.rotate(-90)
        width, height = image.size
{% endhighlight %}
            
### Scaling
The printer's horizontal resolution is 384 pixels, so scale everything up or down to that maximum width:
{% highlight python %}
scale_factor = float(max_width) / float(width)
new_height = int(scale_factor * height)
image = image.resize((max_width, new_height), Image.ANTIALIAS)
{% endhighlight %}

### Black & white
Then convert it to black and white. Not greyscale--it has to be a binary bitmap. Therefore, I had to find a set threshold that worked well for most every comic. I found that a threshold of 60 works well. If the brightness is less than 60, it maps to 0. More than 60, maps to 255.
{% highlight python %}
image = image.convert('1')
image = image.point(lambda x: 0 if x < 60 else 255, '1')
{% endhighlight %}        

{% include image.html url="http://192.168.1.108:4000/assets/images/pi/xkcd.bmp" description="The resulting image is actually pretty good (comic #438 for comparison)" %}

### Printing
Alright, at this point, my thermal printer came. I used a tiny (OTG USB adapter)[https://www.amazon.com/gp/product/B015GZOHKW] to save space plugging it in.
Surely, I'll have to do some file conversions, or converting the image to a byte array or compact the 0-255 bytes into 0/1 bits in a single array. I mean, surely, Adafruit's library wouldn't be able to print the image directly from the PIL library I used...

{% highlight python %}
from Adafruit_Thermal import *
printer = Adafruit_Thermal("/dev/ttyUSB0", 19200, timeout=5)
printer.printImage(image)
{% endhighlight %}

Oh. Python is going to take my job some day, isnt it?

# Case & assembly
I'd like to think of myself as a lazy person. Continuing the Python trend of letting everyone else do the work for me, I found an enclosure someone had made in OpenSCAD on Thingiverse made by the beautiful LBussy. He calls it the [Customizable Parametric Enclosure with Lid](http://www.thingiverse.com/thing:1918169) presumably because it is all of those things.

Side bar--do you 3D print stuff? If so, you need calipers. You'll thank me later. [Buy this one](https://www.amazon.com/gp/product/B001AQEZ2W) because it's great.

After measuring with the aforementioned calipers, I had to modify the OpenSCAD file to subtract holes for the printer and the barrel jack.

{% include image.html url="http://192.168.1.108:4000/assets/images/pi/4.jpg" description="Final design--I made it a little large so I wouldn't have to print it twice." %}

## Printing the case
Pretty straight forward. Click and go. Printed with 0.3mm layer height because I ain't got time for anything less.

<iframe src='https://gfycat.com/ifr/DenseCluelessJay' frameborder='0' scrolling='no' allowfullscreen width='640' height='480' style='margin: auto;'></iframe>

## Soldering crap
I cut the Micro USB tip with a couple inches of wire and stripped the end. I was greeted with definitely-not-standard coloring, and had no clue which ones carried power. 

{% include image.html url="http://192.168.1.108:4000/assets/images/pi/5.jpg" description="Where is your God now?" %}

The solution was to plug the other half into a USB charger and poke at the exposed wires with my multimeter. Very scientific. Turns out, for this Anker USB cord, the positive wire is the pink one and the grey one is the negative or ground or whatever. For the USB printer, it was your standard red/positive and black/negative. Solder negative to negative, positive to positive, jam it in the barrel jack. 

## Put crap into the box
Hot glue it into the opening. Shove everything into the case. Not too complicated. My printer is calibrated pretty well, so I didn't have to re-print or re-measure anything.

{% include image.html url="http://192.168.1.108:4000/assets/images/pi/2.jpg" description="Unceremoniously cram it in--nice" %}

# Finishing touches
{% include image.html url="http://192.168.1.108:4000/assets/images/pi/3.jpg" description="All finished in a neat little package." %}

In the end, I spruced it up a bit more. I also print the title and the alt text, along with some text wrapping since the printer library didn't natively support that. I updated the script so that it keeps track of what it has already printed (just using the file system), so that it can fetch a random comic if it has already printed the latest one. Lastly, I made a cron job so I get a new XKCD comic every morning.

# Send help
{% include image.html url="http://192.168.1.108:4000/assets/images/pi/6.jpg" description="What am I going to do with all of these???" %}


# Appendix
* [Final Python code (Github)](https://github.com/Jamish/xkcd-pi-printer)
* Case Files
  * [pi-printer-box-and-lid.scad](http://192.168.1.108:4000/assets/files/pi/pi-printer-box-and-lid.scad)
  * [pi-printer-lid.stl](http://192.168.1.108:4000/assets/files/pi/pi-printer-lid.stl)
  * [pi-printer-box.stl](http://192.168.1.108:4000/assets/files/pi/pi-printer-box.stl)

I don't own XKCD or anything like that, in case it wasn't obvious. 

